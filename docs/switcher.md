# Switcher

A **switcher** is a control flow [module](module.md) used to break sequences 
into *exclusive paths* through its two tasks: [`select`](#Select) and 
[`commute`](#Commute).  

A switcher is a C++ object of the `spu::module::Switcher` class. The 
following sections give an overview of the most important attributes and 
methods to facilitate the code understanding. The user point of view (with
examples of loops and switch-case) is described in
[Control Flow](control_flow.md).

## Main Attributes

<a name="Path"></a>
```cpp
size_t path;
```
The *exclusive path* to take when the `commute` task is reached. Read on the
[`in_ctrl` socket](#Commute) of the `commute` task each time it is executed
(modulo `n_data_sockets`). The `path` value is reset to `n_data_sockets - 1`
by `reset()`, which `Sequence::exec()` calls before each execution of the graph
(`exec_seq()` and `exec_step()` do not).

```cpp
const size_t n_data_sockets;
```
The number of data sockets for the `commute` and `select` tasks (not the total, 
but the *individual* number).

```cpp
const std::type_index datatype_commute;
const std::type_index datatype_select;
```
The type of data conveyed by the data sockets of each task.

```cpp
const size_t n_elmts_commute;
const size_t n_elmts_select;
```
The number of elements conveyed by each data socket.
With `datatype` they define what and how much each data socket is expected to
read/write.

```cpp
const size_t n_bytes_commute;
const size_t n_bytes_select;
```
The product of the size of `datatype` with `n_elmts`: the number of bytes
expected on each data socket (per frame).

```cpp
bool no_copy_commute;
bool no_copy_select;
```
Set by the sequence in the [no-copy mode](sequence_usage.md#no-copy-mode): the
tasks do not copy the data, the sequence rebinds the sockets on the fly
instead.

## Tasks

Since those tasks have a variable number of sockets they are accessed through
the subscript `operator[]` with a numerical index or a `std::string` unlike 
regular tasks which use namespaces and enumerators.

<u>Examples:</u>

=== "Numerical indexes"

    ```cpp
    Switcher swi(2, 6, typeid(uint8_t)); // n_data_sockets, n_elemts, datatype

    swi[module::swi::tsk::select ][0]; // input  socket data0
    swi[module::swi::tsk::select ][1]; // input  socket data1
    swi[module::swi::tsk::select ][2]; // output socket data
    swi[module::swi::tsk::select ][3]; // output socket status

    swi[module::swi::tsk::commute][0]; // input  socket data
    swi[module::swi::tsk::commute][1]; // input  socket ctrl
    swi[module::swi::tsk::commute][2]; // output socket data0
    swi[module::swi::tsk::commute][3]; // output socket data1
    swi[module::swi::tsk::commute][4]; // output socket status
    ```

=== "`std::string`"

    ```cpp
    Switcher swi(2, 6, typeid(uint8_t)); // n_data_sockets, n_elemts, datatype

    swi[ "select::in_data0" ];         // input  socket data0
    swi[ "select::in_data1" ];         // input  socket data1
    swi[ "select::out_data" ];         // output socket data
    swi[ "select::status"   ];         // output socket status 

    swi["commute::in_data"  ];         // input  socket data
    swi["commute::in_ctrl"  ];         // input  socket ctrl
    swi["commute::out_data0"];         // output socket data0
    swi["commute::out_data1"];         // output socket data1
    swi["commute::status"   ];         // output socket status
    ```

<a name="Commute"></a>
### Commute

The **`commute` task** is used to **create exclusive paths**. When the `commute` 
task is executed it reads the [`path`](#Path) to take from its `in_ctrl` socket
(an `int8_t`), sets the path attribute of the module to the value read (modulo
`n_data_sockets`), and copies the bytes from its `in_data` input socket to the
`out_data{path}` output socket (unless the no-copy mode is enabled). The
codelet returns the path: it is written in the `status` socket and read by the
sequence to select the sub-sequence to execute next.

Any task bound to its output sockets **before `status`** will be considered in a
different *exclusive path*.
 
<u>Sockets</u>

| Type        | Name            | Index    |
| ----------- | --------------- | -------- |
| `Input`     | in_data         | 0        |
| `Input`     | in_ctrl         | 1        |
| `Output`    | out_data{0..N-1}| {2..N+1} |
| `Output`    | status          | N+2      |

<a name="Select"></a>
### Select

The `select` task is used to join exclusive paths.

When a `select` task is executed it gets the [`path`](#Path) from the module and
copies the bytes from its `in_data{path}` input to its `out_data` output socket
(unless the no-copy mode is enabled). Any task bound to its input sockets will
be considered in a different *exclusive path*.

!!! danger
    The `select` task does not know from which path it has been reached: it
    always uses the [`path`](#Path) stored in the module. If this value does not
    match the path actually taken, the data are read from the wrong input
    socket. That is most likely to happen in a [loop](#Examples) because the
    `select` is executed before the `commute`. Then the path defaults to
    `n_data_sockets - 1` meaning that in a loop, **the first path taken before
    the first `select` execution should always be bound to the last input
    socket**.

<u>Sockets</u>

| Type        | Name            | Index    |
| ----------- | --------------- | -------- |
| `Input`     | in_data{0..N-1} | {0..N-1} |
| `Output`    | out_data        | N        |
| `Output`    | status          | N+1      |

<a name="Examples"></a>
## Examples

=== "While loop"
    <figure markdown>
      ![while loop](./assets/while_loop.svg){ width=1000 }
      <figcaption>Example of a sequence with a while-loop.</figcaption>
    </figure>

=== "Switch-case"
    <figure markdown>
      ![switch case](./assets/switch_case.svg){ width=1000 }
      <figcaption>Example of a sequence with a switch-case.</figcaption>
    </figure>