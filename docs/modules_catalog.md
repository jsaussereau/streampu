# Built-in Modules

This page lists the modules provided by `StreamPU` (namespace `spu::module`).
For each module, the tasks and their sockets are given in creation order. The
number of elements of a socket is given **per frame**. Every task also has a
last output socket named `status` (not listed below, see
[The `status` Socket](concepts_overview.md#the-status-socket)).

The sockets can be accessed with strings (`rly["relay::in"]`) or with the
enumerations given for each module (`rly[module::rly::sck::relay::in]`), and
the tasks with `rly("relay")` or `rly[module::rly::tsk::relay]`.

Unless stated otherwise, the templated modules are explicitly instantiated for
`int8_t`, `uint8_t`, `int16_t`, `uint16_t`, `int32_t`, `uint32_t`, `int64_t`,
`uint64_t`, `float` and `double`.

## Summary

| Module | Main tasks | Replicable tasks |
| ------ | ---------- | ---------------- |
| [`Initializer<T>`](#initializer) | `initialize` | yes |
| [`Finalizer<T>`](#finalizer) | `finalize` | yes |
| [`Incrementer<T>`](#incrementer) | `increment`, `incrementf` | yes |
| [`Relayer<T>`](#relayer) | `relay`, `relayf` | yes |
| [`Sleeper`](#sleeper) | `sleep` | yes |
| [`Delayer<D>`](#delayer) | `memorize`, `produce` | yes |
| [`Unaryop<TI,TO,UOP>`](#unaryop) | `perform` | yes |
| [`Binaryop<TI,TO,BOP>`](#binaryop) | `perform`, `performf` | yes |
| [`Reducer<TI,TO,BOP>`](#reducer) | `reduce` | yes |
| [`Source_AZCW<B>`](#sources), [`Source_random<B>`](#sources) | `generate` | yes |
| [`Source_user<B>`](#sources), [`Source_user_binary<B>`](#sources) | `generate` | **no** |
| [`Sink_NO<B>`](#sinks) | `send`, `send_count` | yes |
| [`Sink_user_binary<B>`](#sinks) | `send`, `send_count` | **no** |
| [`Switcher`](control_flow.md) | `commute`, `select` | yes |
| [`Iterator`](#iterator) | `iterate`, `reset` | yes |
| [`Controller_static`, `Controller_cyclic`, `Controller_limit`](#controllers) | `control`, `reset` | `control` only |
| [`Set`](#set) | `exec` | yes |
| [Probes](probes.md) | `probe` | **no** |
| [`Stateless`](custom_modules.md#stateless-modules) | user defined | yes |
| [`Stateless_Julia`, `Stateful_Julia`](julia.md) | user defined | see [Julia](julia.md) |
| `Adaptor_m_to_n` | `push`, `pull` | internal (pipelines) |

## Data Generation and Validation

### Initializer

```cpp
template <typename T = int> class Initializer;
Initializer(const size_t n_elmts, const size_t ns = 0);
```

| Task | Sockets | Behavior |
| ---- | ------- | -------- |
| `initialize` | `out` (`T`, `n_elmts`) | Copies the initialization data of the current frame into `out`. |

Enumerations: `module::ini::tsk::initialize`, `module::ini::sck::initialize::{out, status}`.

The initialization data are zeros by default. They are set with:

- `set_init_data(const T val)`: all the elements of all the frames;
- `set_init_data(const std::vector<T>& data)`: the same frame for all the frames
  (`data.size() == n_elmts`);
- `set_init_data(const std::vector<std::vector<T>>& data)`: one vector per frame
  (`data.size() == n_frames`).

`get_init_data()` returns the data. If `ns > 0`, the task actively waits until
`ns` nanoseconds have elapsed since its start (`set_ns()`/`get_ns()`).

### Finalizer

```cpp
template <typename T = int> class Finalizer;
Finalizer(const size_t n_elmts, const size_t history_size = 1, const size_t ns = 0);
```

| Task | Sockets | Behavior |
| ---- | ------- | -------- |
| `finalize` | `in` (`T`, `n_elmts`) | Copies `in` into an internal history of `history_size` streams. |

Enumerations: `module::fin::tsk::finalize`, `module::fin::sck::finalize::{in, status}`.

`get_final_data()` returns the last received stream (a vector of `n_frames`
vectors of `n_elmts` elements), `get_histo_data()` the whole history.
`reset()` resets the history position. `ns`: same as `Initializer`.

### Incrementer

```cpp
template <typename T = int> class Incrementer;
Incrementer(const size_t n_elmts, const size_t ns = 0);
```

| Task | Sockets | Behavior |
| ---- | ------- | -------- |
| `increment` | `in` (`T`, `n_elmts`), `out` (`T`, `n_elmts`) | `out[i] = in[i] + 1` |
| `incrementf` | `fwd` (`T`, `n_elmts`) | `fwd[i] = fwd[i] + 1` (in place) |

Enumerations: `module::inc::tsk::{increment, incrementf}`,
`module::inc::sck::increment::{in, out, status}`,
`module::inc::sck::incrementf::{fwd, status}`. `ns`: same as `Initializer`.

### Relayer

```cpp
template <typename T = int> class Relayer;
Relayer(const size_t n_elmts, const size_t ns = 0);
```

| Task | Sockets | Behavior |
| ---- | ------- | -------- |
| `relay` | `in` (`T`, `n_elmts`), `out` (`T`, `n_elmts`) | Copies `in` into `out`. |
| `relayf` | `fwd` (`T`, `n_elmts`) | Does nothing on the data (forward socket). |

Enumerations: `module::rly::tsk::{relay, relayf}`,
`module::rly::sck::relay::{in, out, status}`,
`module::rly::sck::relayf::{fwd, status}`. `ns`: same as `Initializer`.

### Sleeper

```cpp
Sleeper(const size_t ns = 0);
```

| Task | Sockets | Behavior |
| ---- | ------- | -------- |
| `sleep` | none | Actively waits `ns` nanoseconds (busy loop, no system call). |

Enumerations: `module::slp::tsk::sleep`, `module::slp::sck::sleep::status`.

### Delayer

```cpp
template <typename D = int> class Delayer;
Delayer(const size_t size, const D init_val);
```

| Task | Sockets | Behavior |
| ---- | ------- | -------- |
| `memorize` | `in` (`D`, `size`) | Stores `in` in an internal buffer. |
| `produce` | `out` (`D`, `size`) | Copies the internal buffer into `out`. |

Enumerations: `module::dly::tsk::{memorize, produce}`,
`module::dly::sck::memorize::{in, status}`, `module::dly::sck::produce::{out, status}`.

When `produce` is executed before `memorize` in a sequence, the data are
delayed by one stream. The buffer is initialized with `init_val`
(`set_data()` sets other initial values).

## Arithmetic

### Unaryop

```cpp
template <typename TI, typename TO, tools::proto_uop<TI, TO> UOP> class Unaryop;
Unaryop(const size_t n_elmts);
```

| Task | Sockets | Behavior |
| ---- | ------- | -------- |
| `perform` | `in` (`TI`, `n_elmts`), `out` (`TO`, `n_elmts`) | `out[i] = UOP(in[i])` |

Enumerations: `module::uop::tsk::perform`, `module::uop::sck::perform::{in, out, status}`.

Aliases (`template <typename TI, typename TO = TI>`), with the corresponding
operator defined in `Tools/Math/unaryop.h`:

| Alias | Operation |
| ----- | --------- |
| `Unaryop_abs` | `std::abs(a)` |
| `Unaryop_cast` | `(TO)a` |
| `Unaryop_neg` | `-a` |
| `Unaryop_not` | `~a` |
| `Unaryop_not_abs` | `~std::abs(a)` |
| `Unaryop_sign` | `std::signbit(a)` (`1` if negative, `0` otherwise) |

```cpp
module::Unaryop_abs<float> abs_op(1024); // task "perform"
```

These modules are header-only templates (no explicit instantiation). The name
of the module is `"Unaryop<op>"`; the operator name is detected at compile time
and some operators can only be detected with C++17 (`if constexpr`),
otherwise `"ukn"` is displayed. The same applies to the `Binaryop` and
`Reducer` names.

### Binaryop

```cpp
template <typename TI, typename TO, tools::proto_bop<TI, TO> BOP> class Binaryop;
Binaryop(const size_t n_in0, const size_t n_in1);
Binaryop(const size_t n_elmts);
```

| Task | Sockets | Behavior |
| ---- | ------- | -------- |
| `perform` | `in0` (`TI`, `n_in0`), `in1` (`TI`, `n_in1`), `out` (`TO`, `max(n_in0, n_in1)`) | `out[i] = BOP(in0[i], in1[i])` |
| `performf` | `in` (`TI`, `n_in0`), `fwd` (`TI`, `n_in1`) | `fwd[i] = BOP(fwd[i], in[i])` (in place) |

Enumerations: `module::bop::tsk::{perform, performf}`,
`module::bop::sck::perform::{in0, in1, out, status}` and
`module::bop::sck::performf::{in0, in1, status}` (for `performf`, `in0` is the
`in` socket and `in1` is the `fwd` socket).

`n_in0` and `n_in1` must be equal, or one of them must be 1: in this case the
single value is applied to all the elements (scalar broadcast).

!!! note
    In the `performf` task, the forward socket is the **left** operand:
    `fwd = fwd OP in`. This matters for non-commutative operators.

Aliases (`template <typename TI, typename TO = TI>`):
`Binaryop_add` (`a + b`), `Binaryop_sub` (`a - b`), `Binaryop_mul` (`a * b`),
`Binaryop_div` (`a / b`), `Binaryop_or` (`a | b`), `Binaryop_xor` (`a ^ b`),
`Binaryop_and` (`a & b`), `Binaryop_min`, `Binaryop_max`, `Binaryop_gt`
(`a > b`), `Binaryop_ge` (`a >= b`), `Binaryop_lt` (`a < b`), `Binaryop_le`
(`a <= b`), `Binaryop_ne` (`a != b`), `Binaryop_eq` (`a == b`). The result is
converted to `TO`.

```cpp
module::Binaryop_add<int32_t> adder(256);     // task "perform", module name "Binaryop<add>"
module::Binaryop_gt<float, int8_t> cmp(1, 1); // int8_t output, usable as a Switcher control
```

### Reducer

```cpp
template <typename TI, typename TO, tools::proto_bop<TI, TO> BOP> class Reducer;
Reducer(const size_t n_elmts);
```

| Task | Sockets | Behavior |
| ---- | ------- | -------- |
| `reduce` | `in` (`TI`, `n_elmts`), `out` (`TO`, 1) | Reduces all the elements with `BOP`. |

Enumerations: `module::red::tsk::reduce`, `module::red::sck::reduce::{in, out, status}`.

`Reducer` is a single-wave module: the reduction is performed over all the
elements of **all the frames**, and the result is written in the first
element of `out`.

Aliases (`template <typename TI, typename TO = TI, tools::proto_bop<TI, TO> BOP = ...>`,
used as `Reducer_add<int>` or `Reducer_add<int, float>`): `Reducer_add`,
`Reducer_sub`, `Reducer_mul`, `Reducer_div`, `Reducer_or`, `Reducer_xor`,
`Reducer_and`, `Reducer_min`, `Reducer_max`.

## Sources

```cpp
template <typename B = int> class Source; // abstract
```

| Task | Sockets | Behavior |
| ---- | ------- | -------- |
| `generate` | `out_data` (`B`, `max_data_size`), `out_count` (`uint32_t`, 1) | Produces a frame and the number of valid elements in `out_count`. |

Enumerations: `module::src::tsk::generate`,
`module::src::sck::generate::{out_data, out_count, status}`.

The sources implement `tools::Interface_is_done` (used by the
[auto-stop](sequence_usage.md#auto-stop) mode), `tools::Interface_reset` and
`tools::Interface_set_seed`.

| Module | Constructor | Behavior | Replicable |
| ------ | ----------- | -------- | ---------- |
| `Source_AZCW<B>` | `(int max_data_size)` | Produces zeros (All Zero Code Word). | yes |
| `Source_random<B>` | `(int max_data_size, int seed = 0)` | Produces random 0 and 1 (Mersenne Twister). `set_seed()` changes the seed. | yes |
| `Source_user<B>` | `(int max_data_size, const std::string& filename, bool auto_reset, int start_idx = 0)` or `(int max_data_size, const std::string& filename, int start_idx = 0)` | Reads frames from a text file (see below). Without `auto_reset`, `is_done()` becomes `true` after the last frame. | no |
| `Source_user_binary<B>` | `(int max_data_size, const std::string& filename, bool auto_reset = true, bool fifo_mode = false)` | Reads a binary file **bit by bit**: each bit of the file becomes one element (0 or 1). `out_count` contains the number of valid bits; the end of the last frame is padded with zeros. At the end of the file, the file is read again if `auto_reset` is `true`, otherwise `is_done()` becomes `true`. | no |

The text format of `Source_user` is: the number of frames, the frame size
(which must be equal to `max_data_size`), then all the values.

## Sinks

```cpp
template <typename B = int> class Sink; // abstract
```

| Task | Sockets | Behavior |
| ---- | ------- | -------- |
| `send` | `in_data` (`B`, `max_data_size`) | Sends all the elements. |
| `send_count` | `in_data` (`B`, `max_data_size`), `in_count` (`uint32_t`, 1) | Sends the first `in_count` elements. |

Enumerations: `module::snk::tsk::{send, send_count}`,
`module::snk::sck::send::{in_data, status}`,
`module::snk::sck::send_count::{in_data, in_count, status}`.

| Module | Constructor | Behavior | Replicable |
| ------ | ----------- | -------- | ---------- |
| `Sink_NO<B>` | `(int max_data_size)` | Does nothing. | yes |
| `Sink_user_binary<B>` | `(int max_data_size, const std::string& filename)` | Writes the elements in a binary file, **one element = one bit** (the elements must be 0 or 1). `reset()` truncates the file. | no |

`Source_user_binary` and `Sink_user_binary` are typically used together, the
`out_count` socket of the source being bound to the `in_count` socket of the
sink (see the [Simple Pipeline](tests_bootstrap.md#simple-pipeline) test).

## Control Flow

The `Switcher` module is described in [Control Flow](control_flow.md).

### Iterator

```cpp
Iterator(const size_t limit);
```

| Task | Sockets | Behavior |
| ---- | ------- | -------- |
| `iterate` | `out` (`int8_t`, 1) | Writes `0` during `limit` calls, then `1` once, and starts again. |
| `reset` | none | Resets the counter. |

Enumerations: `module::ite::tsk::{iterate, reset}`, `module::ite::sck::iterate::{out, status}`.

Methods: `set_limit()`, `get_limit()`, `reset()`. Single-wave module.

### Controllers

`Controller` is an abstract module (`Controller(const size_t init_path = 0)`):

| Task | Sockets | Behavior |
| ---- | ------- | -------- |
| `control` | `out` (`int8_t`, 1) | Writes a path index. |
| `reset` | none | Calls `reset()` (the path goes back to `init_path`). |

Enumerations: `module::ctr::tsk::{control, reset}`, `module::ctr::sck::control::{out, status}`.

| Module | Constructor | Produced values |
| ------ | ----------- | --------------- |
| `Controller_static` | `(const size_t init_path = 0)` | Always the current path (`set_path()` changes it). |
| `Controller_cyclic` | `(const size_t n_paths, const size_t init_path = 0)` | `init_path`, then the next path at each call, modulo `n_paths`. |
| `Controller_limit` | `(const size_t limit, const size_t init_path = 0)` | `init_path` during `limit` calls, then `!init_path` (0 or 1). |

The controllers are single-wave modules. Only their `control` task is
replicable.

## Set

```cpp
explicit Set(runtime::Sequence& sequence);       // uses the given sequence
explicit Set(const runtime::Sequence& sequence); // uses a clone of the sequence
```

A `Set` encapsulates a [sequence](sequence_usage.md) into a **single task**
named `exec`, which executes the whole sequence once. The sockets of the
`exec` task are created from the sequence:

1. one input socket per input socket of the first tasks of the sequence (same
   name, type and size);
2. one output socket per output socket of the last tasks of the sequence (the
   `status` sockets excepted).

The sequence must be executed by one thread. Forward sockets are not
supported on the first and last tasks. The sockets have no names enumeration:
use their index (`aset("exec")[0]`) or their name.

```cpp
// incs[0] -> ... -> incs[5] become a single task
runtime::Sequence partial_sequence((*incs[0])("increment"), (*incs[5])("increment"));
module::Set aset(partial_sequence);
aset("exec")[0] = initializer["initialize::out"]; // input of incs[0]
finalizer["finalize::in"] = aset("exec")[1];      // output of incs[5]
```

`get_sequence()` returns the encapsulated sequence. `Sequence::get_modules()`
also returns the modules contained in the sets (when `set_modules` is
`true`).

## Probes

The probes (`Probe_latency`, `Probe_occurrence`, `Probe_stream`,
`Probe_throughput`, `Probe_time`, `Probe_timestamp`, `Probe_value<T>`) are
described in [Probes & Reporters](probes.md).

## Adaptor

`Adaptor_m_to_n` implements the synchronization between pipeline stages. It is
created and bound automatically by the [pipeline](pipeline_usage.md) and is
not meant to be used directly. It is described in the developer page
[Pipeline](pipeline.md#adaptor).
