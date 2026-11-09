# Task

A **task** represents **the code executed by a node in the data flow graph**. In 
other languages, a **task** can be referred to as a *job* or a *filter*. 
A task is defined by its input and output data and the code to execute when 
triggered. In other words, a task comes with a set of data called 
[sockets](socket.md) (not to be confused with network and system sockets). The 
sockets model the data that are consumed (input socket) and produced (output 
socket) by the current task. Finally, the code to execute is stored in a 
so-called *codelet*.

A task is a C++ object of the `spu::runtime::Task` class. The following
sections give an overview of the most important attributes and methods to
facilitate the code understanding. The user point of view is described in
[Writing Modules](custom_modules.md).

## Main Attributes

```cpp
std::vector<std::shared_ptr<runtime::Socket>> sockets;
```
The list of sockets of this task, in creation order. The last socket is always
the `status` output socket, created by `create_codelet()`.

```cpp
std::vector<std::shared_ptr<runtime::Socket>> fake_input_sockets;
```
Fake input sockets are used when specifying dependencies between tasks
directly (task-to-task binding). These dependencies are managed through "fake
input sockets" that are created on-the-fly over the current task and bound to
an output socket of the other task (its `status` socket by default). The data
of these sockets are ignored during the codelet execution.

```cpp
std::function<int(module::Module& m, runtime::Task& t, const size_t frame_id)> codelet;
```
The function called by the `_exec()` method (see below), thus dictating the
task's behavior. Usually set in the [module](module.md)'s constructor. Its
return value is written in the `status` socket. By default, the codelet throws
a `tools::unimplemented_error`.

