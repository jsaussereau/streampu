# Sequences

A **sequence** (`spu::runtime::Sequence`) executes a graph of bound tasks in a
fixed order, possibly replicated over several threads. This page describes how
to use it. The internal implementation is described in the developer page
[Sequence](sequence.md).

<figure markdown>
  ![Simple sequence](./assets/simple_sequence.svg){ width="600" }
  <figcaption>Example of a simple sequence of tasks (single threaded).</figcaption>
</figure>

## Creating a Sequence

### Main Constructors

```cpp
Sequence(runtime::Task& first,
         const size_t n_threads = 1,
         const bool thread_pinning = false,
         const std::vector<size_t>& puids = {},
         const bool tasks_inplace = true,
         const bool memory_allocation = true);

Sequence(runtime::Task& first,
         runtime::Task& last,
         /* same optional parameters */);

Sequence(const std::vector<runtime::Task*>& firsts,
         /* same optional parameters */);

Sequence(const std::vector<runtime::Task*>& firsts,
         const std::vector<runtime::Task*>& lasts,
         /* same optional parameters */);

Sequence(const std::vector<runtime::Task*>& firsts,
         const std::vector<runtime::Task*>& lasts,
         const std::vector<runtime::Task*>& exclusions,
         /* same optional parameters */);
```

The parameters are:

- `first`/`firsts`: the task(s) from which the graph is explored. A first task
  usually has no input socket (or only static input sockets bound to user
  data). At least one first task is required.
- `last`/`lasts` (optional): the exploration stops at these tasks, their
  outputs are not followed. Each last task must be reached during the
  exploration, otherwise an exception is raised. A sequence cannot end with a
  `commute` task (an exception is raised), nor with a `select` task (the
  exploration never stops on a `select` task, so it is never found among the
  reached last tasks and the previous exception is raised).
- `exclusions` (optional): tasks that are never added to the sequence, even if
  they are bound to tasks of the sequence. An excluded task cannot be a first
  or a last task.
- `n_threads`: the number of threads that execute the sequence. Each thread
  executes its own replica of the graph.
- `thread_pinning` and `puids`: pin the thread `i` on the processing unit
  `puids[i]` (see [Thread Pinning](thread_pinning.md)); `puids` must contain
  at least `n_threads` values when `thread_pinning` is `true`.
- `tasks_inplace`: if `true`, the thread 0 executes the original tasks (the
  ones given by the user) and the other threads execute clones. If `false`,
  all the threads execute clones and the original tasks are left untouched.
