# Debugging & Profiling

This page presents the tools provided by `StreamPU` to inspect, debug and
profile an application.

## Inspecting Modules

`tools::help()` prints the tasks and the sockets of a module, of a task or of
a socket (`verbose = true` adds the addresses and the data pointers):

```cpp
void tools::help(const module::Module& mdl, const bool& verbose = false, std::ostream& stream = std::cout);
void tools::help(const runtime::Task& tsk, const bool& verbose = false, std::ostream& stream = std::cout);
void tools::help(const runtime::Socket& sck, const bool& verbose = false, std::ostream& stream = std::cout);
```

`tools::get_help()` returns the same text as a `std::string`. Example of
output for `tools::help(module::Incrementer<int>(16))`:

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

## Exporting the Graph

`Sequence::export_dot()` and `Pipeline::export_dot()` write the graph in the
[DOT format](https://graphviz.org/). It is the easiest way to check the
bindings and the execution order of the tasks:

```cpp
std::ofstream file("graph.dot");
seq.export_dot(file);
```

```bash
dot -Tsvg graph.dot -o graph.svg
```

## Debug Mode

When the debug mode of a task is enabled, each execution of the task prints
the content of its input sockets before the codelet, the content of its output
sockets after the codelet, and the returned statuses:

```cpp
for (auto& mod : seq.get_modules<module::Module>(false))
    for (auto& tsk : mod->tasks)
    {
        tsk->set_debug(true);     // enable the debug mode
        tsk->set_debug_limit(4);  // display only the 4 first elements of each frame
    }
```

Example with 2 frames:

```
# Initializer::initialize(int32 out[2x8])
# {OUT} out = [f1_w1(    3,     3,     3,     3, ...),
#              f2_w2(    3,     3,     3,     3, ...)]
# Returned status: [0 'SUCCESS', 0 'SUCCESS']
#
# Incrementer::increment(const int32 in[2x8], int32 out[2x8])
# {IN}  in  = [f1_w1(    3,     3,     3,     3, ...),
#              f2_w2(    3,     3,     3,     3, ...)]
# {OUT} out = [f1_w1(    4,     4,     4,     4, ...),
#              f2_w2(    4,     4,     4,     4, ...)]
# Returned status: [0 'SUCCESS', 0 'SUCCESS']
#
```

`fX_wY` means frame `X` in wave `Y`. The debug options of a task are:

| Method | Description |
| ------ | ----------- |
| `set_debug(bool)` | Enables the debug mode (default: disabled). |
| `set_debug_limit(uint32_t)` | Maximum number of elements displayed per frame (default: all). |
| `set_debug_frame_max(uint32_t)` | Maximum number of frames displayed (default: all). |
| `set_debug_precision(uint8_t)` | Number of decimals for the floating-point values (default: 2). |
| `set_debug_hex(bool)` | Displays the values in hexadecimal. |

!!! tip
    The debug mode is much easier to read with one thread. With several
    threads, the outputs of the threads are mixed. Use
    [`exec_seq()` or `exec_step()`](sequence_usage.md#step-by-step-execution)
    to execute the sequence step by step.

## Statistics

When the statistics of a task are enabled, its execution time is measured
(number of calls, total, minimum, maximum and average durations, and the
[sub-timers](custom_modules.md#sub-timers)):

```cpp
for (auto& mod : seq.get_modules<module::Module>(false))
    for (auto& tsk : mod->tasks)
    {
        tsk->reset();         // reset the previous statistics
        tsk->set_stats(true); // enable the statistics
    }

seq.exec(/* ... */);

tools::Stats::show(seq.get_modules_per_types(), true /* ordered */, false /* display throughput */);
```

```
# ---------------------------------------------------------------||------------------------------||--------------------------------
#                  Statistics for the given task                 ||       Basic statistics       ||        Measured latency
#               ('*' = any, '-' = same as previous)              ||          on the task         ||
# ---------------------------------------------------------------||------------------------------||--------------------------------
# -------------------|-------------------|-----|-------|---------||----------|----------|--------||----------|----------|----------
#        MODULE NAME |         TASK NAME | REP | ORDER |   TIMER ||    CALLS |     TIME |   PERC ||  AVERAGE |  MINIMUM |  MAXIMUM
#                    |                   |     |       |         ||          |      (s) |    (%) ||     (us) |     (us) |     (us)
# -------------------|-------------------|-----|-------|---------||----------|----------|--------||----------|----------|----------
#            Relayer |             relay | yes |     2 |       * ||      100 |     0.01 |  59.45 ||    60.35 |    60.14 |    75.11
#            Relayer |             relay | yes |     3 |       * ||      100 |     0.00 |  20.22 ||    20.53 |    20.13 |    48.74
...
```

`tools::Stats` is an alias of `tools::Statistics`:

```cpp
template <class MODULE_OR_TASK>
static void show(std::vector<MODULE_OR_TASK*> modules_or_tasks,
                 const bool ordered = false, const bool display_thr = true, std::ostream& stream = std::cout);
template <class MODULE_OR_TASK>
static void show(std::vector<std::vector<MODULE_OR_TASK*>> modules_or_tasks,
                 const bool ordered = false, const bool display_thr = true, std::ostream& stream = std::cout);
```

`MODULE_OR_TASK` can be `module::Module` or `runtime::Task` (`const` or not).
With a vector of vectors (for instance `get_modules_per_types()` or
`get_tasks_per_types()`), the statistics of the replicas of a module or task are
merged in one line. `ordered = true` sorts the lines by decreasing total time;
`display_thr = true` adds a throughput column group (in Mb/s: the number of
elements of the **last data socket** of the task (the one before `status`),
each element counted as one bit, divided by the measured durations). The
`MODULE NAME` column displays the **short name** of the modules.

The statistics of a task are also available through its methods:
`get_n_calls()`, `get_duration_total()`, `get_duration_avg()`,
`get_duration_min()`, `get_duration_max()` (and `get_timers_*()` for the
sub-timers). `reset()` clears them.

!!! note
    Measuring the time has a cost: enable the statistics only when needed.

## Fast Mode

By default, `Task::exec()` checks that the task can be executed (all the
sockets have data), that the frame index is valid, and the sockets check the
types and sizes when they are bound. The fast mode disables these checks to
reduce the runtime overhead:

```cpp
tsk->set_fast(true);   // for one task (and its sockets)
mdl.set_fast(true);    // for all the tasks of a module
```

When the debug mode or the statistics of a task are enabled, its frame index
is still checked. Compiling `StreamPU` with the CMake option `-DSPU_FAST=ON` removes
the checks at compile time (most of the checks in the library are removed, and
`is_fast()` always returns `true`).

## Exceptions

`StreamPU` throws exceptions derived from `spu::tools::exception` (which
derives from `std::exception`, or from `cpptrace::exception_with_message` when
`SPU_STACKTRACE` is defined):

| Exception | Default message |
| --------- | --------------- |
| `tools::cannot_allocate` | Cannot allocate the object. |
| `tools::control_flow_error` | Control flow error. |
| `tools::domain_error` | Domain error. |
| `tools::invalid_argument` | Given argument is invalid. |
| `tools::length_error` | Length error. |
| `tools::logic_error` | Logic error. |
| `tools::out_of_range` | Out of range. |
| `tools::overflow_error` | Overflow error. |
| `tools::processing_aborted` | Processing aborted. |
| `tools::range_error` | Range error. |
| `tools::runtime_error` | Runtime error. |
| `tools::underflow_error` | Underflow error. |
| `tools::unimplemented_error` | Unimplemented function or method. |
| `tools::waiting_canceled` | Waiting canceled thread. |

They are constructed with the location of the error:

```cpp
std::stringstream message;
message << "'n_elmts' has to be greater than 0 ('n_elmts' = " << n_elmts << ").";
throw tools::invalid_argument(__FILE__, __LINE__, __func__, message.str());
```

The message is formatted as
`In the '<file>' file at line <line> ('<function>' function): "<message>"`.
When `StreamPU` is compiled with `SPU_STACKTRACE=ON` (default), `what()` also
contains the stack trace of the location where the exception has been
created. The static attribute `tools::exception::no_stacktrace` (default
`false`) disables the stack trace in `what()`.

Inside `Sequence::exec()` and `Pipeline::exec()`, the exceptions raised by the
tasks are caught, all the threads are stopped, and a `std::runtime_error`
containing the message of the exception is thrown by `exec()` (see
[Sequences](sequence_usage.md#exceptions)). Two exceptions have a special
meaning: `processing_aborted` (a codelet can throw it to abandon the current
execution of the sequence; the loop continues) and `waiting_canceled` (used
to stop the pipelines).

With `SPU_STACKTRACE_SEGFAULT=ON`, the [signal handler](#signal-handler) also
tries to print the stack trace when a segmentation fault occurs.

## Signal Handler

```cpp
tools::Signal_handler::init();
```

When this function is called (usually at the beginning of `main()`), a
`SIGINT` (`Ctrl+C`) does not kill the application anymore: it cleanly stops
the running `Sequence::exec()` or `Pipeline::exec()` calls, which return
normally. A second `Ctrl+C` within 250 ms kills the application.

| Function | Description |
| -------- | ----------- |
| `static int init()` | Installs the handlers (`SIGINT`, and `SIGSEGV` with `SPU_STACKTRACE_SEGFAULT`). |
| `static bool is_sigint()` | `true` if a `SIGINT` has been caught. |
| `static void reset_sigint()` | Resets the `SIGINT` flag (done automatically at the beginning of each `exec()`). |
| `static bool is_sigsegv()`, `static void reset_sigsegv()` | Same for `SIGSEGV`. |

After `exec()`, `tools::Signal_handler::is_sigint()` tells whether the
execution has been interrupted by the user.

## Deprecated Functions

The CMake option `-DSPU_SHOW_DEPRECATED=ON` prints a warning (with the stack
trace if `SPU_STACKTRACE` is enabled) each time a deprecated function is
called, for instance `Socket::bind()` (replaced by the `=` operator) or the
non-template `Socket::get_dataptr()`.