```cpp
std::vector<int> status;
```
The memory of the `status` socket: one value per wave (see
[Frames and Waves](concepts_overview.md#frames-and-waves)).

```cpp
module::Module* module;
```
A pointer to the module that owns the task. See the [Module](module.md)
section.

```cpp
const std::string name;
```
A name to identify the task. This name is unique in the module.

```cpp
bool stats;
```
If `true`, records statistics regarding the task's execution (number of calls,
total, minimum and maximum durations). By default this attribute is set to
`false`.

```cpp
bool fast;
```
If `true`, skips the `can_exec()` runtime check, thus improving performance.
The sockets of this task are also set to `fast` (no type/size checks when they
are bound). By default this attribute is set to `false`. When `SPU_FAST` is
defined, `is_fast()` always returns `true`.

```cpp
bool debug;
bool debug_hex;
int32_t debug_limit;
uint8_t debug_precision;
int32_t debug_frame_max;
```
If `debug` is `true`, the task displays the data of its sockets and its status
upon execution (on the standard output). The other attributes control the
display (see [Debug Mode](debugging.md#debug-mode)).

```cpp
bool replicable;
```
`true` if the task can be replicated in a sequence or a pipeline stage. It is
initialized to `module.is_clonable()` when the task is created.

```cpp
bool outbuffers_allocated;
```
`true` when the memory of the output sockets has been allocated (see
`allocate_outbuffers()`).

```cpp
size_t n_input_sockets;
size_t n_output_sockets;
size_t n_fwd_sockets;
```
The number of sockets of each type (the fake input sockets are counted in
`n_input_sockets`, the `status` socket in `n_output_sockets`). They are used by
the sequence to know when all the inputs of a task have been fed.

```cpp
std::vector<int8_t*> sockets_dataptr_init;
std::vector<size_t> sockets_databytes_per_frame;
std::vector<std::vector<int8_t>> sockets_data;
```
Precomputed values and temporary buffers used by `_exec()` to iterate over the
frames and the waves.

## Main Methods

```cpp
const std::vector<int>& exec(const int frame_id = -1, const bool managed_memory = true);
```
Calls the `_exec()` method, records the execution statistics (if
`stats == true`) and prints the debug logs (if `debug == true`). Unless the
task is in fast mode (with neither `stats` nor `debug` enabled), it first checks
the `frame_id` (`tools::length_error` if it is not `-1` and not smaller than
`n_frames`); if the task is not in fast mode, `can_exec()` is also called and a
`tools::runtime_error` is thrown when it returns `false`. Returns the `status`
values.

```cpp
void _exec(const int frame_id = -1, const bool managed_memory = true);
```
Executes the task's `codelet` and sets the `status` for this specific call.
If `frame_id == -1`, all the frames are processed: the codelet is called once
per wave, the socket data pointers being moved to the first frame of the wave
before each call (and restored after). If the last wave is incomplete, the data
are copied into the `sockets_data` temporary buffers so that the codelet always
processes `n_frames_per_wave` frames. If a codelet returns
`status_t::FAILURE_STOP`, the remaining waves are not executed. If
`frame_id >= 0`, only the corresponding frame (or wave) is processed.

```cpp
bool can_exec() const;
```
Returns `true` if all the sockets are associated to a buffer (`dataptr` is not
`nullptr`), otherwise returns `false`.

```cpp
void operator=(Socket& s_out);
void operator=(Task& t_out);
```
Task-to-task binding: adds a fake input socket to the current task (see the
`fake_input_sockets` attribute) and binds it to `s_out` (an output or forward
socket), or to the `status` socket of `t_out`. The new socket's `datatype` and
`databytes` match the bound socket. If `s_out` is already bound to an input
socket of the task, nothing is done. The `bind()` methods are deprecated
equivalents.

```cpp
size_t unbind(Socket& s_out);
size_t unbind(Task& t_out);
```
Unbinds and deletes the corresponding fake input socket. Returns the position
of the removed binding in the bound sockets of `s_out`.

```cpp
Socket& operator[](const size_t id);
Socket& operator[](const std::string& sck_name);
```
Returns a socket by index (creation order) or by name.

```cpp
void set_replicability(const bool replicable);
```
Forces the task to be non-replicable (`false`). Setting `true` throws if the
module is not clonable.

```cpp
void allocate_outbuffers();
void deallocate_outbuffers();
```
Allocate (resp. deallocate) the memory of the output sockets (the `status`
socket excepted), and propagate the new data pointers to the bound input and
forward sockets. Called by the sequences, or at the task creation when the
*task autoalloc* mode is enabled
(`tools::Buffer_allocator::set_task_autoalloc(true)`).

```cpp
Task* clone() const;
```
Copies the task and its sockets. The output sockets of the clone are not bound
and their memory is not allocated (unless the *task autoalloc* mode is
enabled); the input sockets of the clone point to the same data as the
original ones. Used by `Module::deep_copy()`.

```cpp
void reset();
```
Resets the task's statistics (number of calls, durations and sub-timers). Not
to be confused with the `reset()` method of the modules that implement
`tools::Interface_reset`.

```cpp
void update_timer(const size_t id, const std::chrono::nanoseconds& duration);
void register_timer(const std::string& key); // protected
```
Sub-timers: `register_timer()` creates a named timer, `update_timer()` adds a
duration to it (only when the statistics are enabled).

## Socket Creation (protected)

```cpp
template <typename T>
size_t create_2d_socket_in (const std::string& name, const size_t n_rows, const size_t n_cols);
template <typename T>
size_t create_2d_socket_out(const std::string& name, const size_t n_rows, const size_t n_cols, const bool hack_status = false);
template <typename T>
size_t create_2d_socket_fwd(const std::string& name, const size_t n_rows, const size_t n_cols);
void create_codelet(std::function<int(module::Module&, Task&, const size_t)>& codelet);
```
These methods are called by the `Module` class (`n_rows` already includes the
number of frames). A socket cannot be named `status` (except with
`hack_status`, used to create the `status` socket), two sockets cannot have the
same name, and no socket can be created after the `status` socket.

```cpp
void update_n_frames(const size_t old_n_frames, const size_t new_n_frames);
void update_n_frames_per_wave(const size_t old_n_frames_per_wave, const size_t new_n_frames_per_wave);
```
Called by the module when its number of frames (or of frames per wave)
changes: the sizes of the sockets (and the output buffers) are updated.
