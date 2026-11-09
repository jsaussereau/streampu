# Module

A **module** is a container of [tasks](task.md). It contains the tasks 
themselves and their *inner data*. Inner data can be useful to avoid useless 
buffers re-allocations each time a task is executed, or to contain a state 
that is updated each time a task is triggered. Multiple **tasks can be grouped 
into one module** and can share data through the common module (as opposed to 
the use of [sockets](socket.md)).

The `spu::module::Module` class is the base class of all the modules. It cannot
be instantiated directly (protected constructor) and its methods to create
tasks, sockets and codelets are **private**. Two classes inherit from it and
expose these methods:

- `spu::module::Stateful`: the methods are `protected`; a new module is a class
  that inherits from `Stateful` and creates its tasks in its constructor;
- `spu::module::Stateless`: the methods are `public`; a `Stateless` object can
  be filled with tasks without writing a new class. A stateless module is
  always clonable.

See [Writing Modules](custom_modules.md) for the user point of view and
[Module States & Tasks Replicability](concepts_overview.md#module-states-tasks-replicability)
for the clonability rules.

## Main Attributes

```cpp
std::vector<std::shared_ptr<runtime::Task>> tasks;
```
The tasks list of the current module (public). All the tasks in the vector are 
allocated (no `nullptr`).

```cpp
std::vector<std::shared_ptr<runtime::Task>> tasks_with_nullptr;
```
The tasks list of the current module where the tasks have a fixed position in
the vector (given by the `id` parameter of `create_task()`). This is useful when
a task is conditionally created. In the case of a task that is not created in
the current module, its value is set to `nullptr`. `operator[](size_t)` uses
this vector.

```cpp
size_t n_frames;
size_t n_frames_per_wave;
size_t n_waves;
size_t n_frames_per_wave_rest;
bool single_wave;
```
Number of frames to process each time a task is executed (1 by default). For 
instance, if `n_frames == 2`, all the tasks of the current module will 
process 2 frames each time they are triggered. The frames are processed by
waves of `n_frames_per_wave` frames (1 by default): `n_waves` is
`ceil(n_frames / n_frames_per_wave)` and `n_frames_per_wave_rest` is
`n_frames % n_frames_per_wave`. If `single_wave` is `true`,
`n_frames_per_wave` is kept equal to `n_frames`. See
[Frames and Waves](custom_modules.md#frames-and-waves).

```cpp
std::string name;
```
Name of the Module (`"Module"` by default). This name is the same for all the
instances of one class.

```cpp
std::string short_name;
```
Short name of the Module (`"Module"` by default). This name is the same for all
the instances of one class.

```cpp
std::string custom_name;
```
Custom name of the Module. This name can be redefined by the user for each
instance (empty by default).

## Methods to Create Tasks (private, exposed by `Stateful` and `Stateless`)

```cpp
void set_name(const std::string& name);
void set_short_name(const std::string& short_name);
```
Set the module name and short name.

```cpp
runtime::Task& create_task(const std::string& name, const int id = -1);
```
Creates a new task, two tasks cannot share the same `name`.

```cpp
template <typename T>
size_t create_socket_in(runtime::Task& task, const std::string& name, const size_t n_elmts);
template <typename T>
size_t create_socket_out(runtime::Task& task, const std::string& name, const size_t n_elmts);
template <typename T>
size_t create_socket_fwd(runtime::Task& task, const std::string& name, const size_t n_elmts);
```
Create an input, output or forward socket over a given task. `n_elmts` is the
number of elements per frame: the socket is created with `n_frames` rows. The
`create_2d_socket_*()` variants create sockets of `n_rows × n_cols` elements per
frame. All these methods also exist with a runtime type parameter
(`std::type_index` or `runtime::datatype_t`) and with the `create_sck_*()`
short names.

```cpp
void create_codelet(runtime::Task& task, std::function<int(Module& m, runtime::Task& t, const size_t frame_id)> codelet);
```
Creates the codelet of the given task (and its `status` socket).

```cpp
void register_timer(runtime::Task& task, const std::string& key);
virtual void set_n_frames_per_wave(const size_t n_frames_per_wave);
void set_single_wave(const bool enable_single_wave);
void deep_copy(const Module& m); // exposed by Stateful only
```
Register a sub-timer, change the wave configuration, and clone the tasks of
another module into the current one (used by the `clone()` methods).

## Main Public Methods

```cpp
virtual Module* clone() const;
```
Returns a copy of the module. Throws `tools::unimplemented_error` in the
`Module` class: the modules that support the replication override it.

```cpp
size_t get_n_frames() const;
virtual void set_n_frames(const size_t n_frames);
size_t get_n_frames_per_wave() const;
size_t get_n_waves() const;
size_t get_n_frames_per_wave_rest() const;
bool is_single_wave() const;
```
Get or set the number of frames to process each time a task is executed. When
the number of frames changes, the sockets of all the tasks are resized.

```cpp
const std::string& get_name() const;
const std::string& get_short_name() const;
void set_custom_name(const std::string& custom_name);
const std::string& get_custom_name() const;
void remove_custom_name();
```
Get the names, and set/get/remove the custom name (each instance can have a
different custom name).

```cpp
runtime::Socket& operator[](const std::string& tsk_sck);
```
Returns the socket if it exists, otherwise throws a `tools::invalid_argument`
exception. The expected string format is `"task_name::socket_name"` (the
spaces are ignored).

```cpp
runtime::Task& operator()(const std::string& tsk_name);
runtime::Task& operator[](const size_t id);
```
Returns the task by name (throws a `tools::invalid_argument` exception if it
does not exist) or by index in `tasks_with_nullptr` (checked with `assert`
only).

```cpp
void set_fast(const bool fast);
```
Sets the fast mode of all the tasks of the module.

```cpp
void create_reset_task();
```
If the module implements `tools::Interface_reset`, adds a task named `reset`
that calls `reset()`.

```cpp
bool is_stateless() const;
bool is_stateful() const;
bool is_clonable() const;
```
`is_stateless()` returns `true` if the module is a `module::Stateless` (or
derived) object. `is_clonable()` returns `true` for stateless modules, and
otherwise tries to call `clone()` and returns `false` if it throws a
`tools::unimplemented_error`.
