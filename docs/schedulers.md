# Schedulers

Choosing the stages of a [pipeline](pipeline_usage.md) and the number of
threads of each stage can be tedious. `StreamPU` provides **schedulers**
(namespace `spu::sched`) that profile a sequence and generate a pipeline
automatically.

| Scheduler | Class | Strategy |
| --------- | ----- | -------- |
| OTAC | `sched::Scheduler_OTAC` | Finds the stages and the number of threads per stage that maximize the throughput, with at most `R` threads in total. |
| GR | `sched::Scheduler_GR` | One stage per task; each replicable task is replicated `R` times. |
| FILE | `sched::Scheduler_from_file` | Reads the decomposition from a JSON file. |

!!! warning
    The schedulers only support linear chains of tasks **without control
    flow**: the sequence given to a scheduler must be executed by **one
    thread** and must not contain a `Switcher`.

## Usage

```cpp
// 1. build the chain and a single-threaded sequence
runtime::Sequence seq(ini("initialize"), 1);

// 2. create the scheduler (here: OTAC with at most 4 threads)
sched::Scheduler_OTAC sched(seq, 4);

// 3. profile the tasks: the sequence is executed 100 times
sched.profile(100);
sched.print_profiling(); // optional: display the profiling

// 4. compute the solution
sched.schedule();
for (auto& s : sched.get_solution())
    std::cout << "stage: " << s.first << " tasks, " << s.second << " threads" << std::endl;

// 5. create the pipeline (here: buffers of 16, passive waiting, no pinning)
std::unique_ptr<runtime::Pipeline> pipeline(sched.instantiate_pipeline(16, false));
pipeline->exec([&counter]() { return ++counter >= 1000; });
```

The tasks are taken **in the execution order of the sequence**: a stage
contains a number of consecutive tasks of the sequence. The pipeline returned
by `instantiate_pipeline()` is allocated with `new` and owned by the caller.
It takes the ownership of the output buffers: the original sequence must not
be executed anymore.

`generate_pipeline()` performs all the steps at once (profiling with 100
executions if not done, scheduling if not done, then instantiation with the
synchronization parameters and the thread mapping given by the scheduler).

## API

All the schedulers inherit from `sched::Scheduler`:

| Method | Description |
| ------ | ----------- |
| `void profile(const size_t n_exec = 100)` | Executes the sequence `n_exec` times and records the average duration of each task. Throws if the profiling has already been done (call `reset()` first). |
| `void profile(const std::vector<size_t>& puids, const size_t n_exec = 100)` | Same, once per processing unit (the thread is pinned on each PU in turn). |
| `void print_profiling(std::ostream& stream = std::cout)` | Displays the profiling (task statistics). |
| `const std::vector<task_desc_t>& get_profiling()` | The profiled tasks (`tptr`: pointer to the task, `exec_duration`: the average durations). |
| `virtual void schedule() = 0` | Computes the solution. Requires the profiling. |
| `std::vector<std::pair<size_t, size_t>> get_solution()` | The solution: for each stage, the number of tasks and the number of threads. |
| `runtime::Pipeline* instantiate_pipeline(const size_t buffer_size = 1, const bool active_wait = false, const bool thread_pining = false, const std::string& pinning_policy = "")` | Creates the pipeline, the same synchronization parameters being used between all the stages. |
| `runtime::Pipeline* instantiate_pipeline(const std::vector<size_t> synchro_buffer_sizes, const std::vector<bool> synchro_active_waitings, const std::vector<bool> thread_pinings, const std::string& pinning_policy)` | Same, with per-stage parameters. |
| `runtime::Pipeline* generate_pipeline()` | Profiles, schedules and instantiates the pipeline. |
| `std::string get_threads_mapping() const` | A [pinning policy](thread_pinning.md) for the solution. By default, the threads are pinned on consecutive PUs (`PU_0` for the first thread, `PU_1` for the second, and so on). |
| `std::vector<size_t> get_sync_buff_sizes() const` | Synchronization buffer sizes (default: 1). |
| `std::vector<bool> get_sync_active_waitings() const` | Synchronization waiting types (default: passive). |
| `std::vector<bool> get_thread_pinnings() const` | Per-stage pinning (default: `true` if `tools::Thread_pinning::init()` has been called). |
| `size_t get_n_alloc_ressources() const` | Total number of threads of the solution. |
| `virtual double get_throughput_est() const` | Estimated throughput (streams per second). Only implemented by OTAC. |
| `virtual void reset()` | Clears the profiling and the solution. |

!!! note
    The profiling enables the statistics and the fast mode of all the tasks of
    the sequence, and they stay enabled after the profiling.

## OTAC

```cpp
Scheduler_OTAC(runtime::Sequence& sequence, const size_t R = std::thread::hardware_concurrency());
```

