# Pipeline

A **pipeline** is a feature offered by `StreamPU` allowing to split a
[sequence](sequence.md) into multiple [stages](#Stage). Each stage is executed 
on one or more threads in parallel. The pipeline takes care of the 
**synchronizations between stages**. This is achieved through an implementation 
of **a producer/consumer algorithm**.

<figure markdown>
  ![Sequence and pipeline](./assets/sequence_to_pipeline.svg){ width=1200 }
  <figcaption>Example of a sequence on the left and a pipeline on the right.</figcaption>
</figure>

A pipeline is a C++ object of the `spu::runtime::Pipeline` class. The 
following sections give an overview of the most important attributes and 
methods to facilitate the code understanding. The user point of view is
described in [Pipelines](pipeline_usage.md).

## Main Attributes

```cpp 
runtime::Sequence original_sequence;
``` 
The original sequence (one thread) built from the first and last tasks given to
the constructor. It is used to check that the stages contain exactly the same
tasks.

<a name="Stage"></a>
```cpp 
std::vector<std::shared_ptr<runtime::Sequence>> stages;
``` 
Vector of the different stages in the pipeline. Each stage is a
[sequence](sequence.md).

```cpp
std::vector<std::pair<std::vector<std::shared_ptr<module::Adaptor_m_to_n>>,
                      std::vector<std::shared_ptr<module::Adaptor_m_to_n>>>> adaptors;
```
The [adaptors](#Adaptor) between each pair of consecutive stages: the first
vector contains the *pushers* (one per thread of the producer stage), the
second one the *pullers* (one per thread of the consumer stage).

```cpp
//                               sck out addr      stage   tsk id  sck id  unbind_pos
std::vector<std::pair<std::tuple<runtime::Socket*, size_t, size_t, size_t, size_t>,
//                               sck in addr       stage   tsk id  sck id  tsk in addr
                      std::tuple<runtime::Socket*, size_t, size_t, size_t, runtime::Task*>>> sck_orphan_binds;
```
Vector of [sockets](socket.md) with broken connections due to the pipeline
stages creation (hence "orphan"). These bindings are replaced by bindings to
special modules called [adaptors](#Adaptor) that make "bridges" between the
stages, and restored when the adaptors are unbound. The `tsk in addr` field is
used for the task-to-task bindings (fake input sockets).

```cpp
//                     sck out addr      sck in addr       priority  tsk in addr
std::vector<std::tuple<runtime::Socket*, runtime::Socket*, size_t,   runtime::Task*>> adaptors_binds;
```
The bindings between the adaptors and the tasks of the stages, `priority` is
used to order the tuples.

## Main Methods

```cpp
void exec();
```
This is the public method that runs the pipeline in loop. Other variants exist
where it is possible to give stop conditions (see
[Executing a Pipeline](pipeline_usage.md#executing-a-pipeline)). Each stage is
executed by a thread of the pipeline thread pool (the last stage by the
calling thread), which calls `Sequence::exec()` on the stage.

```cpp
void init(const std::vector<runtime::Task*> &firsts,
          const std::vector<runtime::Task*> &lasts,
          const std::vector<std::tuple<std::vector<runtime::Task*>, std::vector<runtime::Task*>, std::vector<runtime::Task*>>> &sep_stages = {},
          const std::vector<size_t> &n_threads = {},
          const std::vector<size_t> &synchro_buffer_sizes = {},
          const std::vector<bool> &synchro_active_waiting = {},
          const std::vector<bool> &thread_pinning = {},
          const std::vector<std::vector<size_t>> &puids = {},
          const std::string &pipeline_policy_pinning = "");
```

This method creates the pipeline given:

- The first and last tasks of the original sequence (`firsts` and `lasts`).
- The first and last tasks of each stage (`sep_stages`).
- The number of threads to allocate to each stage (`n_threads`).
- The number of buffers between stages (`synchro_buffer_sizes`).
- The type of waiting for the adaptor tasks (`synchro_active_waiting`).
- The thread pinning of each stage (`thread_pinning` and `puids`, or
  `pipeline_policy_pinning`).

Each stage is created as a `runtime::Sequence` (with `tasks_inplace = true` and
without memory allocation), then the adaptors are created and bound, and
finally the memory of the stages is allocated.

```cpp
void create_adaptors(const std::vector<size_t> &synchro_buffer_sizes = {},
                     const std::vector<bool> &synchro_active_waiting = {});
```
This function creates the Adaptor modules (and so the `pull` & `push` tasks) 
that are added between each stage to transmit data from the stage $S$ to the 
stage $S+1$. 

```cpp
void _bind_adaptors(const bool bind_adaptors = true);
```
Adaptor module tasks `pull` & `push` need to be bound to each task in the two
consecutive stages, the target sockets to bind are stored in the vector
`sck_orphan_binds.`

<a name="Adaptor"></a>
## Adaptor

`spu::module::Adaptor_m_to_n` is a special module automatically inserted
between stages when creating a pipeline. Adaptors serve as "bridges" between
the stages: they are bound to the tasks of the consecutive stages that were
connected in the original sequence. The purpose of adaptors is to synchronize
data exchange between stages using pre-allocated buffer pools. In other words,
this is **an implementation of the producer-consumer algorithm**, for `m`
producer threads and `n` consumer threads.

An adaptor has two tasks:

- `push`: executed at the end of the producer stage $S$ (one clone of the
  adaptor per thread of $S$, called a *pusher*). Its input sockets `in0`,
  `in1`, ... receive the data produced by the stage; the task waits for an
  empty buffer, fills it, and marks it as full.
- `pull`: executed at the beginning of the consumer stage $S+1$ (one clone per
  thread of $S+1$, called a *puller*). The task waits for a full buffer and
  provides its content on its output sockets `out0`, `out1`, ..., then marks the
  buffer as empty.

There is one socket per piece of data crossing the boundary between the two
stages. In the [no-copy mode](sequence_usage.md#no-copy-mode), the buffers
are not copied: the data pointers of the sockets bound to the `push` and `pull`
tasks are exchanged with the buffers of the pool.

With `m` pushers and `n` pullers, the adaptor allocates `lcm(m, n)` circular
queues (`lcm` = least common multiple, at most 1000) of `buffer_size` buffers
each. The pusher `i` writes successively in the queues `i`, `i + m`,
`i + 2m`, ... (modulo `lcm(m, n)`) and the puller `j` reads successively the
queues `j`, `j + n`, `j + 2n`, ... This round-robin distribution guarantees
that the frames are consumed in the order in which they have been produced.

### Main Attributes

```cpp
const size_t buffer_size;
```
The number of buffers of each queue (the `synchro_buffer_sizes` parameter of
the pipeline).

```cpp
std::shared_ptr<std::vector<std::vector<std::vector<int8_t*>>>> buffer;
```
Pointers to each buffer of the inter-stage pool (`[queue][socket][buffer]`),
shared by all the clones of the adaptor. Each buffer contains `n_frames` frames.

```cpp
std::shared_ptr<std::vector<uint32_t>> first;
std::shared_ptr<std::vector<uint32_t>> last;
std::shared_ptr<std::vector<std::atomic<uint32_t>>> counter;
```
Per-queue indexes: `first` is used to get the filled buffers, `last` for the
empty ones, and `counter` is the number of free slots.

```cpp
const bool active_waiting;
```
If `true`, the threads spin while the queue is full (pusher) or empty (puller);
otherwise they sleep on a condition variable and are woken up by the other
side.

```cpp
std::shared_ptr<std::atomic<bool>> waiting_canceled;
```
Set by `cancel_waiting()` to stop the waiting threads (a
`tools::waiting_canceled` exception is thrown by the `push`/`pull` tasks).

### Main Methods

These are the methods used to synchronize the buffer pool between the pipeline
stages. When getting a buffer, **the thread may wait** if there is no
empty (resp. filled) buffer available. When a buffer becomes available, the
waiting thread is **woken up** (passive waiting).

```cpp
void wait_push();
void wait_pull();
```
Wait until the current queue has a free slot (resp. a filled slot), or until the
waiting is canceled.

```cpp
void* get_empty_buffer(const size_t sid);
void* get_filled_buffer(const size_t sid);
```
Get a pointer to the first empty buffer (at index `last`) or to the first
filled buffer (at index `first`) of the current queue, for the socket `sid`.

```cpp
void* get_empty_buffer(const size_t sid, void* swap_buffer);
void* get_filled_buffer(const size_t sid, void* swap_buffer);
```
Same, but the buffer of the pool is replaced by `swap_buffer` (used in the
no-copy mode).

```cpp
void wake_up_puller();
void wake_up_pusher();
```
Mark the current buffer as filled (resp. empty), wake up the other side if
needed, and move to the next queue.

```cpp
void cancel_waiting();
void reset();
```
Stop all the waiting threads; reset the queues (called at the end of
`Pipeline::exec()`).
