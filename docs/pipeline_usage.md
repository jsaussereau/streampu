# Pipelines

A **pipeline** (`spu::runtime::Pipeline`) splits a graph of tasks into
**stages**. Each stage is a [sequence](sequence_usage.md) executed by its own
thread(s), and consecutive stages exchange their data through synchronization
buffers. The stages are executed in parallel (pipeline parallelism) and each
stage can be replicated over several threads (replication parallelism).

<figure markdown>
  ![Sequence and pipeline](./assets/sequence_to_pipeline.svg){ width=1200 }
  <figcaption>Example of a sequence on the left and a pipeline on the right.</figcaption>
</figure>

The internal implementation is described in the developer page
[Pipeline](pipeline.md).

## Creating a Pipeline

### Example

The following example is extracted from the `tests/bootstrap/simple_pipeline.cpp`
test. The chain reads a file (`Source_user_binary`), relays the data through 6
`Relayer` modules, and writes the data in another file (`Sink_user_binary`).
Reading and writing files are sequential operations: the first and the last
stages are executed by one thread, while the relays are replicated.

```cpp
module::Source_user_binary<uint8_t> source(data_length, in_filepath, false /* auto_reset */);
module::Sink_user_binary<uint8_t> sink(data_length, out_filepath);
std::vector<std::shared_ptr<module::Relayer<uint8_t>>> rlys(6);
for (size_t s = 0; s < rlys.size(); s++)
    rlys[s].reset(new module::Relayer<uint8_t>(data_length));

// sockets binding
(*rlys[0])["relay::in"] = source["generate::out_data"];
for (size_t s = 0; s < rlys.size() - 1; s++)
    (*rlys[s + 1])["relay::in"] = (*rlys[s])["relay::out"];
sink["send_count::in_data"] = (*rlys[rlys.size() - 1])["relay::out"];
sink["send_count::in_count"] = source["generate::out_count"];

runtime::Pipeline pipeline(
  source("generate"), // first task of the whole graph
  {
    // pipeline stage 0: first & last tasks
    { { &source("generate") }, { &source("generate") } },
    // pipeline stage 1: first & last tasks
    { { &(*rlys[0])("relay") }, { &(*rlys[rlys.size() - 1])("relay") } },
    // pipeline stage 2: first task (no need to specify the last task)
    { { &sink("send_count") }, {} },
  },
  {
    1, // number of threads in the stage 0
    4, // number of threads in the stage 1
    1  // number of threads in the stage 2
  },
  {
    16, // synchronization buffer size between stages 0 and 1
    16, // synchronization buffer size between stages 1 and 2
  },
  {
    false, // type of waiting between stages 0 and 1 (true = active, false = passive)
    false, // type of waiting between stages 1 and 2 (true = active, false = passive)
  });

pipeline.exec(); // stops when the source reaches the end of the file
```

### Constructors

```cpp
Pipeline(runtime::Task& first,
         const std::vector<std::pair<std::vector<runtime::Task*>, std::vector<runtime::Task*>>>& sep_stages = {},
         const std::vector<size_t>& n_threads = {},
         const std::vector<size_t>& synchro_buffer_sizes = {},
         const std::vector<bool>& synchro_active_waiting = {},
         const std::vector<bool>& thread_pinning = {},
         const std::vector<std::vector<size_t>>& puids = {});

Pipeline(runtime::Task& first,
         runtime::Task& last,
         /* same parameters */);

Pipeline(const std::vector<runtime::Task*>& firsts,
         /* same parameters */);

Pipeline(const std::vector<runtime::Task*>& firsts,
         const std::vector<runtime::Task*>& lasts,
         /* same parameters */);
```

The parameters are:

- `first`/`firsts` and `last`/`lasts`: the first (and last) tasks of the
  **whole** graph, as for a [sequence](sequence_usage.md#main-constructors).
- `sep_stages`: the description of the stages, one element per stage. Each
  element is a pair of vectors: the first tasks and the last tasks of the
  stage. Each stage is built as a sequence from its first tasks, the
  exploration stopping at its last tasks. The last tasks of the last stage can
  be left empty.
- `n_threads`: the number of threads of each stage (1 by default). Its size
  must be equal to the number of stages (or 0).
- `synchro_buffer_sizes`: the number of buffers (of `n_frames` frames)
  between two consecutive stages (1 by default). Its size must be equal to
  the number of stages minus one (or 0).
- `synchro_active_waiting`: for each synchronization, `true` for active waiting
  (the threads spin when the buffer is full or empty: lower latency, but the
  cores are always busy), `false` for passive waiting (condition variables).
  Passive waiting is the default. Its size must be equal to the number of
  stages minus one (or 0).
- `thread_pinning` and `puids`: per-stage [thread pinning](thread_pinning.md)
  (sizes equal to the number of stages, or 0).

The constructors taking `firsts` (and `lasts`) vectors also exist with a
`std::tuple` of three vectors per stage (first tasks, last tasks,
**excluded tasks**) instead of a pair, to exclude tasks from a stage:

```cpp
Pipeline(const std::vector<runtime::Task*>& firsts,
         const std::vector<runtime::Task*>& lasts,
         const std::vector<std::tuple<std::vector<runtime::Task*>,
                                      std::vector<runtime::Task*>,
                                      std::vector<runtime::Task*>>>& sep_stages = {},
         /* same parameters */);
```

Finally, all the constructors have a variant where `puids` is replaced by a
`const std::string& pipeline_pinning_policy` (see
[Thread Pinning](thread_pinning.md#sequence-pipeline)). In this variant, all
the parameters are mandatory.

### Rules

- The stages are given in the order of the data flow.
- Together, the stages must contain exactly the tasks of the sequence built
  from `firsts` (and `lasts`). Otherwise an exception is raised and two DOT
  files are written in the current directory to help the debugging:
  `dbg_ref_sequence.dot` (the expected graph) and `dbg_cur_pipeline.dot` (the
  graph built from the stages).
- A stage executed by more than one thread must only contain
  [replicable tasks](concepts_overview.md#module-states-tasks-replicability).
- Consecutive stages can be replicated: the synchronizations support *m*
  producer threads and *n* consumer threads and **preserve the order of the
  frames**.
- The bindings between two stages (even non-consecutive ones) are
  automatically replaced by synchronization tasks (`push` in the producer
  stage, `pull` in the consumer stage). Forward sockets crossing several stages
  are supported (see [Forward Socket](socket_fwd.md)).
- A control flow structure (the `commute` and `select` tasks of a
  [`Switcher`](control_flow.md)) must be entirely contained in one stage, and a
  stage cannot end with a `commute` task (see
  [Work in Progress](wip.md#End-of-sequence)).

## Executing a Pipeline

```cpp
void exec();
void exec(std::function<bool()> stop_condition);
void exec(std::function<bool(const std::vector<const int*>&)> stop_condition);
void exec(const std::vector<std::function<bool()>>& stop_conditions);
void exec(const std::vector<std::function<bool(const std::vector<const int*>&)>>& stop_conditions);
```

`exec()` starts all the stages in parallel and returns when all of them are
stopped. With a single stop condition, the condition is evaluated by the
threads of the **last stage** only. With a vector of conditions (one per
stage), each stage evaluates its own condition. In all cases:

- a stage stops when its stop condition is met, when one of its modules is
  done (auto-stop mode, enabled by default, see
  [Sequences](sequence_usage.md#auto-stop)), or when a `SIGINT` is caught;
- when a stage stops, the next stage is notified and stops once the
  synchronization buffer is empty;
- when the last stage stops, all the previous stages are stopped.

```cpp
std::atomic<size_t> counter(0);
pipeline.exec([&counter]() { return ++counter >= 1000; }); // ~1000 executions of the last stage
```

```cpp
void set_auto_stop(const bool auto_stop); // applied to all the stages
bool is_auto_stop() const;
```

## Number of Frames

```cpp
size_t get_n_frames() const;
void set_n_frames(const size_t n_frames);
```

Sets the number of frames of all the stages and of the synchronization
buffers.

## Accessing the Stages, Modules and Tasks

| Method | Description |
| ------ | ----------- |
| `std::vector<Sequence*> get_stages()` | The stages (one sequence per stage). |
| `Sequence& operator[](const size_t stage_id)` | A given stage. |
| `std::vector<C*> get_modules<C>(const bool set_modules = true) const` | All the modules of type `C` of all the stages (synchronization modules included). |
| `get_modules_per_threads()`, `get_modules_per_types()`, `get_tasks_per_threads()`, `get_tasks_per_types()` | Same as in [Sequences](sequence_usage.md#accessing-the-modules-and-tasks), for all the stages. |
| `std::vector<std::vector<module::Module*>> get_original_modules() const` | The modules of the original sequence. |
| `get_firsts_tasks()`, `get_lasts_tasks()` | First tasks of the first stage and last tasks of the last stage (as reported by these stages). |
| `void export_dot(std::ostream& stream = std::cout) const` | Exports the pipeline (stages and synchronizations) in the DOT format. |

The statistics can be displayed per stage:

```cpp
auto stages = pipeline.get_stages();
for (size_t s = 0; s < stages.size(); s++)
{
    std::cout << "# Pipeline stage " << s << std::endl;
    tools::Stats::show(stages[s]->get_tasks_per_types(), true, false);
}
```

## Synchronization Tasks

The synchronizations are implemented by the `module::Adaptor_m_to_n` module,
automatically inserted by the pipeline. They are **bound** to the tasks of the
stages during the construction, and **unbound** (the original bindings being
restored) in the destructor:

```cpp
void bind_adaptors();
void unbind_adaptors();
bool is_bound_adaptors() const;
```

Calling `unbind_adaptors()` restores the original bindings of the user tasks
(for instance, to build another sequence or pipeline from the same modules).
`exec()` requires the adaptors to be bound.

## Pipeline Builder

The `tools::Pipeline_builder` class builds a pipeline step by step with a
fluent interface. It is equivalent to the constructors above.

```cpp
runtime::Pipeline pipeline =
  tools::Pipeline_builder()
    .add_task_for_checking(source("generate"))   // = the 'firsts' of the whole graph
    .add_stage(tools::Pipeline_builder::Stage_builder()
                 .add_first_task(source("generate"))
                 .add_last_task(source("generate"))
                 .set_n_threads(1))
    .configure_interstage_synchro(tools::Pipeline_builder::Synchro_builder()
                 .set_active_waiting(false)
                 .set_buffer_size(16))
    .add_stage(tools::Pipeline_builder::Stage_builder()
                 .add_first_task((*rlys[0])("relay"))
                 .add_last_task((*rlys[rlys.size() - 1])("relay"))
                 .set_n_threads(4))
    .configure_interstage_synchro(tools::Pipeline_builder::Synchro_builder()
                 .set_active_waiting(false)
                 .set_buffer_size(16))
    .add_stage(tools::Pipeline_builder::Stage_builder()
                 .add_first_task(sink("send_count"))
                 .set_n_threads(1))
    .build();
```

!!! warning
    The builder stores pointers to the `Stage_builder` and `Synchro_builder`
    objects. Temporaries are fine in a single expression as above; otherwise,
    the builder objects must live until `build()` is called.

`Pipeline_builder` methods:

| Method | Description |
| ------ | ----------- |
| `add_task_for_checking(Task&)`, `set_tasks_for_checking(std::vector<Task*>)`, `remove_task_for_checking(const Task*)` | First tasks of the whole graph. |
| `add_stage(Stage_builder&)`, `remove_stage(const Stage_builder&)`, `remove_stage(size_t)`, `get_stage(size_t)`, `get_n_stages()` | Manage the stages (in order). |
| `configure_interstage_synchro(Synchro_builder&)`, `get_interstage_synchro(size_t)` | Synchronization `i` is between stages `i` and `i+1`. Unconfigured synchronizations use the default values. |
| `build()`, `build_ptr()` | Create the pipeline (`build_ptr()` returns a pointer owned by the caller). |

`Stage_builder` methods: `set_n_threads()` (default 1),
`add_first_task()`/`set_first_tasks()`/`remove_first_task()`,
`add_last_task()`/`set_last_tasks()`/`remove_last_task()`,
`add_excluded_task()`/`set_excluded_tasks()`/`remove_excluded_task()`,
`enable_threads_pinning()`/`disable_threads_pinning()`/`set_threads_pinning()`
(default disabled) and `set_pinning_policy()` (policy of the stage, without
the `|` separator).

`Synchro_builder` methods: `set_buffer_size()` (default 1),
`enable_active_waiting()`/`disable_active_waiting()`/`set_active_waiting()`
(default passive).

## Automatic Pipeline Generation

Instead of describing the stages manually, a [scheduler](schedulers.md) can
profile a sequence and generate a pipeline automatically.