`R` is the maximum number of threads (resources). The algorithm performs a
binary search on the period `P` (the inverse of the throughput): for a given
`P`, a greedy algorithm packs consecutive tasks into stages whose weight does
not exceed `P` (replicable stages being replicated to fit `P`), and checks that
no more than `R` threads are used. The solution with the smallest feasible `P`
is kept. The non-replicable tasks are never replicated.

Additional methods: `double get_period() const` (the period of the solution,
in nanoseconds) and `get_throughput_est()`.

## GR

```cpp
Scheduler_GR(runtime::Sequence& sequence, const size_t R = std::thread::hardware_concurrency());
```

Creates one stage per task. A replicable task is executed by `R` threads, a
non-replicable task by one thread. There is no limit on the total number of
threads.

## FILE

```cpp
Scheduler_from_file(runtime::Sequence& sequence, const std::string filename, uint8_t file_version = 2);
```

Reads the solution from a JSON file. The sum of the `tasks` fields must be
equal to the number of tasks of the sequence. The profiling is still required
before `instantiate_pipeline()` (it provides the list of tasks), but its
results are not used. With this scheduler, `get_sync_buff_sizes()`,
`get_sync_active_waitings()`, `get_thread_pinnings()` and
`get_threads_mapping()` return the values read from the file:

```cpp
sched::Scheduler_from_file sched(seq, "sched.json");
sched.profile();
sched.schedule();
std::unique_ptr<runtime::Pipeline> pipeline(
    sched.instantiate_pipeline(sched.get_sync_buff_sizes(),
                               sched.get_sync_active_waitings(),
                               sched.get_thread_pinnings(),
                               sched.get_threads_mapping()));
```

### File Format (version 2, default)

```json
{
  "resources": {
    "p-core": { "node-list": ["PU0-3"], "smt": 1 },
    "e-core": { "node-list": ["PU4-7"], "smt": 1 }
  },
  "schedule": [
    { "tasks": 1, "threads": 1, "core-type": "p-core", "pinning-policy": "packed", "sync_buff_size": 4, "sync_waiting_type": "passive" },
    { "tasks": 3, "threads": 3, "core-type": "e-core", "pinning-policy": "packed", "sync_buff_size": 4 },
    { "tasks": 1, "threads": 1, "core-type": "p-core", "pinning-policy": "loose" }
  ]
}
```

The file must be valid JSON (no comments, no trailing commas).

`resources` (required) describes the two types of cores: `p-core`
(performance) and `e-core` (efficiency). Both are required, with:

- `node-list`: a list of processing units. Each item is a PU index (integer),
  `"PU<n>"`, a range `"PU<a>-<b>"`, a core `"core<n>"` or a range of cores
  `"core<a>-<b>"`. The `core` forms require `StreamPU` to be linked with
  `hwloc`. The indexes are `hwloc` logical indexes.
- `smt` (required, as `node-list`): the number of hardware threads per core. When `smt > 1`, the
  second (third, ...) hardware threads of the cores are placed at the end of the
  list, so that they are used last.

`schedule` (required) contains one entry per pipeline stage, in order:

| Field | Required | Description |
| ----- | -------- | ----------- |
| `tasks` | yes | Number of consecutive tasks in the stage. |
| `threads` | yes | Number of threads of the stage. |
| `core-type` | yes | `"p-core"` or `"e-core"`. |
| `pinning-policy` | yes | `"packed"`, `"guided"`, `"distant"` or `"loose"` (an unknown value is replaced by `"loose"`). |
| `sync_buff_size` | no | Size of the synchronization buffer after this stage (default 1). Must not be set on the last stage (with the version 2 format, this is **not** checked by the parser and leads to an out-of-bounds write). |
| `sync_waiting_type` | no | `"active"` or `"passive"` (default), for the synchronization after this stage. Must not be set on the last stage (an exception is thrown otherwise). |

The pinning policies are:

- `packed`: each thread is pinned on the next available PU of the list of its
  core type (the PUs are consumed in the order of the list, across the
  stages);
- `guided`: each thread can run on any PU of its core type (the PUs are not
  consumed);
- `distant`: the threads are spread over the list of PUs of their core type,
  alternating between the beginning and the middle of the list (the PUs are
  consumed);
- `loose`: the threads are not restricted (pinned on the whole machine).

If the thread pinning is not available (`StreamPU` not linked with `hwloc`),
the policies have no effect.

### File Format (version 1)

With `file_version = 1`, `schedule` is an array of stages with the fields
`cores` (the number of threads, or a list of PU indexes), `tasks`, and the
optional `sync_buff_size` and `sync_waiting_type` fields described above.
`resources` is not used.

## From the Command Line

The [`spu-benchmark`](benchmark.md) application exposes the schedulers with the
`-C` (chain description), `-S` (scheduler: `OTAC`, `GR` or `FILE`), `-F`
(JSON file) and `-l` (number of profiling executions) parameters.
