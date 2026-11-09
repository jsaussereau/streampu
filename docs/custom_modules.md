# Writing Modules

There are two ways to create new tasks in `StreamPU`:

- create a `module::Stateless` object and add tasks to it (no new class
  needed), when the tasks do not need any inner data;
- write a new class that inherits from `module::Stateful`, when the tasks
  need inner data (a state, pre-allocated buffers, parameters, ...).

The same protected methods are used in both cases to create the tasks, the
sockets and the codelets. They are **public** in `module::Stateless` and
**protected** in `module::Stateful`.

## Stateless Modules

A `module::Stateless` object is an empty module whose methods to create tasks
are public. Its codelets cannot rely on inner data (except constant values
captured by copy in the lambda), so a stateless module is always
[clonable](concepts_overview.md#module-states-tasks-replicability) and its
tasks can be replicated.

```cpp
const size_t n_elmts = 1024;
const float gain = 2.f;

module::Stateless amplifier;
amplifier.set_name("Amplifier");      // optional, used in the logs, the statistics and the DOT export
amplifier.set_short_name("Amp");

auto& t = amplifier.create_task("amplify");
const size_t s_in  = amplifier.create_socket_in <float>(t, "in",  n_elmts);
const size_t s_out = amplifier.create_socket_out<float>(t, "out", n_elmts);
amplifier.create_codelet(t,
    [s_in, s_out, gain](module::Module& m, runtime::Task& t, const size_t frame_id) -> int
    {
        const float* in  = t[s_in ].get_dataptr<const float>();
        float*       out = t[s_out].get_dataptr<float>();
        for (size_t i = 0; i < n_elmts; i++)
            out[i] = gain * in[i];
        return runtime::status_t::SUCCESS;
    });
```

!!! warning
    When the module is cloned (replication), the lambda is **copied**. Capture
    values by copy (`[=]` or explicit copies), not references to local
    variables that could be destroyed or shared between threads.

## Stateful Modules

A stateful module is a C++ class that inherits from `module::Stateful`. The
tasks and their sockets are created in the constructor. The codelet receives
a reference to the module (`module::Module& m`), which has to be cast to the
right type to access the inner data.

```cpp
#include <streampu.hpp>
using namespace spu;

// accumulates its input over the executions
template<typename T>
class Accumulator : public module::Stateful
{
  protected:
    const size_t n_elmts;
    std::vector<T> acc; // inner data (= state)

  public:
    Accumulator(const size_t n_elmts)
      : Stateful()
      , n_elmts(n_elmts)
      , acc(n_elmts, 0)
    {
        this->set_name("Accumulator");
        this->set_short_name("Accumulator");

        auto& t = this->create_task("accumulate");
        const auto s_in  = this->template create_socket_in <T>(t, "in",  n_elmts);
        const auto s_out = this->template create_socket_out<T>(t, "out", n_elmts);
        this->create_codelet(t,
            [s_in, s_out](module::Module& m, runtime::Task& t, const size_t frame_id) -> int
            {
                auto& self = static_cast<Accumulator<T>&>(m);
                const T* in  = t[s_in ].template get_dataptr<const T>();
                T*       out = t[s_out].template get_dataptr<T>();
                for (size_t i = 0; i < self.n_elmts; i++)
                    out[i] = (self.acc[i] += in[i]);
                return runtime::status_t::SUCCESS;
            });
    }

    virtual ~Accumulator() = default;

    // optional: makes the module clonable and its tasks replicable
    virtual Accumulator<T>* clone() const
    {
        auto m = new Accumulator<T>(*this); // copy the attributes (the state)
        m->deep_copy(*this);                // clone the tasks and the sockets
        return m;
    }
};
```

!!! important
    Always use the module reference given to the codelet (`m`), never `this`
    captured in the lambda: when the module is cloned, the codelet of the clone
    must work on the clone, not on the original module.

### Making a Stateful Module Clonable

Without `clone()`, the tasks of a stateful module are not replicable: the
module can only be used in a sequence executed by one thread, or in a pipeline
stage executed by one thread. To enable the replication, override the
`clone()` method:

1. call the copy constructor (`new MyModule(*this)`): it copies the attributes,
   including the state;
2. call `deep_copy(*this)`: it clones the tasks and their sockets and attaches
   them to the new module;
3. if the class contains pointers or non-copyable objects, fix them after the
   copy (for instance by overriding `deep_copy()` and calling
   `Stateful::deep_copy()` in it).

The tasks of a clonable module are replicable by default. A given task can
still be forced to be non-replicable with
`this->tasks[i]->set_replicability(false)`.

!!! warning
    The replicability of a task is computed in `create_task()` by calling
    `clone()`: `clone()` must work on a partially constructed object (that is,
    from the constructor, right after `create_task()`).

If a class inherits from a non-clonable module (for instance
`module::Source<B>`) and implements `clone()`, the tasks created by the
parent constructor remain non-replicable: call
`this->tasks[i]->set_replicability(true)` at the end of the child
constructor (this is what `module::Source_AZCW` does).

## API to Create Tasks, Sockets and Codelets

### Tasks

```cpp
runtime::Task& create_task(const std::string& name, const int id = -1);
```

Creates a new task in the module. Two tasks of a module cannot have the same
name. The optional `id` places the task at a given position in the
`tasks_with_nullptr` vector (used by `module[id]`); this is useful when tasks
are created conditionally. `create_tsk()` is an alias.

### Sockets

```cpp
template <typename T>
size_t create_socket_in (runtime::Task& task, const std::string& name, const size_t n_elmts);
template <typename T>
size_t create_socket_out(runtime::Task& task, const std::string& name, const size_t n_elmts);
template <typename T>
size_t create_socket_fwd(runtime::Task& task, const std::string& name, const size_t n_elmts);
```

Create an input, output or forward socket of `n_elmts` elements of type `T`
**per frame**. They return the index of the socket in the task, which can be
used to access it quickly in the codelet (`t[index]`). The sockets must be
created before the codelet.

Each method has two non-template overloads where the type is given at runtime,
as a last parameter:

```cpp
size_t create_socket_in(runtime::Task& task, const std::string& name, const size_t n_elmts,
                        const std::type_index& datatype);          // ex: typeid(float)
size_t create_socket_in(runtime::Task& task, const std::string& name, const size_t n_elmts,
                        const runtime::datatype_t datatype);       // ex: runtime::datatype_t::F32
```

`create_sck_in()`, `create_sck_out()` and `create_sck_fwd()` are aliases.

#### 2D Sockets

```cpp
template <typename T>
size_t create_2d_socket_in (runtime::Task& task, const std::string& name, const size_t n_rows, const size_t n_cols);
template <typename T>
size_t create_2d_socket_out(runtime::Task& task, const std::string& name, const size_t n_rows, const size_t n_cols);
template <typename T>
size_t create_2d_socket_fwd(runtime::Task& task, const std::string& name, const size_t n_rows, const size_t n_cols);
```

A 2D socket contains `n_rows × n_cols` elements per frame, stored contiguously
(row-major). In the codelet, `get_2d_dataptr<T>()` returns an array of row
pointers (`T**`, with `T` `const` for an input socket); `get_dataptr<T>()`
still returns the contiguous memory.
The same runtime-type overloads and `create_2d_sck_*()` aliases exist.

### Codelets

```cpp
void create_codelet(runtime::Task& task,
                    std::function<int(module::Module& m, runtime::Task& t, const size_t frame_id)> codelet);
```

Sets the function executed by the task. Creating the codelet also creates the
`status` output socket: no socket can be added after this call.
`create_cdl()` is an alias.

The codelet parameters are:

- `m`: the module that owns the task (the clone, when the module has been
  replicated);
- `t`: the task, used to access the sockets (`t[socket_index]` or
  `t["socket_name"]`);
- `frame_id`: the index of the first frame processed by this call (see
  [Frames and Waves](#frames-and-waves)).

The returned integer is written in the `status` socket (see
[The `status` Socket](concepts_overview.md#the-status-socket)).

### Accessing the Socket Data

| Method                                        | Description                                                         |
| --------------------------------------------- | ------------------------------------------------------------------- |
| `T* get_dataptr<T>(const size_t start_col = 0)` | Pointer to the data (`T` must be `const` for an input socket). `get_dptr<T>()` is an alias. |
| `T** get_2d_dataptr<T>(const size_t start_row = 0, const size_t start_col = 0)` | Array of row pointers (2D sockets). `get_2d_dptr<T>()` is an alias. |
| `size_t get_n_elmts()`                        | Number of elements of the socket (all frames included).             |
| `size_t get_databytes()`                      | Number of bytes of the socket (all frames included).                |
| `size_t get_n_rows()`                         | Number of rows (all frames included).                               |
| `const std::type_index& get_datatype()`       | Data type of the socket.                                            |
| `const std::string& get_name()`               | Name of the socket.                                                 |

!!! warning
    Calling `get_dataptr<T>()` with a non-`const` `T` on an input socket throws
    an exception (unless `SPU_FAST` is defined). The non-template
    `get_dataptr()` overloads are deprecated.

### Other Protected Methods

| Method                                          | Description                                                              |
| ----------------------------------------------- | ------------------------------------------------------------------------ |
| `void set_name(const std::string&)`             | Sets the name of the module (by convention, the same for all the instances of a class). |
| `void set_short_name(const std::string&)`       | Sets the short name of the module (displayed by `tools::Stats::show()`, `"Module"` by default). |
| `void register_timer(runtime::Task&, const std::string& key)` | Registers a named sub-timer in a task (see below).         |
| `void set_n_frames_per_wave(const size_t)`      | Sets the number of frames processed by one codelet call.                 |
| `void set_single_wave(const bool)`              | If `true`, one codelet call processes all the frames.                    |
| `void deep_copy(const Module&)`                 | Clones the tasks of the given module into the current one (stateful only). |

The public `set_custom_name(const std::string&)` method sets a name per
instance; it is displayed instead of the class name when it is not empty.

#### Sub-timers

A task can measure the duration of parts of its codelet. The timer is
registered after the codelet creation, and updated in the codelet with
`Task::update_timer(id, duration)` (`id` being the registration order). The
measures are recorded only when the statistics of the task are enabled and
are displayed by `tools::Stats::show()` (see
[Debugging & Profiling](debugging.md#statistics)).

```cpp
this->create_codelet(t, [](module::Module& m, runtime::Task& t, const size_t frame_id) -> int
{
    auto t_start = std::chrono::steady_clock::now();
    // ... first part of the processing ...
    t.update_timer(0, std::chrono::steady_clock::now() - t_start);
    return runtime::status_t::SUCCESS;
});
this->register_timer(t, "first_part");
```

#### Reset Task

If a module inherits from `tools::Interface_reset` (and implements
`void reset()`), the public `create_reset_task()` method adds a `reset` task
to it. Its codelet simply calls `reset()`. This makes it possible to reset a
module from the dataflow graph (by binding the `reset` task to another task).

## Frames and Waves

A socket created with `n_elmts` elements actually contains
`n_frames × n_elmts` elements (`n_frames` is set with `set_n_frames()` on the
module, the sequence or the pipeline). There are two ways to process these
frames in a codelet.

**Default behavior (one call per frame).** The codelet is called `n_frames`
times per task execution. Before each call, the socket pointers are moved to
the current frame and `frame_id` is the index of this frame. The codelet only
processes `n_elmts` elements. This is the case of the examples above.

**Single wave (one call for all the frames).** If the module constructor calls
`set_single_wave(true)`, the codelet is called once with `frame_id = 0`, and
the socket pointers give access to all the frames:

```cpp
this->set_single_wave(true);
// ...
this->create_codelet(t, [s_in, s_out](module::Module& m, runtime::Task& t, const size_t frame_id) -> int
{
    const size_t n_frames = m.get_n_frames();
    const float* in  = t[s_in ].get_dataptr<const float>();
    float*       out = t[s_out].get_dataptr<float>();
    for (size_t f = 0; f < n_frames; f++)
        for (size_t i = 0; i < n_elmts; i++)
            out[f * n_elmts + i] = in[f * n_elmts + i]; // frames are contiguous
    return runtime::status_t::SUCCESS;
});
```

**Waves of `n` frames.** `set_n_frames_per_wave(n)` is the general case: the
codelet is called once per group of `n` frames (`get_n_waves()` calls,
`frame_id` being the first frame of the wave). If `n_frames` is not a multiple
of `n`, the last wave is incomplete: its data are copied into a temporary
buffer of `n` frames, so the codelet can always process `n` frames.

The `status` socket contains one integer per wave.

!!! info
    If a codelet returns `runtime::status_t::FAILURE_STOP`, the remaining waves
    of the current task execution are not processed.

## Executing a Task Directly

A task can be executed outside of a sequence with `Task::exec()`, for
instance for unit tests. The output buffers are allocated by the sequences;
to execute a task alone, the *task autoalloc* mode has to be enabled **before
the creation of the modules**:

```cpp
tools::Buffer_allocator::set_task_autoalloc(true);

module::Incrementer<int> inc(4);
std::vector<int> in = { 1, 2, 3, 4 };
inc["increment::in"] = in;
inc("increment").exec();
const int* out = inc["increment::out"].get_dataptr<const int>(); // { 2, 3, 4, 5 }
```

Most built-in modules also provide helper methods that bind user data and
execute a task in one call, for instance
`Incrementer<T>::increment(const std::vector<T>& in, std::vector<T>& out)`.

`Task::exec(const int frame_id = -1, const bool managed_memory = true)`
processes all the frames when `frame_id == -1`, or only the frame `frame_id`
otherwise. It returns the content of the `status` socket.

## Inspecting a Module

`tools::help()` prints the tasks and sockets of a module (or of a task, or of
a socket):

```cpp
module::Incrementer<int> inc(16);
tools::help(inc);
```

```
# module:
# |- name: Incrementer
# |- short name: Incrementer
# |- n_frames: 1
# |- tasks:
# |  |- name: increment
# |  |  |- sockets in:
# |  |  |  |- in: { n_elts = 16, datatype = int32 }
# |  |  |- sockets out:
# |  |  |  |- out: { n_elts = 16, datatype = int32 }
# |  |  |  |- status: { n_elts = 1, datatype = int32 }
# |  |- name: incrementf
# |  |  |- sockets in:
# |  |  |  |- fwd: { n_elts = 16, datatype = int32 }
# |  |  |- sockets out:
# |  |  |  |- status: { n_elts = 1, datatype = int32 }
```

See [Debugging & Profiling](debugging.md#inspecting-modules) for more details.
