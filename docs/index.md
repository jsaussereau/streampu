# Introduction

`StreamPU` is a **Domain Specific Embedded Language** (DSEL) for **streaming
applications**, written in C++11. It comes in the form of a C++ library to
link with.

<figure markdown>
  ![Sequence and pipeline](./assets/sequence_to_pipeline.svg){ width=1200 }
  <figcaption>A sequence of tasks (left) and the same tasks in a 3-stage pipeline (right).</figcaption>
</figure>

Here are the main features of `StreamPU`:

- Definition of dataflow components: **modules**, **tasks** and **sockets**;
- Elementary modules and tasks implementations;
- **Multi-threaded runtime** with **replication** and **pipeline** parallel
  constructs;
- **Control flow** (loops, conditional paths) inside the dataflow graphs;
- Automatic pipeline generation (**schedulers**), **thread pinning**, probes
  and statistics;
- Tasks written in [Julia](julia.md).

The DSEL is suitable for SDR systems, audio/video processing and more generally
it matches **single-rate SDF streaming applications**.

!!! note
    The **DSEL term** can be sometimes confusing and `StreamPU` can also be
    seen as a **standard C++ library**. The name "DSEL" comes from the ability 
    to write **interpreted Turing-complete programs** using the C++ library.

!!! warning
    This library is **NOT intended to address data and task parallelisms**.
    `StreamPU` focuses on replication and pipeline parallelisms. For data 
    parallelism, `StreamPU` combines well with OpenMP. If you look for task
    parallelism, using OpenMP can also be a possible solution, or other runtime 
    like the excellent [StarPU](https://starpu.gitlabpages.inria.fr) can be a 
    good choice.

## How to Read this Documentation

The documentation is split in two parts:

- **For Users**: how to install `StreamPU` and how to write applications with
  it. Start with [Installation](installation.md),
  [Concepts Overview](concepts_overview.md) and
  [Getting Started](getting_started.md).
- **For Developers**: how `StreamPU` is built internally (classes, attributes,
  algorithms). This part is useful to modify `StreamPU` and to contribute.

The [slides](https://largo.lip6.fr/~cassagnea/docs/spu/StreamPU_slides_anim.pdf)
([version without animation](https://largo.lip6.fr/~cassagnea/docs/spu/StreamPU_slides_no_anim.pdf))
used to teach `StreamPU` to Master students at Sorbonne University contain many
illustrative and simple use cases.

## A Minimal Example

```cpp
#include <atomic>
#include <streampu.hpp>
using namespace spu;

int main()
{
    module::Initializer<int> initializer(16); // produces frames of 16 integers
    module::Incrementer<int> incrementer(16); // adds 1 to its input
    module::Finalizer<int>   finalizer(16);   // memorizes the last frame

    incrementer["increment::in"] = initializer["initialize::out"];
    finalizer["finalize::in"] = incrementer["increment::out"];

    runtime::Sequence seq(initializer("initialize"), 4); // 4 threads

    std::atomic<size_t> counter(0);
    seq.exec([&counter]() { return ++counter >= 1000; });
    return 0;
}
```

See [Getting Started](getting_started.md) for a detailed explanation.

## License and Citation

`StreamPU` is licensed under the MIT license. The main contributions of this
work are described in the following journal article:

- A. Cassagne, R. Tajan, O. Aumage, D. Barthou, C. Leroux and C. Jégo,
  "**StreamPU: A DSEL for High Throughput and Low Latency Software-Defined
  Radio on Multicore CPUs**," *Wiley Concurrency and Computation: Practice and
  Experience (CCPE)*, 2023.
  [[Open access article](https://doi.org/10.1002/cpe.7820)]
