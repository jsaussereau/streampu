# Thread Pinning

`StreamPU` enables to select on which process units (PUs) the threads are 
effectively run. This is called *thread pinning* and it can significantly 
benefit to the performance, especially on modern heterogeneous architectures. 
To do so, the runtime relies on the 
[`hwloc`](https://www.open-mpi.org/projects/hwloc) library.

!!! warning
    To use thread pinning, `hwloc` library has to be installed on the system and
    `StreamPU` needs to be compiled with the `SPU_HWLOC` preprocessor 
    definition. It can simply be achieved using the following CMake option:
    ```bash
    cmake .. -DSPU_LINK_HWLOC=ON
    ``` 
    If `StreamPU` is not linked with the `hwloc` library, then the thread 
    pinning interface will have no effect and the threads will not be pinned.

!!! important
    The `hwloc` topology has to be loaded **before** creating the sequences or
    pipelines that use thread pinning, and released at the end of the
    application:
    ```cpp
    tools::Thread_pinning::init();    // load the hwloc topology
    // ... create and execute the sequences/pipelines ...
    tools::Thread_pinning::destroy(); // release the hwloc topology
    ```
    If `init()` has not been called, the pinning functions do nothing.

!!! info
	Thread pinning relies the OS. The later needs to expose appropriated system 
	calls. While Linux and Windows provide these syscalls,	macOS does not... 
	Thus, **thread pinning will have no effect on macOS :-(**.

## Portable Hardware Locality

*Portable Hardware Locality* (`hwloc` in short) is a library which provides a 
**portable abstraction** of the **hierarchical topology of modern 
architectures** (see the figure below).

<figure markdown>
  ![Orange Pi 5](./assets/hwloc_orangepi5.svg)
  <figcaption>
  	Result of the hwloc-ls command on the Orange Pi 5 Plus board (Rockchip 
  	RK3588 SoC, cores 0-3 are energy efficient ARM Cortex-A55 and cores 4-7 are 
  	powerful ARM Cortex-A76).
  </figcaption>
</figure>

`hwloc` gives the ability to pin threads over various level of hierarchy 
represented by a tree structure. The deepest/lowest nodes (the leaves) are the 
PUs while higher nodes represent sets of PUs that are physically close. For 
instance, a PUs set can share the same UMA node (in the case of a NUMA 
architecture), the same LLC or the same package. 

In the Orange Pi 5 SBC, if we pin a thread on the `Package L#0`, it will run 
over the following set of PUs: `PU L#0`, `PU L#1`, `PU L#2` and `PU L#3`. 
Thus, **the pinned thread can move in the selected `hwloc` node during the 
execution** and it is up to the OS to schedule the thread on the selected PUs 
set.

!!! warning
	The indexes given by `hwloc` can be different from those given by the OS: 
	they are logical indexes that express the real locality. **Consequently, in 
	`StreamPU`, it is important to use `hwloc` logical indexes.** The 
	`hwloc-ls` command gives an overview of the current topology with these 
	logical indexes.

## Sequence & Pipeline

In `StreamPU`, thread pinning can be set in `runtime::Sequence` and 
`runtime::Pipeline` class constructors. In both cases, there is a dedicated 
argument of `std::string` type named `sequence_pinning_policy` for 
`runtime::Sequence` or `pipeline_pinning_policy` for `runtime::Pipeline`. The
pinning also has to be enabled with the `thread_pinning` boolean argument (one
boolean per stage for a pipeline):

```cpp
runtime::Sequence seq(first_task, 4 /* n_threads */, true /* thread_pinning */, "PU_0; PU_1; PU_2; PU_3");

runtime::Pipeline pipeline(firsts, lasts, sep_stages,
                           { 1, 4, 1 },           // number of threads per stage
                           { 16, 16 },            // synchronization buffer sizes
                           { false, false },      // active waiting
                           { true, true, true },  // thread pinning per stage
                           "PU_0 | PACKAGE_1; PACKAGE_1; PACKAGE_2; PACKAGE_2 | PACKAGE_0");
```

The policy can also be changed after the construction with
`Sequence::set_thread_pinning(const bool, const std::string&)`.

!!! note
    An older interface is also available: instead of a policy string, a vector
    of PU logical indexes (`puids`, one per thread) is given to the
    constructor. Each thread is then pinned on a single PU
    (`tools::Thread_pinning::pin(puid)`).

!!! info
    For NUMA architectures, it is important to specify thread pinning at the 
    construction of the `runtime::Sequence`/`runtime::Pipeline` object to 
    guarantee that the data will be allocated and initialized on the right 
    memory banks (according to the first touch policy) during the replication 
    process.

To specify the pinning policy, we defined a syntax to express `hwloc` objects 
with three different separators:  

- Pipeline stage (does not concern `runtime::Sequence`): `|`
- Replicated stage (= replicated sequence = one thread): `;`  
- For one thread, the list of pinned `hwloc` objects (= logical or): `,`  

The spaces are ignored. If a pipeline policy contains a single stage
description (no `|`), it is applied to all the stages; if a stage description
contains a single thread description (no `;`), it is applied to all the threads
of the stage. Otherwise, the number of stages (resp. threads) of the policy
must match the pipeline (resp. the stage).

Then, the pinning policy can contains all the available `hwloc` objects. Below 
is the correspondence between the `std::string` and the `hwloc` object types:

| String    | `hwloc` object type  | String | `hwloc` object type | String | `hwloc` object type  |
| --------- | -------------------- | ------ | ------------------- | ------ | -------------------- |
| `MACHINE` | `HWLOC_OBJ_MACHINE`  | `L1D`  | `HWLOC_OBJ_L1CACHE` | `L1I`  | `HWLOC_OBJ_L1ICACHE` |
| `GROUP`   | `HWLOC_OBJ_GROUP`    | `L2D`  | `HWLOC_OBJ_L2CACHE` | `L2I`  | `HWLOC_OBJ_L2ICACHE` |
| `NUMA`    | `HWLOC_OBJ_NUMANODE` | `L3D`  | `HWLOC_OBJ_L3CACHE` | `L3I`  | `HWLOC_OBJ_L3ICACHE` |
| `PACKAGE` | `HWLOC_OBJ_PACKAGE`  | `L4D`  | `HWLOC_OBJ_L4CACHE` | `CORE` | `HWLOC_OBJ_CORE`     |
|           |                      | `L5D`  | `HWLOC_OBJ_L5CACHE` | `PU`   | `HWLOC_OBJ_PU`       |

To specify the index `X` of an `hwloc` object, the following syntax is used: 
`OBJECT_X` (ex: `PU_5` refers to the logical PU n°5).

!!! info
    `CORE` and `PU` objects can be confusing. If the CPU cores do not support
    SMT, then `CORE` and `PU` are the same. However, if the CPU cores support
    SMT, then the `PU` is the hardware thread identifier inside a given `CORE`.

### Illustrative Examples

This section gives some examples to understand how the syntax works. We 
suppose that we have a CPU with 8 PUs with the same topology as the the Orange 
Pi 5 Plus SBC presented before.

#### Example 1

Let's suppose we want to setup a 3-stage pipeline with the following 
characteristics:

- **Stage 1** - No replication (= 1 thread): 
     - Pinned to `PU_0`
- **Stage 2** - 4 replications (= 4 threads): 
     - Thread n°1 is pinned to `PU_4` or `PU_5`
     - Thread n°2 is pinned to `PU_4` or `PU_5`
     - Thread n°3 is pinned to `PU_6` or `PU_7`
     - Thread n°4 is pinned to `PU_6` or `PU_7`
- **Stage 3** -  No replication (= 1 thread): 
     - Pinned to `PU_0`, `PU_1`, `PU_2` or `PU_3`

```mermaid
graph LR;
S1T1(Stage 1, thread 1 - pin: PU_0)-->SYNC1;
SYNC1(Sync)-->S2T1;
SYNC1(Sync)-->S2T2;
SYNC1(Sync)-->S2T3;
SYNC1(Sync)-->S2T4;
S2T1(Stage 2, thread 1 - pin: PU_4 or PU_5)-->SYNC2;
S2T2(Stage 2, thread 2 - pin: PU_4 or PU_5)-->SYNC2;
S2T3(Stage 2, thread 3 - pin: PU_6 or PU_7)-->SYNC2;
S2T4(Stage 2, thread 4 - pin: PU_6 or PU_7)-->SYNC2;
SYNC2(Sync)-->S3T1(Stage 3, thread 1 - pin: PU_0, PU_1, PU_2 or PU_3);
```

In the previous configuration, 6 threads will execute simultaneously (even if 
the given architecture supports up to 8 executions in parallel).

To instantiate this `runtime::Pipeline`, here are the corresponding constructor 
parameters:  

- Number of replications (= threads) per stage: `{ 1, 4, 1 }`
- Enabling pinning per stage: `{ true, true, true }`  
- Pinning policy: 
  `"PU_0 | PU_4, PU_5; PU_4, PU_5; PU_6, PU_7; PU_6, PU_7 | PU_0, PU_1, PU_2, PU_3"`

The previous pinning policy syntax can be compressed a little bit as follow:

- Pinning policy : 
  `"PU_0 | PACKAGE_1; PACKAGE_1; PACKAGE_2; PACKAGE_2 | PACKAGE_0"`

#### Example 2

Let's now consider that we want to pin all the threads of the stage 2 on the 
`PU_4`, `PU_5`, `PU_6` or `PU_7` (this is less restrictive than the previous 
example). The pinning strategy for stage 1 and 3 is unchanged.

```mermaid
graph LR;
S1T1(Stage 1, thread 1 - pin: PU_0)-->SYNC1;
SYNC1(Sync)-->S2T1;
SYNC1(Sync)-->S2T2;
SYNC1(Sync)-->S2T3;
SYNC1(Sync)-->S2T4;
S2T1(Stage 2, thread 1 - pin: PU_4, PU_5, PU_6 or PU_7)-->SYNC2;
S2T2(Stage 2, thread 2 - pin: PU_4, PU_5, PU_6 or PU_7)-->SYNC2;
S2T3(Stage 2, thread 3 - pin: PU_4, PU_5, PU_6 or PU_7)-->SYNC2;
S2T4(Stage 2, thread 4 - pin: PU_4, PU_5, PU_6 or PU_7)-->SYNC2;
SYNC2(Sync)-->S3T1(Stage 3, thread 1 - pin: PU_0, PU_1, PU_2 or PU_3);
```

Here are the corresponding parameters: 

- Number of replications (= threads) per stage: `{ 1, 4, 1 }`
- Enabling pinning per stage: `{ true, true, true }`  
- Pinning policy : `"PU_0 | PACKAGE_1, PACKAGE_2 | PACKAGE_0"`

With the previous syntax, the 4 threads of the stage 2 will apply the 
`PACKAGE_1, PACKAGE_2` policy.

#### Example 3

It is also possible to choose the stages we want to pin or not using a vector of 
`boolean`. Let's suppose we do not want to specify any pinning for the stage 1. 

```mermaid
graph LR;
S1T1(Stage 1, thread 1 - no pinning)-->SYNC1;
SYNC1(Sync)-->S2T1;
SYNC1(Sync)-->S2T2;
SYNC1(Sync)-->S2T3;
SYNC1(Sync)-->S2T4;
S2T1(Stage 2, thread 1 - pin: PU_4, PU_5, PU_6 or PU_7)-->SYNC2;
S2T2(Stage 2, thread 2 - pin: PU_4, PU_5, PU_6 or PU_7)-->SYNC2;
S2T3(Stage 2, thread 3 - pin: PU_4, PU_5, PU_6 or PU_7)-->SYNC2;
S2T4(Stage 2, thread 4 - pin: PU_4, PU_5, PU_6 or PU_7)-->SYNC2;
SYNC2(Sync)-->S3T1(Stage 3, thread 1 - pin: PU_0, PU_1, PU_2 or PU_3);
```

Here are the corresponding parameters:

- Number of replications (= threads) per stage: `{ 1, 4, 1 }`
- Enabling pinning per stage: `{false, true, true}`  
- Pinning policy: `"| PACKAGE_1, PACKAGE_2 | PACKAGE_0"`

In this case, the OS will be in charge of pinning the thread of the first stage.
  
## The `Thread_pinning` Class

The `tools::Thread_pinning` class (static methods only) is used by the
sequences and pipelines, and can also be used directly to pin any thread:

| Method | Description |
| ------ | ----------- |
| `static void init()` | Loads the `hwloc` topology (to call once, before any pinning). |
| `static void destroy()` | Releases the `hwloc` topology. |
| `static bool is_init()` | `true` if `init()` has been called. |
| `static void pin(const size_t puid)` | Pins the calling thread on the PU of logical index `puid`. |
| `static void pin(const std::string hwloc_objects)` | Pins the calling thread on a list of `hwloc` objects (ex: `"PU_0, CORE_3"`). |
| `static void unpin()` | Unpins the calling thread: the OS can schedule it on all the PUs. |
| `static std::string get_cur_cpuset_str()` | The current cpuset of the calling thread. |
| `static void set_logs(const bool)`, `static bool is_logs()` | Enables logs on the pinning operations. |

The threads of the sequences and pipelines are pinned at the beginning of
`exec()` and unpinned at the end.

!!! warning
    We assume that the user is aware of the computer architecture, uses the 
    logical indexes of `hwloc` and follows the previously defined syntax rules, 
    otherwise the code will throw exceptions.
