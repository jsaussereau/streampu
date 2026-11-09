# Socket

Sockets are used to exchange data between [tasks](task.md). There are 3 
different types of sockets:

- Input socket (`socket_t::SIN`): read only data,
- Output socket (`socket_t::SOUT`): write only data (the socket owns the memory),
- Forward socket (`socket_t::SFWD`): read and write data (in place).

A task can have multiple sockets of different types (input, output and forward).
This is illustrated in the following figure:

<figure markdown>
  ![Task with sockets!](./assets/task_sockets.svg){ width="600" }
  <figcaption>Tasks with different socket types.</figcaption>
</figure>

A socket is a C++ object of the `spu::runtime::Socket` class. The following
sections give an overview of the most important attributes and methods to
facilitate the code understanding. The user point of view is described in
[Concepts](concepts_overview.md#binding) and [Writing Modules](custom_modules.md).


## Main Attributes

```cpp
socket_t type;
```
The socket type: `socket_t::SIN`, `socket_t::SOUT` or `socket_t::SFWD`.

```cpp
std::string name;
```
The name of the socket (unique in its task).

```cpp
Task& task;
```
The task that owns the socket.

```cpp
std::type_index datatype;
```
The type of data exchanged (one of the 10 supported C++ arithmetic types).

```cpp
size_t databytes;
```
The size of the socket in bytes, **all the frames included**.

```cpp
size_t n_rows;
void** rowsptr;
```
The number of rows (`n_frames` for a 1D socket, `n_frames × n_rows` for a 2D
socket) and the array of row pointers returned by `get_2d_dataptr()`.

```cpp
void* dataptr;
```
Pointer to the data of the socket (memory space). For an input or forward
socket, it points to the memory of the bound socket (or to user data).

```cpp
buffer out_buffer;
```
The memory of an output socket (a 64-byte aligned `std::vector<uint8_t>`),
allocated by `allocate_buffer()`.

```cpp
std::vector<Socket*> bound_sockets;
```
The `input` or `forward` sockets bound to the current socket. Only relevant 
for `output` or `forward` sockets.

```cpp
Socket* bound_socket;
```
The unique `output` or `forward` socket bound to the current socket. Only
relevant for `input` or `forward` sockets.

```cpp
bool fast;
```
If `true`, the type, size and allocation checks are skipped when the socket is
bound.

## Main Methods

The most important methods of the socket class are the binding operators.

```cpp
void operator=(Socket& s);
```
Binds two sockets. The direction is deduced from the socket types: if `s` is an
output (or forward) socket and the current socket an input (or forward) socket,
the current socket is bound to `s`; in the opposite case, `s` is bound to the
current socket. The bound socket gets the `dataptr` of the output socket and is
added to its `bound_sockets`. Unless the socket is in fast mode, the data types
and sizes must be the same. An input socket cannot be bound twice and a socket
cannot be bound twice to the same output socket (these two checks are only
disabled when `SPU_FAST` is defined). A `tools::runtime_error` is thrown when a
check fails.

```cpp
void bind(Socket& s_out, const int priority = -1); // deprecated
```
Same as `operator=`, the current socket being the input. `priority` gives the
position of the current socket in the `bound_sockets` of `s_out`.

```cpp
template <typename T, class A = std::allocator<T>> void operator=(std::vector<T, A>& vector);
template <typename T> void operator=(T* array);
void operator=(Task& t);
```
Bind an input (or forward) socket to user data (a vector of exactly
`get_n_elmts()` elements, or a pointer whose type matches the socket data
type), or bind a task to the current output socket (see
[Task-to-task binding](getting_started.md#task-to-task-binding)).

```cpp
template <typename T> T* get_dataptr(const size_t start_col = 0) const;
template <typename T> T** get_2d_dataptr(const size_t start_row = 0, const size_t start_col = 0);
```
Return the data of the socket (`T` must be `const` for an input socket).

```cpp
size_t get_n_elmts() const;
size_t get_databytes() const;
const std::type_index& get_datatype() const;
const std::string& get_datatype_string() const; // "int8", "uint8", ..., "float32", "float64"
uint8_t get_datatype_size() const;
```
Information about the socket (sizes include all the frames).

```cpp
void allocate_buffer();
void deallocate_buffer();
```
Allocate (resp. release) the memory of an output socket. They throw on input
and forward sockets.

Below are some examples of valid and invalid socket bindings:

=== "Valid bindings"
    <figure markdown>
        ![Valid bindings](./assets/valid_bindings.svg#only-light){ width=750 }
        ![Valid bindings](./assets/valid_bindings_neg.svg#only-dark){ width=750 }
        <figcaption>Examples of **valid** socket bindings.</figcaption>
    </figure>

=== "Invalid bindings"
    <figure markdown>
        ![Invalid bindings](./assets/invalid_bindings.svg#only-light){ width=487 }
        ![Invalid bindings](./assets/invalid_bindings_neg.svg#only-dark){ width=487 }
        <figcaption>Examples of **invalid** socket bindings.</figcaption>
    </figure>

For invalid socket bindings, `StreamPU` will throw an exception at runtime.

```cpp
size_t unbind(Socket& s_out);
```
This function is used to disconnect sockets from each other. It returns the
position of the current socket in the `bound_sockets` of `s_out`.

```cpp
void reset();
```
Unbinds the socket from the socket it is bound to (`bound_socket`) and unbinds
all the sockets bound to it (`bound_sockets`). Called by the destructor.

!!! note
    `s_out` must be bound to the caller socket otherwise `StreamPU` will
    throw an exception.

## Standard `SIN`/`SOUT` Sockets versus `SFWD` Socket

Using a couple of `SIN`/`SOUT` sockets or a single `SFWD` socket can have an 
impact on the code behavior and on the performance of the application. The most 
important point is the impact on the socket `dataptr` attribute.
 
- In the case of `SIN`/`SOUT` sockets, the input and the output sockets have 
  their own `dataptr`. The `input` socket receives the pointer from its bound 
  socket and the `output` socket has its own allocated memory space, the data 
  received and computed by the task are written to the `output` memory space. 
  The initial data are not modified in this case, there are no *side effects*.
- In the case of a single `SFWD` socket, the socket receives its `dataptr` from 
  the bound socket like an `input`. But unlike in the `SIN`/`SOUT` case, the 
  computed data are written directly on the provided memory space, thus 
  overwriting it (and potentially losing important information), there are 
  *side effects*.
