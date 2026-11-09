# Control Flow

By default, a [sequence](sequence_usage.md) executes all its tasks, in the
same order, for each stream. `StreamPU` also supports **conditional
execution** and **loops** through the `module::Switcher` module. With control
flow, the DSEL is Turing-complete.

The internal implementation (sub-sequences, digraph) is described in the
developer page [Switcher](switcher.md).

## The `Switcher` Module

```cpp
Switcher(const size_t n_data_sockets, const size_t n_elmts, const std::type_index datatype);
Switcher(const size_t n_data_sockets,
         const size_t n_elmts_commute, const std::type_index datatype_commute,
         const size_t n_elmts_select,  const std::type_index datatype_select);
```

A switcher has two tasks that create and join **exclusive paths** (only one
path is executed for a given stream):

- **`commute`** creates `n_data_sockets` paths. It reads the index of the path
  to take on its `in_ctrl` socket (one `int8_t`, taken modulo
  `n_data_sockets`), copies its `in_data` socket
  into the corresponding `out_data{path}` socket, and returns the path index
  in its `status` socket. Only the tasks bound to `out_data{path}` are then
  executed.
- **`select`** joins `n_data_sockets` paths. It copies its `in_data{path}`
  socket into its `out_data` socket, where `path` is the last path selected by
  the `commute` task of the same switcher.

| Task      | Sockets (in creation order)                                                               |
| --------- | ----------------------------------------------------------------------------------------- |
| `commute` | `in_data` (in), `in_ctrl` (in, `int8_t`, 1 element), `out_data0` ... `out_dataN-1` (out), `status` |
| `select`  | `in_data0` ... `in_dataN-1` (in), `out_data` (out), `status`                              |

The data sockets of `commute` have `n_elmts_commute` elements of type
`datatype_commute`, and those of `select` have `n_elmts_select` elements of
type `datatype_select` (the first constructor uses the same values for both
tasks).

```cpp
module::Switcher swi(2, 1024, typeid(float)); // 2 paths, 1024 floats per frame
swi["commute::in_data"];  swi["commute::in_ctrl"];
swi["commute::out_data0"]; swi["commute::out_data1"];
swi["select::in_data0"];  swi["select::in_data1"];  swi["select::out_data"];
// or by index (the enumeration only exists for the tasks):
swi[module::swi::tsk::commute][2]; // = "commute::out_data0"
```

!!! important
    The current path of a switcher is reset to `n_data_sockets - 1` (the
    **last** path) at the beginning of each execution of the sequence by
    `exec()` (not by `exec_seq()`/`exec_step()`). A
    `select` task that is executed before its `commute` task (which is the
    case in a loop) thus takes its data from its **last** input socket the
    first time.

The `Switcher` is a single-wave module: with several frames, the path is
the same for all the frames of a stream (it is read in the first frame of the
`in_ctrl` socket).

## Control Modules