- `memory_allocation`: if `true`, the output buffers of all the tasks are
  allocated by the constructor (see [Memory](#memory)).

!!! note
    When the `firsts`, `lasts` and `exclusions` vectors are given as brace
    lists (`{ &t1 }, {}, { &t2 }`), the call may be ambiguous for the compiler.
    Use named `std::vector<runtime::Task*>` variables in this case.

### `const` Constructors

The same constructors exist with `const runtime::Task&` and
`std::vector<const runtime::Task*>` parameters (without the `tasks_inplace`
parameter). With these constructors, **all the threads execute clones**: the
original modules are never executed and are left untouched. As a consequence,
all the tasks of the graph must be replicable, even with `n_threads = 1`.

!!! warning
    The overload is selected by the C++ compiler: a non-`const` reference to a
    task selects the in-place constructor, a `const` reference selects the
    clone-only constructor. If the graph contains a non-replicable task (for
    instance a stateful module without `clone()`), use a non-`const` task and
    `n_threads = 1`.

### Constructors with a Pinning Policy

Each of the previous constructors has a variant where `puids` is replaced by a
`const std::string& sequence_pinning_policy`, for instance:

```cpp
Sequence(runtime::Task& first,
         const size_t n_threads,
         const bool thread_pinning,
         const std::string& sequence_pinning_policy,
         const bool tasks_inplace = true,
         const bool memory_allocation = true);
```

The syntax of the policy is described in [Thread Pinning](thread_pinning.md).
If `thread_pinning` is `true`, the policy cannot be empty.

### How the Graph is Explored

The constructor explores the graph from the first tasks, following the
bindings of the output and forward sockets (and the
[task-to-task bindings](getting_started.md#task-to-task-binding)). A task is
added to the execution order once **all its input and forward sockets** have
been fed by tasks already in the order (static input sockets, bound to user
data, are ignored). A `select` task is a special case: it is explored as soon
as it is reached through its **last** input socket (see
[Control Flow](control_flow.md#rules-and-limitations)). The tasks that are not
reachable from the first tasks are not part of the sequence.

!!! warning
    A task whose inputs are bound to a task outside of the sequence (neither
    reachable from the first tasks nor itself a first task) is never added to
    the sequence. In this case, give this task (or its producer) as an
    additional first task.

During the construction, the sequence also:

1. checks that the tasks can be replicated (if `n_threads > 1`, or with the
   `const` constructors);
2. clones the modules and replicates the bindings for each thread;
3. generates the internal *processes* (see [No-copy Mode](#no-copy-mode));
4. allocates the output buffers (if `memory_allocation` is `true`).

!!! important
    All the bindings must be made **before** the creation of the sequence.
    Modifying the bindings of a graph used by a sequence leads to undefined
    behavior.

## Executing a Sequence

### `exec()`

```cpp
void exec();
void exec(std::function<bool()> stop_condition);
void exec(std::function<bool(const std::vector<const int*>&)> stop_condition);
```

These methods execute the sequence **in loop** on `n_threads` threads (the
calling thread is the thread 0) and return when all the threads have stopped.
Each thread stops when one of the following conditions is met:

- the stop condition returns `true` (it is evaluated by each thread after each
  execution of its replica of the graph);
- the *auto-stop* mode is enabled (default) and one of the modules of the
  sequence reports that it is done (see below);
- a `SIGINT` signal (`Ctrl+C`) has been caught by the
  [signal handler](debugging.md#signal-handler);
- an exception has been raised by one of the threads (in this case all the
  threads stop).

`exec()` without parameter only stops for the three last reasons.

The stop condition is shared by all the threads and called concurrently: it
must be thread-safe (use atomic variables). For instance, to execute the
sequence about `n` times in total:

```cpp
std::atomic<size_t> counter(0);
seq.exec([&counter, n]() { return ++counter >= n; });
```

The second form of the stop condition receives the statuses of the tasks of
the last execution: `statuses[i]` points to the `status` socket of the task
whose identifier is `i` in the sequence, or is `nullptr` if this task has not
been executed (when it belongs to an exclusive path that has not been taken).

```cpp
seq.exec([](const std::vector<const int*>& statuses)
{
    return statuses.back() != nullptr && statuses.back()[0] == runtime::status_t::FAILURE;
});
```

### Auto-stop

When the auto-stop mode is enabled (default), the sequence collects all the
modules that implement the `tools::Interface_is_done` interface (the
[`Source`](modules_catalog.md#sources) modules). The execution stops when one
of them returns `true` from `is_done()` (for instance, when a
`Source_user_binary` reaches the end of its file and `auto_reset` is
`false`).

```cpp
void set_auto_stop(const bool auto_stop);
bool is_auto_stop() const;
bool is_done() const; // true if one of the modules is done
```

### Exceptions

If a codelet throws an exception, the thread that caught it stops the other
threads (they finish their current execution of the graph), and `exec()`
throws a `std::runtime_error` containing the message of the exception once all
the threads are joined. Two exception types are handled
silently: `tools::processing_aborted` (the current execution is abandoned and
the loop continues) and `tools::waiting_canceled` (used internally to stop the
pipelines).

### Step-by-step Execution

```cpp
void exec_seq(const size_t tid = 0, const int frame_id = -1);
runtime::Task* exec_step(const size_t tid = 0, const int frame_id = -1);
```

`exec_seq()` executes the replica `tid` of the sequence **once**, on the
calling thread. `exec_step()` executes only the next task of the replica `tid`
and returns a pointer to this task, or `nullptr` when the end of the sequence
is reached (the next call restarts from the first task). This is useful for
debugging:

```cpp
for (size_t tid = 0; tid < seq.get_n_threads(); tid++)
    while (runtime::Task* t = seq.exec_step(tid))
        std::cout << "executed: " << t->get_name() << std::endl;
```

These methods are executed by the calling thread only. They ignore the stop
conditions and the auto-stop mode, and they do not catch the
`tools::processing_aborted` exceptions. The [no-copy mode](#no-copy-mode) is
not applied by these methods (the `Switcher` tasks copy their data).

!!! note
    Unlike `exec()`, these methods do not reset the path of the `Switcher`
    modules before executing the graph (see
    [Control Flow](control_flow.md#the-switcher-module)).

## Number of Frames

```cpp
size_t get_n_frames() const;
void set_n_frames(const size_t n_frames);
```

`set_n_frames()` sets the number of frames of all the modules of the sequence
(including the clones). The bindings are automatically preserved and the
buffers reallocated. All the modules of a sequence must have the same number of
frames (`get_n_frames()` throws otherwise). See
[Frames and Waves](concepts_overview.md#frames-and-waves).

## Accessing the Modules and Tasks

| Method | Description |
| ------ | ----------- |
| `size_t get_n_threads() const` | Number of threads (= replicas). |
| `std::vector<C*> get_modules<C>(const bool set_modules = true) const` | All the modules of type `C` (all threads). If `set_modules` is `true`, the modules inside [`Set`](modules_catalog.md#set) modules are also returned. |
| `std::vector<C*> get_cloned_modules<C>(const C& module_ref) const` | The replicas of a module (one per thread, in thread order). |
| `std::vector<std::vector<module::Module*>> get_modules_per_threads() const` | Modules, grouped by thread. |
| `std::vector<std::vector<module::Module*>> get_modules_per_types() const` | Modules, grouped by original module (one vector per module, one element per thread). |
| `std::vector<std::vector<runtime::Task*>> get_tasks_per_threads() const` | Tasks, grouped by thread, in execution order. |
| `std::vector<std::vector<runtime::Task*>> get_tasks_per_types() const` | Tasks, grouped by original task. |
| `const std::vector<std::vector<runtime::Task*>>& get_firsts_tasks() const` | First tasks of each thread. |
| `const std::vector<std::vector<runtime::Task*>>& get_lasts_tasks() const` | Last tasks of each thread. |
| `bool is_control_flow() const` | `true` if the sequence contains a `Switcher`. |

A typical use is to configure all the tasks after the creation of the
sequence:

```cpp
for (auto& mod : seq.get_modules<module::Module>(false))
    for (auto& tsk : mod->tasks)
    {
        tsk->set_stats(true); // enable the statistics
        tsk->set_fast(true);  // disable the runtime checks
    }
```

## No-copy Mode

```cpp
void set_no_copy_mode(const bool no_copy_mode);
bool is_no_copy_mode() const;
```

When the no-copy mode is enabled (default), the tasks of the
[`Switcher`](control_flow.md) (`commute` and `select`) and of the pipeline
adaptors (`push` and `pull`) do not copy their data: instead, the sequence
updates on the fly the data pointers of the sockets bound to them. This mode
is applied during `exec()` and the initial pointers are restored at the end
of `exec()` (at the end of `Pipeline::exec()` when the sequence is a pipeline
stage). Disabling it (`set_no_copy_mode(false)`) is mainly useful for
debugging and testing.

## Memory

```cpp
void allocate_outbuffers();
void deallocate_outbuffers();
```

By default (`memory_allocation = true`), the constructor allocates the output
buffers of all the tasks of all the threads. If the sequence is built with
`memory_allocation = false`, `allocate_outbuffers()` has to be called before
the execution. `deallocate_outbuffers()` releases the buffers. Each method
throws if the memory is already in the requested state.

## Exporting the Graph

```cpp
void export_dot(std::ostream& stream = std::cout) const;
```

Writes the graph of the thread 0 in the [DOT format](https://graphviz.org/).
The tasks are grouped by module and by sub-sequence; the execution order of
each task is displayed.

```cpp
std::ofstream file("graph.dot");
seq.export_dot(file);
```

```bash
dot -Tsvg graph.dot -o graph.svg
```

## Other Methods

| Method | Description |
| ------ | ----------- |
| `Sequence* clone() const` | Creates a new sequence (with cloned modules) from the same first, last and excluded tasks. |
| `void set_thread_pinning(const bool, const std::vector<size_t>& puids)` | Changes the thread pinning (applied at the next `exec()`). `puids` has to be given: a call with the boolean only is ambiguous (an overload without `puids` is declared but not implemented). |
| `void set_thread_pinning(const bool, const std::string& policy)` | Same, with a pinning policy. |
| `bool is_thread_pinning()` | Returns `true` if the thread pinning is enabled. |
| `bool has_julia() const` | Returns `true` if the sequence contains [Julia modules](julia.md). |

!!! note
    A sequence is used as a building block by the [`Set`](modules_catalog.md#set)
    module (a sequence seen as a single task), and each stage of a
    [pipeline](pipeline_usage.md) is a sequence.
