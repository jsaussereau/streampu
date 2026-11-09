# Installation

`StreamPU` is a C++11 library built with CMake. This page explains how to get
the sources, compile and install the library, and link it into your own
project.

## Requirements

- A C++11 compiler.
- CMake 3.5 or newer.
- A threading library (`pthread` on Unix-like systems), found automatically
  through CMake's `find_package(Threads)`.
- Optional dependencies:
    * [`hwloc`](https://www.open-mpi.org/projects/hwloc) for
      [thread pinning](thread_pinning.md) (`SPU_LINK_HWLOC=ON`);
    * [Julia](https://julialang.org/) 1.10.9 or newer for
      [Julia modules](julia.md) (`SPU_LINK_JULIA=ON`);
    * Valgrind (Helgrind headers) for `SPU_HELGRIND=ON`.

`StreamPU` relies on three Git submodules located in the `lib/` folder:

| Submodule      | Usage                                                            | Required                                           |
| -------------- | ---------------------------------------------------------------- | -------------------------------------------------- |
| `lib/rang`     | Colors in the terminal (header-only)                             | Always                                             |
| `lib/json`     | JSON parsing (`nlohmann/json`, header-only), used by the `FILE` [scheduler](schedulers.md) | Always                   |
| `lib/cpptrace` | Stack traces in exceptions                                       | Only if `SPU_STACKTRACE` or `SPU_STACKTRACE_SEGFAULT` is `ON` |

Clone the repository with its submodules:

```bash
git clone --recursive https://github.com/aff3ct/streampu.git
```

Or, if the repository is already cloned:

```bash
git submodule update --init --recursive
```

!!! note
    If a required submodule is missing, CMake stops with an error message that
    gives the exact `git submodule update --init -- ...` command to run.

## Compilation

### CMake Options

| Option                    | Default | Description                                                                                                       |
| ------------------------- | ------- | ----------------------------------------------------------------------------------------------------------------- |
| `SPU_COMPILE_STATIC_LIB`  | `ON`    | Compile the static library (target `spu-static-lib`).                                                             |
| `SPU_COMPILE_SHARED_LIB`  | `OFF`   | Compile the shared library (target `spu-shared-lib`).                                                             |
| `SPU_LINK_HWLOC`          | `OFF`   | Link with `hwloc` and define `SPU_HWLOC` (required for [thread pinning](thread_pinning.md)).                      |
| `SPU_COLORS`              | `ON`    | Define `SPU_COLORS`: enable colors in the terminal outputs.                                                       |
| `SPU_BENCHMARK`           | `ON`    | Compile the [`spu-benchmark`](benchmark.md) executable (forced to `ON` when `SPU_TESTS=ON`).                      |
| `SPU_TESTS`               | `ON`    | Compile the [tests](tests_bootstrap.md) and register them in CTest.                                               |
| `SPU_STACKTRACE`          | `ON`    | Define `SPU_STACKTRACE`: print the stack trace when a `StreamPU` exception is raised (links with `cpptrace`).      |
| `SPU_STACKTRACE_SEGFAULT` | `OFF`   | Define `SPU_STACKTRACE_SEGFAULT`: try to print the stack trace when a segfault occurs (links with `cpptrace`).    |
| `SPU_SHOW_DEPRECATED`     | `OFF`   | Define `SPU_SHOW_DEPRECATED`: print a message each time a deprecated function is called.                          |
| `SPU_FAST`                | `OFF`   | Define `SPU_FAST`: remove most runtime checks to speed up the code (see [Debugging](debugging.md#fast-mode)).     |
| `SPU_HELGRIND`            | `OFF`   | Annotate the code for the Helgrind thread error detector (defines `HAS_HELGRIND` if Helgrind is found).           |
| `SPU_OVERRIDE_VERSION`    | `OFF`   | Compile without a `.git` folder. Set it to a version string (e.g. `-DSPU_OVERRIDE_VERSION=v1.0.0`) instead of `ON`. |
| `SPU_LINK_JULIA`          | `OFF`   | Link with Julia and define `SPU_JULIA` (see [Julia Interoperability](julia.md)). Requires Julia ≥ 1.10.9.         |

!!! info
    The version of the library is normally computed from Git
    (`git describe --tags --abbrev=7`). When the sources are not in a Git
    repository, CMake prints a warning and falls back to `v0.0.0`, unless
    `SPU_OVERRIDE_VERSION` is given.

### Debug Build

```bash
cmake -S . -B build_debug
cmake --build build_debug -j 4
ctest --test-dir build_debug
```

### Release Build

```bash
cmake -S . -B build_release -DCMAKE_BUILD_TYPE=RelWithDebInfo -DCMAKE_CXX_FLAGS="-Wall -march=native -funroll-loops"
cmake --build build_release -j 4
ctest --test-dir build_release
```

The executables (`spu-benchmark`, `test-*`) are generated in the `bin/`
sub-folder of the build directory and the libraries in its `lib/` sub-folder.

### Running the Tests

When `SPU_TESTS=ON`, the generic test executables (simple chains, loops,
switch-case and, with `SPU_LINK_JULIA=ON`, the Julia chains) are registered several times in CTest
with different command line parameters. Each registration carries a CTest
label:

| Label          | Command line                         |
| -------------- | ------------------------------------ |
| `run-1t`       | `-d 2048 -s 5 -e 10 -t 1`            |
| `run`          | `-d 2048 -s 5 -e 100`                |
| `inter-frames` | `-d 2048 -s 5 -e 10 -f 13`           |
| `step-by-step` | `-d 2048 -s 5 -e 11 -f 13 -b`        |
| `copy-mode`    | `-d 2048 -s 5 -e 15 -f 13 -c`        |
| `graph-dot`    | `-d 2048 -s 5 -e 15 -f 13 -o graph.dot` |

Some tests are also registered with specific parameters under dedicated labels
(`set`, `exclusive-paths`, `simple-pipeline`, `pipeline-probe`, ...); the
pipeline tests are only registered this way. See the
end of the root `CMakeLists.txt` file for the complete list.

It is possible to run only one category, for instance:

```bash
ctest --test-dir build_release -L run-1t
```

See the [Tests](tests_bootstrap.md) page for a description of each test.

## Installation

Install the library on the system (in `/usr/local` by default):

```bash
cmake --install build_release
```

Install the library in a custom path (here `/opt/streampu`):

```bash
cmake --install build_release --prefix /opt/streampu/
```

The installation copies the headers (in `include/streampu/`), the compiled
libraries, the `rang` and `json` headers (in `include/rang/` and
`include/json/`), the CMake package configuration files (in
`lib/cmake/streampu/`) and, when `SPU_BENCHMARK=ON`, the `spu-benchmark`
executable (in `bin/`).

## Link with the Library

`StreamPU` exposes two CMake targets:

- `spu::spu-static-lib`: static library, includes and definitions (compiled by
  default);
- `spu::spu-shared-lib`: shared library, includes and definitions (requires
  `SPU_COMPILE_SHARED_LIB=ON`).

The preprocessor definitions selected at compile time (`SPU_COLORS`,
`SPU_FAST`, `SPU_HWLOC`, `SPU_STACKTRACE`, ...) are `PUBLIC`: they are
propagated to the projects that link with these targets.

In your source files, include the main header, which includes all the other
`StreamPU` headers:

```cpp
#include <streampu.hpp>
using namespace spu;
```

### From a Sub-directory

Suppose that the `my-exe` executable has to be linked with `StreamPU` and that
the `StreamPU` repository is located in `my-exe-root/lib/streampu/`. Here is a
minimal `CMakeLists.txt`:

```cmake
cmake_minimum_required(VERSION 3.5)

project(my_project CXX)

# require C++11 compiler ------------------------------------------------------
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# create the `my-exe` executable from `src/main.cpp` --------------------------
add_executable(my-exe ${CMAKE_CURRENT_SOURCE_DIR}/src/main.cpp)

# compile `StreamPU` static lib -----------------------------------------------
option(SPU_TESTS "" OFF) # do NOT compile the tests
add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/lib/streampu/)

# `my-exe` links with `StreamPU` static lib -----------------------------------
target_link_libraries(my-exe PUBLIC spu-static-lib)

# `my-exe` finds and links with thread library --------------------------------
set(CMAKE_THREAD_PREFER_PTHREAD ON)
set(THREADS_PREFER_PTHREAD_FLAG ON)
find_package(Threads REQUIRED)
target_link_libraries(my-exe PUBLIC Threads::Threads)

# `my-exe` links with `cpptrace` (`SPU_STACKTRACE=ON`) ------------------------
if(SPU_STACKTRACE OR SPU_STACKTRACE_SEGFAULT)
  target_link_libraries(my-exe PUBLIC cpptrace::cpptrace)
endif()
```

### From an Installed Library

Suppose that `my-exe` has to be linked with the `StreamPU` static library
installed on the system. Here is a minimal `CMakeLists.txt`:

```cmake
cmake_minimum_required(VERSION 3.5)

project(my_project CXX)

# require C++11 compiler ------------------------------------------------------
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# create the `my-exe` executable from `src/main.cpp` --------------------------
add_executable(my-exe ${CMAKE_CURRENT_SOURCE_DIR}/src/main.cpp)

# find and link with thread library -------------------------------------------
set(CMAKE_THREAD_PREFER_PTHREAD ON)
set(THREADS_PREFER_PTHREAD_FLAG ON)
find_package(Threads REQUIRED)
target_link_libraries(my-exe PUBLIC Threads::Threads)

# find and link with `cpptrace` if installed (if not, don't) ------------------
find_package(cpptrace CONFIG 0.4.1)
if (cpptrace_FOUND)
  target_link_libraries(my-exe PUBLIC cpptrace::cpptrace)
endif()

# find and link with `StreamPU` -----------------------------------------------
find_package(StreamPU CONFIG 0.0.0 REQUIRED)
target_link_libraries(my-exe PUBLIC spu::spu-static-lib)
```

If `StreamPU` is installed in a specific location (for instance
`/opt/streampu`), help CMake to find it:

```bash
cmake .. -DStreamPU_DIR=/opt/streampu/lib/cmake/streampu
```

Or, if `StreamPU` has been compiled with the `cpptrace` library:

```bash
cmake .. -DStreamPU_DIR=/opt/streampu/lib/cmake/streampu -Dcpptrace_DIR=/opt/streampu/lib/cmake/cpptrace
```

## Building this Documentation

This documentation is written in Markdown and generated with
[MkDocs](https://www.mkdocs.org/) and the
[Material for MkDocs](https://squidfunk.github.io/mkdocs-material/) theme. The
Python requirements are listed in `docs/requirements.txt`:

```bash
pip install -r docs/requirements.txt
mkdocs serve   # live preview on http://127.0.0.1:8000
mkdocs build   # static website in the `site/` folder
```