The `in_ctrl` socket of a `commute` task is fed by a task that produces an
`int8_t`. The following built-in modules produce such values (see the
[module catalog](modules_catalog.md#controllers) for details):

| Module                                   | Task      | Produced value                                                      |
| ---------------------------------------- | --------- | ------------------------------------------------------------------- |
| `Iterator(limit)`                        | `iterate` | `0` for `limit` calls, then `1` once, then again (loop counter).    |
| `Controller_static(path)`                | `control` | always `path` (can be changed with `set_path()`).                   |
| `Controller_cyclic(n_paths, init_path)`  | `control` | `init_path`, `init_path + 1`, ... modulo `n_paths`.                 |
| `Controller_limit(limit, init_path)`     | `control` | `init_path` for `limit` calls, then `!init_path`.                   |

Any user task that writes an `int8_t` can also be used (for instance, a
task that tests a condition on the data).

## Patterns

The following examples are extracted from the tests of the `tests/bootstrap/`
folder. They all use `Incrementer` tasks (`+1` on the data) between an
`Initializer` and a `Finalizer`.

### `for` Loop

<figure markdown>
  ![for loop](./assets/test_for_loop.svg){ width="1000" }
</figure>

The condition is evaluated before the body of the loop: the `select` task
(entry of the loop) is followed by the `iterate` task, whose output controls
the `commute` task. Path 0 executes the body and goes back to the `select`
task, path 1 exits the loop.

```cpp
module::Initializer<uint8_t> initializer(data_length);
module::Finalizer<uint8_t> finalizer(data_length);
module::Switcher switcher(2, data_length, typeid(uint8_t));
module::Iterator iterator(n_loop);
std::vector<std::shared_ptr<module::Incrementer<uint8_t>>> incs(6);
for (auto& inc : incs)
    inc.reset(new module::Incrementer<uint8_t>(data_length));

switcher["select::in_data1"] = initializer["initialize::out"];   // entry of the loop (last path)
iterator("iterate") = switcher("select");                        // 'iterate' is executed after 'select'
switcher["commute::in_data"] = switcher["select::out_data"];
switcher["commute::in_ctrl"] = iterator["iterate::out"];
(*incs[0])["increment::in"] = switcher["commute::out_data0"];    // path 0: body of the loop
for (size_t s = 0; s < incs.size() - 1; s++)
    (*incs[s + 1])["increment::in"] = (*incs[s])["increment::out"];
switcher["select::in_data0"] = (*incs[incs.size() - 1])["increment::out"]; // back to the loop entry
finalizer["finalize::in"] = switcher["commute::out_data1"];      // path 1: exit of the loop

runtime::Sequence sequence(initializer("initialize"), n_threads);
```

Note the [task-to-task binding](getting_started.md#task-to-task-binding)
`iterator("iterate") = switcher("select")`: the `iterate` task has no input
socket, this binding forces it to be executed after `select` in each
iteration.

### `do-while` Loop

<figure markdown>
  ![do while loop](./assets/test_do_while_loop.svg){ width="1000" }
</figure>

The body of the loop is executed before the condition:

```cpp
switcher["select::in_data1"] = initializer["initialize::out"];
iterator("iterate") = switcher("select");
switcher["commute::in_ctrl"] = iterator["iterate::out"];
(*incs[0])["increment::in"] = switcher["select::out_data"];      // body of the loop
for (size_t s = 0; s < incs.size() - 1; s++)
    (*incs[s + 1])["increment::in"] = (*incs[s])["increment::out"];
switcher["commute::in_data"] = (*incs[incs.size() - 1])["increment::out"];
switcher["select::in_data0"] = switcher["commute::out_data0"];   // path 0: loop again
finalizer["finalize::in"] = switcher["commute::out_data1"];      // path 1: exit
```

### `switch-case`

<figure markdown>
  ![switch case](./assets/test_exclusive_paths.svg){ width="1000" }
</figure>

A `commute` task selects one of three paths, and a `select` task joins them:

```cpp
module::Controller_cyclic controller(3 /* n_paths */, 0 /* init_path */);
module::Switcher switchex(3, data_length, typeid(uint8_t));

controller("control") = initializer["initialize::out"];          // 'control' after 'initialize'
switchex["commute::in_data"] = initializer["initialize::out"];
switchex["commute::in_ctrl"] = controller["control::out"];
// path 0
(*incs[0])["increment::in"] = switchex["commute::out_data0"];
(*incs[1])["increment::in"] = (*incs[0])["increment::out"];
(*incs[2])["increment::in"] = (*incs[1])["increment::out"];
switchex["select::in_data0"] = (*incs[2])["increment::out"];
// path 1
(*incs[3])["increment::in"] = switchex["commute::out_data1"];
(*incs[4])["increment::in"] = (*incs[3])["increment::out"];
switchex["select::in_data1"] = (*incs[4])["increment::out"];
// path 2
(*incs[5])["increment::in"] = switchex["commute::out_data2"];
switchex["select::in_data2"] = (*incs[5])["increment::out"];
// end
finalizer["finalize::in"] = switchex["select::out_data"];
```

### Nested Loops

Loops can be nested by using one `Switcher` and one `Iterator` per loop (see
`tests/bootstrap/nested_loops.cpp` and
`tests/bootstrap/nested_do_while_loops.cpp`).

<figure markdown>
  ![nested loops](./assets/test_nested_loops.svg){ width="1000" }
</figure>

## Rules and Limitations

- The `commute` and `select` tasks of a switcher are always associated: a
  `select` task joins the paths created by the `commute` task of the **same**
  `Switcher` object.
- A sequence (or a pipeline stage) cannot **end** with a `commute` task.
- In a loop, the path that enters the loop must be bound to the **last** input
  socket of the `select` task (see the note above).
- In a pipeline, all the tasks of a switcher must be in the same stage. In
  some configurations, a `relay` task has to be added after a `select` task or
  before the first `select` task of a stage (see
  [Pipeline & Control Flow](pipeline_ctrl_flow.md) and
  [Work in Progress](wip.md#End-of-sequence)).
- The tasks bound directly to the data sockets of a switcher should use input
  and output sockets rather than forward sockets (see
  [Work in Progress](wip.md#forward-socket-with-control-flow)).
- `Sequence::is_control_flow()` returns `true` if a sequence contains control
  flow. The automatic [schedulers](schedulers.md) do not support control flow.
