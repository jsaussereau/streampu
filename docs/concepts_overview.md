# Concepts Overview

This page introduces the vocabulary of `StreamPU` and the main rules of the
DSEL. The following pages ([Getting Started](getting_started.md),
[Writing Modules](custom_modules.md), [Sequences](sequence_usage.md),
[Pipelines](pipeline_usage.md), ...) detail each concept with code.

<figure markdown>
  ![Sequence and pipeline](./assets/sequence_to_pipeline.svg){ width=1200 }
  <figcaption>A sequence of tasks (left) and the same tasks executed in a 3-stage pipeline (right).</figcaption>
</figure>

## Tasks, Modules & Sockets

A `StreamPU` application is a **dataflow graph**. Its nodes are **tasks** and
its edges connect the **sockets** of these tasks.

- A **task** (`spu::runtime::Task`) is the elementary piece of code. It owns a
  *codelet* (a C++ function, usually a lambda) and a list of sockets. In other
  languages, a task may be called a *job*, a *filter* or an *actor*.
- A **socket** (`spu::runtime::Socket`) is a typed and sized piece of data
  consumed or produced by a task. There are three kinds of sockets:
    * **input** sockets (`socket_t::SIN`): the task reads the data;
    * **output** sockets (`socket_t::SOUT`): the task writes the data, the
      memory belongs to the output socket;
    * **forward** sockets (`socket_t::SFWD`): the task reads **and** writes the
      data in place, the memory comes from the socket it is bound to.
- A **module** (`spu::module::Module`) is a container of tasks. The tasks of a
  module can share data through the module (its attributes). Modules are the
  objects created by the user: the tasks and their sockets are created by the
  module constructor.

<figure markdown>
  ![Task with sockets!](./assets/task_sockets.svg){ width="600" }
  <figcaption>Tasks with different socket types.</figcaption>
</figure>

Sockets are named and can be accessed from the module with the
`"task_name::socket_name"` syntax. Tasks are accessed with the
`module("task_name")` syntax:

```cpp
module::Incrementer<int> inc(16);    // module with 2 tasks: 'increment' and 'incrementf'
runtime::Task&   t = inc("increment");       // a task
runtime::Socket& s = inc["increment::in"];   // a socket of the 'increment' task
runtime::Socket& s2 = inc("increment")["in"]; // the same socket, accessed from the task
```

### Supported Data Types

The data type of a socket is one of the following C++ types: `int8_t`,
`uint8_t`, `int16_t`, `uint16_t`, `int32_t`, `uint32_t`, `int64_t`, `uint64_t`,
`float` and `double`. The enumeration `runtime::datatype_t` (`F64`, `F32`,
`S64`, `S32`, `S16`, `S8`, `U64`, `U32`, `U16`, `U8`) can also be used to
specify a type at runtime.

### The `status` Socket

