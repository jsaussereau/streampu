# Getting Started

This page builds a first complete `StreamPU` program step by step. It assumes
that the library is [installed or available as a sub-directory](installation.md).

!!! tip
    The [slides](https://largo.lip6.fr/~cassagnea/docs/spu/StreamPU_slides_anim.pdf)
    used to teach `StreamPU` contain many illustrated use cases and are a good
    complement to this page. The sources of the [tests](tests_bootstrap.md)
    (`tests/` folder) are also good examples to start with.

## A First Program

The following program creates a chain of three tasks:

1. an `initialize` task (from the built-in `module::Initializer` module) that
   produces a frame of 8 integers;
2. a `double` task, written by the user with a lambda, that multiplies its
   input by 2;
3. a `finalize` task (from the built-in `module::Finalizer` module) that
   memorizes the last received frame.

The chain is then executed by 4 threads.

```cpp
#include <atomic>
#include <fstream>
#include <iostream>
#include <streampu.hpp>

using namespace spu;

int main()
{
    const size_t n_elmts = 8;

    // 1. modules creation -----------------------------------------------------
    module::Initializer<int32_t> initializer(n_elmts); // produces data
    module::Finalizer<int32_t> finalizer(n_elmts);     // memorizes the last received data

    module::Stateless doubler; // a module without inner data, built with a lambda
    doubler.set_name("Doubler");
    auto& t_double = doubler.create_task("double");
    const auto s_in = doubler.create_socket_in<int32_t>(t_double, "in", n_elmts);
    const auto s_out = doubler.create_socket_out<int32_t>(t_double, "out", n_elmts);
    doubler.create_codelet(t_double,
                           [s_in, s_out](module::Module& m, runtime::Task& t, const size_t frame_id) -> int
                           {
                               const auto in = t[s_in].get_dataptr<const int32_t>();
                               auto out = t[s_out].get_dataptr<int32_t>();
                               for (size_t i = 0; i < n_elmts; i++)
                                   out[i] = 2 * in[i];
                               return runtime::status_t::SUCCESS;
                           });

    initializer.set_init_data(21); // all the produced values will be 21

    // 2. sockets binding ------------------------------------------------------
    doubler["double::in"] = initializer["initialize::out"];
    finalizer["finalize::in"] = doubler["double::out"];

    // 3. sequence creation (the tasks graph is analyzed from the first task) ---
    const size_t n_threads = 4;
    runtime::Sequence seq(initializer("initialize"), n_threads);

    // 4. execution: the sequence is executed until the stop condition is true --
    std::atomic<size_t> counter(0);
    seq.exec([&counter]() { return ++counter >= 100; });

    // 5. check the results (one finalizer per thread) --------------------------
    for (auto fin : seq.get_cloned_modules<module::Finalizer<int32_t>>(finalizer))
        std::cout << "final data[0] = " << fin->get_final_data()[0][0] << std::endl;

    // 6. export the graph in the DOT format --------------------------------------
    std::ofstream f("graph.dot");
    seq.export_dot(f);

    return 0;
}
```

Expected output:

```
final data[0] = 42
final data[0] = 42
final data[0] = 42
final data[0] = 42
```

The DOT file can be converted into an image with Graphviz:
`dot -Tsvg graph.dot -o graph.svg`.

The next sections explain each step.

## Step 1: Creating the Modules

`StreamPU` comes with a set of [built-in modules](modules_catalog.md)
(sources, sinks, relayers, incrementers, controllers, switchers, probes,
...). Their tasks and sockets are created by their constructor. Most of them
are templates parameterized by the data type of their sockets.

The user can also create their own modules. The simplest way is to create a
`module::Stateless` object and to add tasks, sockets and codelets to it, as
shown with the `Doubler` module above. The other way is to write a class that
inherits from `module::Stateful`. Both ways are explained in
[Writing Modules](custom_modules.md).

The codelet of a task receives the module, the task and the frame index. The
data of a socket is obtained with `get_dataptr<T>()`, where `T` must be `const`
for input sockets. The codelet returns a status (`runtime::status_t::SUCCESS`
in most cases).

## Step 2: Binding the Sockets

The `=` operator binds an output socket (right side) to an input socket (left
side). The sockets can be accessed:

- with a string: `module["task_name::socket_name"]`;
- from the task: `module("task_name")["socket_name"]` or
  `module("task_name")[socket_id]` (the index returned by
  `create_socket_*()`);
- with the enumerations of the built-in modules, for instance
  `inc[module::inc::sck::increment::in]` for the `in` socket of the
  `increment` task of a `module::Incrementer`.

```cpp
module::Incrementer<int> inc(16);
module::Relayer<int> rly(16);
rly["relay::in"] = inc["increment::out"]; // with strings
// equivalent, with enums:
// rly[module::rly::sck::relay::in] = inc[module::inc::sck::increment::out];
```

!!! note
    An input socket can be bound to only one output socket: binding `relay::in`
    to a second output socket while it is still bound throws a
    `tools::runtime_error` (binding it again to the same output socket is
    silently accepted).

!!! note
    The `bind()` methods of the `Socket` and `Task` classes are deprecated: use
    the `=` operator instead.

An input socket can also be bound to user data (a `std::vector` or a raw
pointer). The vector size must be equal to the number of elements of the
socket (times the number of frames):

```cpp
std::vector<int> in_data(16, 0);
inc["increment::in"] = in_data;
```

A binding can be undone with `unbind()`:

```cpp
rly["relay::in"].unbind(inc["increment::out"]);
```

### Task-to-task Binding

Sometimes a task has to be executed after another one without any data
exchange (for instance, a task that reads a file written by a previous task).
This dependency is expressed by binding the two tasks:

```cpp
t2 = t1;                       // 't2' will be executed after 't1'
t2 = module["task1::out"];     // 't2' will be executed after the task owning the 'out' socket
```

Internally, a *fake* input socket is added to `t2` and bound to the `status`
socket of `t1` (or to the given output socket). The data of the fake input
sockets are ignored by the codelet.

## Step 3: Creating a Sequence

A `runtime::Sequence` is built from the **first task(s)** of the graph. The
constructor explores the graph by following the bindings of the output and
forward sockets, computes the execution order of the tasks, replicates the
graph for each thread (by cloning the modules) and allocates the memory of
the output sockets.

The `Sequence` constructors and options are detailed in
[Sequences](sequence_usage.md).

!!! warning
    The bindings must be done **before** the creation of the sequence. A
    sequence does not see the bindings made after its construction.

## Step 4: Executing

`exec()` executes the sequence in loop, on each thread, until the stop
condition returns `true`. Each thread evaluates the condition after each
execution of its replica of the graph. In the example, the condition is shared
by the 4 threads: this is why the counter is atomic, and why the total number
of executions can slightly exceed 100 (each thread stops at its next
evaluation of the condition).

Other variants of `exec()` exist; they are described in
[Sequences](sequence_usage.md#executing-a-sequence). It is also possible to
stop the execution with `Ctrl+C` if the [signal handler](debugging.md#signal-handler)
is enabled.

## Step 5: Getting the Results

When a sequence is executed by several threads, each thread works on its own
copy of the modules (*clones*). `get_cloned_modules<T>(module)` returns the
copies of a given module (one per thread). With the non-`const` constructors
(the default ones used in the example), the first element is the original
module.

## Going Further

- [Writing Modules](custom_modules.md): create your own stateless and stateful
  modules.
- [Sequences](sequence_usage.md): all the options of the `runtime::Sequence`
  class.
- [Pipelines](pipeline_usage.md): split a sequence into stages executed in
  parallel.
- [Control Flow](control_flow.md): loops and conditional execution.
- [Debugging & Profiling](debugging.md): debug mode, statistics, exceptions.