When the codelet of a task is created, `StreamPU` automatically appends a last
output socket named `status` to the task. This socket contains the `int` value
returned by the codelet (one value per *wave*, see
[Frames and Waves](#frames-and-waves)). The predefined values are listed in the
`runtime::status_t` enumeration:

| Value                     | Meaning                                                                                       |
| ------------------------- | --------------------------------------------------------------------------------------------- |
| `SUCCESS` (`0`)           | The codelet succeeded.                                                                        |
| `FAILURE` (`1`)           | The codelet failed (informative only, the runtime does not stop).                             |
| `FAILURE_STOP` (`-1`)     | The codelet failed: when a task processes several waves, the remaining waves are not executed. |
| `UNKNOWN` (`-2`)          | Value written in the `status` socket before the codelet is called.                            |

Any other integer can be returned. For instance, the `commute` task of the
[`Switcher`](control_flow.md) returns the index of the selected path.

!!! note
    A user socket cannot be named `status` and no socket can be added to a
    task after its codelet has been created.

## Binding

Binding two sockets means connecting an output (or forward) socket to an input
(or forward) socket: after the binding, the input socket points to the memory
of the output socket. **There is no copy**: the data produced by a task is
directly read by the next one.

The binding is written with the `=` operator, usually with the consumer on the
left side (the direction is deduced from the socket types, so
`producer["task::out"] = consumer["task::in"];` is also accepted):

```cpp
consumer["task::in"] = producer["task::out"];
```

The rules are the following:

- `SIN = SOUT`, `SIN = SFWD`, `SFWD = SOUT` and `SFWD = SFWD` are valid bindings;
- an input socket can only be bound to **one** socket, while an output socket can
  be bound to **many** input/forward sockets (fan-out);
- the two sockets must have the same data type and the same size, otherwise an
  exception is raised.

It is also possible to bind a socket to user data (a `std::vector` or a
pointer): `inc["increment::in"] = my_vector;`. The vector must contain exactly
`n_frames × n_elmts` elements and the element type must match the socket data
type. Such an input socket is called a *static* input socket.

Finally, a **task can be bound to another task** to express a dependency
without data exchange: `t2 = t1;` means that `t2` has to be executed after `t1`
(see [Task-to-task binding](getting_started.md#task-to-task-binding)).

## Frames and Waves

Each module processes `n_frames` frames each time one of its tasks is executed
(`n_frames = 1` by default). The size of a socket given at creation time is the
size **of one frame**: the real memory space of a socket is
`n_frames × n_elmts` elements, the frames being contiguous in memory.

Processing several frames per execution (also called *inter-frame level*)
amortizes the overhead of the runtime and can enable SIMD inter-frame
processing. It is set with `set_n_frames()` on a module, a sequence or a
pipeline.

By default, the codelet of a task is called **once per frame**: during each
call, the socket data pointers point to the current frame, and the `frame_id`
parameter of the codelet gives the index of this frame. A module can change
this behavior:

- `set_n_frames_per_wave(n)`: the codelet is called once per group of `n`
  frames (a *wave*), the data pointers point to the first frame of the wave;
- `set_single_wave(true)`: the codelet is called only once for all the frames
  (`n_frames_per_wave == n_frames`).

The `status` socket contains one value per wave. The [Writing
Modules](custom_modules.md#frames-and-waves) page details how to handle frames
in a codelet.

## Sequences and Pipelines

Once the tasks are bound, the graph is executed by a **runtime** object:

- A **sequence** (`spu::runtime::Sequence`) analyzes the graph from one or
  several *first* tasks and computes a **fixed execution order**. Each call to
  `exec()` executes the whole graph again and again (one execution = one
  stream of `n_frames` frames) until a stop condition is met. A sequence can be
  **replicated** over `n` threads: each thread executes its own copy of the
  graph, made of clones of the modules (**replication parallelism**, also
  called *data parallelism over streams*).
- A **pipeline** (`spu::runtime::Pipeline`) splits a sequence into **stages**.
  Each stage is a sequence executed by its own thread(s), and the stages are
  connected by synchronization buffers (**pipeline parallelism**). Each stage
  can be replicated, which allows to combine both pipeline and replication
  parallelisms.

The tasks are ordered as follows: starting from the first task(s), a task is
added to the execution order once **all its input and forward sockets** (and
its task-to-task dependencies) have been fed by tasks already added to the
order. Static input sockets (bound to user data) are not taken into account.

`StreamPU` also offers **control flow** with the
[`Switcher`](control_flow.md) module (loops, switch-case), which makes the
language Turing-complete.

!!! info
    `StreamPU` targets single-rate *Synchronous DataFlow* (SDF) applications:
    each task consumes and produces exactly one frame per execution.

## Module States & Tasks Replicability

In `StreamPU`, modules are either *stateful* or *stateless*. A *stateful*
module is an instance of a class that inherits from `module::Stateful`. A
*stateless* module is an instance of the `module::Stateless` class (or of a
class that inherits from it, like `module::Stateless_Julia`). Both
`module::Stateful` and `module::Stateless` inherit from `module::Module`.

A module is *clonable* if it can be copied: this is required to replicate its
tasks over several threads. A task is *replicable* if it can be replicated in a
sequence or in a pipeline stage.

- **Stateless modules are always clonable**. As a consequence, their tasks are
  replicable by default.
- **Stateful modules are not clonable by default** (`Module::clone()` throws a
  `tools::unimplemented_error`). In this case, their tasks are
  *non-replicable*. The designer of a stateful module can override the
  `clone()` method (that is, express how to copy the module). When it is done,
  the module becomes clonable and its tasks become replicable.

Sometimes one may want to prevent a replicable task from being replicated, for
instance for testing purposes, or because replicating it would make no sense
in the application. The public `runtime::Task::set_replicability(bool)` method
gives this control. If the boolean is set to `false`, the task is forced to be
non-replicable.

!!! warning
    If `set_replicability(true)` is called on a task whose module is not
    clonable, the method throws an exception. Setting the replicability to
    `false` is always possible.

!!! warning
    The replicability of a task is computed **when the task is created**, by
    calling `clone()` on the module being constructed. Consequently, the
    `clone()` method of a module must be callable from its constructor, right
    after `create_task()`.

To summarize:

- stateless module:
    * always clonable;
    * tasks are replicable by default;
    * tasks can be forced to be non-replicable.
- stateful module:
    * non-clonable by default;
    * the designer can implement the `clone()` method to make the module
      clonable;
    * if `clone()` is not implemented, the tasks are non-replicable;
    * if `clone()` is implemented, the tasks are replicable by default and can
      be forced to be non-replicable.

A sequence that contains at least one non-replicable task can only be executed
by one thread, and must be built from **non-`const`** tasks (see
[Sequences](sequence_usage.md#creating-a-sequence)).

## Memory Management

The output sockets own the memory of the graph. By default, this memory is
allocated **when a sequence (or a pipeline) is built**, for all the tasks it
contains. As a consequence, a task that is not part of a sequence cannot be
executed directly, unless the *task autoalloc* mode is enabled **before** the
creation of the modules:

```cpp
tools::Buffer_allocator::set_task_autoalloc(true); // output sockets are allocated at task creation
```

The input and forward sockets never own memory: they point to the memory of
the socket they are bound to.

In a sequence, some tasks exchange buffers instead of copying data (the
*no-copy mode*, enabled by default): this is the case of the
[`Switcher`](control_flow.md) tasks and of the pipeline synchronization tasks.
See [Sequences](sequence_usage.md#no-copy-mode) for more details.
