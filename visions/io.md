# A Vision for Async I/O in Swift

> This document is a draft feature [vision
> document](https://forums.swift.org/t/the-role-of-vision-documents-in-swift-evolution/62101).
> It lays out goals and a direction for input/output in Swift for discussion; it
> is not a set of approved proposals. Everything here is subject to the normal
> Swift Evolution process, which may revise or reject any specific design that
> emerges from it.

## Table of contents

* [Introduction](#introduction)
* [Motivation](#motivation)
* [Goals](#goals)
  * [Asynchronous I/O that comes with the language](#asynchronous-io-that-comes-with-the-language)
  * [Composability of high level types through streams](#composability-of-high-level-types-through-streams)
  * [Bulk and vectored operations](#bulk-and-vectored-operations)
  * [Zero copy I/O](#zero-copy-io)
  * [One model that works across platforms](#one-model-that-works-across-platforms)
  * [Using the platform's best available backend](#using-the-platforms-best-available-backend)
  * [Dynamic configuration of the backend](#dynamic-configuration-of-the-backend)
  * [Cancellation and priority propagate to the I/O operation](#cancellation-and-priority-propagate-to-the-io-operation)
  * [I/O that can run on the executor](#io-that-can-run-on-the-executor)
  * [Extensible to new resources and platforms](#extensible-to-new-resources-and-platforms)
  * [Progressive disclosure across the layers](#progressive-disclosure-across-the-layers)
  * [Making accidental blocking detectable](#making-accidental-blocking-detectable)
* [Asynchronous I/O in practice](#asynchronous-io-in-practice)
* [Resource types](#resource-types)
* [Currency types](#currency-types)
* [Streaming protocols](#streaming-protocols)
* [Integration with Swift Concurrency](#integration-with-swift-concurrency)
  * [The operation scheduler protocol](#the-operation-scheduler-protocol)
  * [Resource-specific operation scheduler protocols](#resource-specific-operation-scheduler-protocols)
  * [Discovering an operation scheduler](#discovering-an-operation-scheduler)
    * [Overriding the default operation scheduler](#overriding-the-default-operation-scheduler)
  * [Solving submission races](#solving-submission-races)
  * [Child-task-free multi-await](#child-task-free-multi-await)
  * [Combining the operation scheduler and the executor](#combining-the-operation-scheduler-and-the-executor)
* [Coexisting with blocking code](#coexisting-with-blocking-code)
* [Alternatives considered](#alternatives-considered)
  * [Eventing as a property of the executor](#eventing-as-a-property-of-the-executor)
  * [A single data-driven operation type](#a-single-data-driven-operation-type)
  * [Asynchronous operation scheduler methods](#asynchronous-operation-scheduler-methods)
* [Prior art](#prior-art)
  * [Go](#go)
  * [Rust](#rust)
  * [C++](#c)
  * [.NET](#net)
  * [Grand Central Dispatch](#grand-central-dispatch)
  * [Java](#java)

## Introduction

Almost every non-trivial program does *input/output*. It reads and writes files,
talks over a network, drives pipes and terminals, waits for timeouts, and waits
on child processes. Programs do this in two fundamentally different ways.
*Synchronously*, where a call blocks the current thread until the work is done,
and *asynchronously*, where the work is submitted and the caller is free to do
other things until it completes. Asynchronous I/O is the right default for
Swift. It is the shape Swift Concurrency is already built for, where a task
submits work, suspends, and is resumed with a result. Yet Swift lacks a story
for it today. There is no coherent, portable set of asynchronous I/O APIs that
feel like they belong to the language.

This vision proposes a way forward for asynchronous I/O from high level types
down to the lower level integrations with platform interfaces. The vision spans
all of Swift's supported platforms: Apple platforms, Linux, Android, FreeBSD,
Windows, WebAssembly, and Embedded Swift. I/O is fundamental everywhere Swift
runs, and the model here is designed to work everywhere, even where the
underlying mechanism differs dramatically.

## Motivation

The absence of an asynchronous I/O story is a constant source of developer
friction. Asynchronous I/O in Swift today means reaching for a framework such as
`SwiftNIO` or `Dispatch` that bring their own event loop, their own thread pool,
and their own currency types, running *beside* the concurrency runtime rather
than within it. While this works it doesn't feel like it belongs to Swift, and
the choice between them is usually made once, early, and is expensive to
reverse.

Running an I/O layer beside the concurrency runtime also carries a cost. Each
framework brings its own eventing loop, so threads multiply, wakeups land at the
wrong priority because a generic reactor thread has no idea which task is
waiting, and libraries built on one framework do not interoperate with another.
The ecosystem fragments along whichever I/O layer a package happened to pick,
which is what a language-level story prevents.

Additionally, many APIs today offer only synchronous I/O, so blocking calls end
up buried a few layers down inside `async` code, where they block one of the
limited concurrency threads. A single synchronous file read backed by a slow
networked filesystem, reached transitively from an async request handler, is
enough to spike a service's latency or stall a daemon.

## Goals

The goals below run from what a developer uses every day for asynchronous I/O
down to what an I/O backend maintainer implements.

### Asynchronous I/O that comes with the language

Swift should ship asynchronous I/O for the resources nearly every program
touches: files and the file system, TCP and UDP sockets and their listeners,
Unix domain sockets, raw sockets, pipes, terminals, signals, and child
processes. These should come with the language's core libraries rather than as
packages because they are currency types. A file, a socket, or a buffer appears
in the signature of many APIs that perform I/O, so when several packages each
define their own, code written against one does not compose with code written
against another.

### Composability of high level types through streams

Most I/O is not a single read but a stream of bytes or elements that composes
with other streams: piping one resource into another, wrapping a stream in a
decoder or a decompressor, and nesting those wrappers. A small set of streaming
protocols that resources conform to makes a file, a socket, a pipe, and an
in-memory buffer interchangeable wherever code only needs bytes in or bytes out.
Similar to the resource APIs of the first goal, those streaming protocols should
come with the language's core libraries since they enable the composability
across the rest of the ecosystem.

### Bulk and vectored operations

I/O operations move entire buffers, sometimes multiple buffers, at a time. The
streaming protocols should therefore work in batches of many elements and even
batches of batches of many elements. This enables highly performant I/O by
reducing the number of suspensions and syscalls.

### Zero copy I/O

It should be possible to move bytes between the kernel and their final
destination without intermediate copies. Buffers are not always the caller's: a
stream may lend part of its own storage, and some I/O primitives pick the buffer
themselves and only report afterwards which one they used. Hence, the streaming
protocols and the resource APIs must not assume a single owner, and in
particular must not make copying through a caller-supplied buffer the only way
to move data. Where the whole path is known in advance, the best implementation
moves the bytes without them entering user space at all.

### One model that works across platforms

The programming model should be identical on every platform Swift supports, even
where the underlying I/O primitive is entirely different, for example an
`io_uring` submission ring on one platform and a thread pool draining blocking
syscalls on another. Likewise, the high level types are the same types no matter
which
backend services them. There is no `io_uring` file type and no `epoll` file
type. A library that reads a file therefore works with whichever backend the
application chose, and swapping that backend does not require changing the
library.

### Using the platform's best available backend

The default implementation should use the most efficient mechanism the target
platform offers, chosen at runtime. For example, on Linux that means `io_uring`
where the kernel provides it and `epoll` otherwise.

### Dynamic configuration of the backend

A program should be able to choose the backend itself, either by replacing the
process-wide default or by selecting one for part of the program. A test suite
can then swap in an in-memory backend to make I/O deterministic, and a server
can route its socket I/O through a single shared `io_uring` backend so that
submissions batch.

### Cancellation and priority propagate to the I/O operation

Cancelling a task should cancel the in-flight operation and release the
resources it holds, rather than only abandoning the suspension. Escalating a
task's priority should likewise re-prioritize the I/O that task is waiting on.

Between submission and completion the kernel may be writing into the caller's
buffer, so the operation cannot be walked away from while that buffer is still
live, and some mechanisms cannot cancel an in-flight operation at all, so it can
only be waited out.

### I/O that can run on the executor

An executor already runs a task's jobs and should also be able to wait for that
task's I/O. This removes extra threads, hops across a thread boundary on every
completion, and makes dealing with priority propagation and escalation easier.
Importantly, it should be possible for executors to offer I/O capabilities
for maximum performance, but it must not be required.

### Extensible to new resources and platforms

A new kind of I/O, or a whole new platform, should be expressible without
changing the core primitives. A package driving a bespoke device, or a platform
whose eventing mechanism is fundamentally different, should be able to plug into
the same model and reuse the same streaming protocols and high level types.

### Progressive disclosure across the layers

Following the principles of progressive disclosure, most programs should only
ever need the high level resource types and the streaming protocols: opening a
file, reading and writing, piping one resource into another, and iterating a
decoded stream. Reaching for those should require no setup, no choice of backend
and no knowledge of how buffers are owned. Expert users can peel away the
layers, taking control of buffers to avoid a copy, choosing which backend
services a scope, or implementing an operation scheduler for specialized
use-cases.

### Making accidental blocking detectable

There are always going to be methods that make blocking calls such as existing
APIs or when calling C libraries. Those calls cannot all be removed, and they
are hard to find by inspection since they sit behind module boundaries, dynamic
dispatch, and protocol witnesses. Reaching a blocking call from an asynchronous
context should therefore be diagnosable, so that it guides a developer towards
the asynchronous APIs during development rather than surfacing as a latent
production failure.

## Asynchronous I/O in practice

This section shows what the expected experience is when using the asynchronous
I/O interfaces. Opening a file and reading from it:

```swift
var file = try await File.open(at: path, options: .read)
defer {
  try? await file.close()
}
try await file.read(into: &buffer, at: offset)
```

Connecting a TCP socket and exchanging bytes:

```swift
var tcpSocket = try await TCPSocket.connect(to: address)
defer {
  try? await tcpSocket.close()
}

try await tcpSocket.write(from: &requestBuffer)
try await tcpSocket.read(into: &responseBuffer)
```

Because every resource conforms to the same streaming protocols, moving bytes
from one to another is simple:

```swift
var file = try await File.open(at: path, options: .read)
defer {
  try? await file.close()
}
var tcpSocket = try await TCPSocket.connect(to: address)
defer {
  try? await tcpSocket.close()
}

try await file.pipe(into: &tcpSocket)
```

## Resource types

The core Swift libraries should cover types for the resource families below
since they are the ones nearly every program uses.

- Clocks: Sleeping until an instant.
- Files and the file system: Opening, reading and writing at an offset,
  truncating, syncing, and closing, along with the operations on the file system
  itself: listing a directory, creating and removing entries, moving them, and
  querying metadata.
- Pipes and terminals: Creating a pipe pair and reading from and writing to
  either end. For terminals, reading and writing along with querying and setting
  the terminal mode and size.
- Child processes: Spawning, wiring up standard input, output, and error to
  pipes, and awaiting exit.
- Network sockets: TCP sockets and listeners, UDP sockets, Unix domain sockets,
  and raw sockets. Connecting, accepting, sending, receiving.
- Signals: Observing process signals such as `SIGTERM`, `SIGINT`, and `SIGHUP`,
  which programs use to shut down gracefully or reload configuration, and taking
  over their handling from the platform's default behavior.

## Currency types

The resource types above are not enough on their own. Every operation on them
needs additional types such as:

- Spans: `RawSpan`, `MutableRawSpan`, `OutputRawSpan`, `InputRawSpan` to
  represent the buffers.
- `FilePath` to refer to a location in the file system.
- `FileOpenOptions` to decide how a file is opened, such as the access mode,
  whether to create or truncate, and the permissions to create with.
- `SocketAddress` to refer to an endpoint across IPv4, IPv6, and Unix domain
  sockets.
- `IOError` to surface portable errors across platforms.

## Streaming protocols

Most I/O is not a single read but a *stream* of bytes or elements that compose
with each other: piping one resource into another, wrapping a stream in a
decoder or a decompressor, and nesting those wrappers. For that to compose
across the ecosystem, streaming needs a small set of currency protocols that
every resource conforms to.

The protocols fall out of a 2×2 of **direction × buffer ownership**:
- read versus write
- and whether the caller or the stream owns the buffer

In the caller-owned forms a reader fills the caller's span and a writer drains
it. In the stream-owned forms the stream hands the caller a mutating span to
drain or fill, so a chunk is consumed or produced in place. They are
element-generic, and they are intended to evolve `AsyncSequence`, keeping its
ergonomics while solving some existing issues such as the lack of a write-side,
bulk-reading, support for `~Copyable` types and more.

```swift
// Caller owned

// Reading the contents of a file at an offset into a buffer.
try await file.read(into: &buffer, at: offset)

// Writing the contents into a file at an offset.
try await file.write(from: &buffer, at: offset)

// Stream owned

// Iterating the contents of a file.
for consuming try await chunk in file {
  handle(&chunk)
}

// Write into a file at an offset
try await file.write(at: offset) { outputSpan in
  outputSpan.append(1)
}
```

A decoder or decompressor is just a stream wrapping another stream:

```swift
// Decompress on the way in: a reader wrapping a reader.
let body = GzipDecompressor(wrapping: reader)
try await body.pipe(into: &file)

// Compress on the way out: a writer wrapping a writer.
var sink = GzipCompressor(wrapping: writer)
try await file.pipe(into: &sink)
```

Because the protocols are element-generic, a codec can turn a byte stream into a
stream of typed messages the same way a decompressor turns bytes into bytes, so
a high-level API such as an RPC framework just iterates decoded messages:

```swift
let requests = ProtobufDecoder<HelloRequest>(wrapping: connection)
for try await request in requests {
  try await respond(to: request)
}
```

## Integration with Swift Concurrency

The high level types above have to be serviced by something. Swift Concurrency
already gave the language a model for *scheduling*: tasks are broken into jobs,
and executors decide when and where those jobs run. Asynchronous I/O, though,
has so far lived *beside* that runtime rather than within it: a separate I/O
thread or event loop that waits for I/O and then hands completions back across a
thread boundary.

Closing that gap starts with understanding how operating systems report I/O,
which comes in one of two shapes. *Readiness-based* systems tell you when a
descriptor is ready and you perform the syscall yourself, e.g. `select`, `poll`,
`epoll`, and `kqueue`. The component that waits for those signals and drives the
syscalls is commonly referred to as a **reactor**. In *completion-based* systems
you hand the system the whole operation and it tells you the result when it is
done, e.g. `io_uring`, `IOCP`, and overlapped I/O. That component is commonly
referred to as a **proactor**. Swift Concurrency is already completion-shaped
through its `async/await` and continuation model. A task submits work, suspends,
and is resumed with a result, making the completion model the natural fit. This
vision proposes to call the component that services I/O in that model an
**operation scheduler**. Readiness mechanisms are easily respelled into an
operation scheduler's API, by turning "read these bytes" into "wait until
readable, then read." Mapping readiness onto completion is cheap whereas going
the other way would give up the syscall batching a completion interface allows.

The two shapes differ in who performs the syscall and who owns the buffer while
the task is suspended, which in turn shapes how cancellation works. A readiness
backend first attempts the operation without blocking, on the task's own thread.
If the operation completes, nothing is registered and the task never suspends.
If it would block, the backend registers interest and suspends the task, and
when the event arrives it performs the syscall itself, on whichever thread the
event was delivered to. So the caller's buffer may be written from another
thread while the task is suspended. Similarly, on a completion backend
the kernel may be reading into or writing from the caller's buffer for as long
as the task is suspended. In both cases the operation scheduler has to either
submit a real kernel cancellation, or coordinate with the thread performing the
syscall, and keep the buffer alive until the operation is confirmed completed or
cancelled. Additionally, some backends cannot cancel an in-flight operation at
all, so they just have to wait until the operation completes.

Intuitively one wants to fold the operation scheduler into the executor, so the
object that runs a task's jobs also waits for its I/O, resulting in no extra
threads, no hops, no priority inversion. Many executors already own everything
an operation scheduler needs, whereas a standalone operation scheduler has to
duplicate all of that and coordinate across a thread boundary for every
completion. That makes combining the two the right *default* for maximum
performance. But the combination should be optional, not required, because many
use-cases want to control them independently. A test harness might swap in an
in-memory operation scheduler to make I/O deterministic while its tasks keep
running on the default executor. A server might route its socket I/O through a
single shared `io_uring` operation scheduler for batched submission without
handing that operation scheduler the whole process's scheduling. This vision
proposes to treat the operation scheduler and the executor as separate roles
that *may* be combined for maximum performance, rather than one thing that is
always both.

The next sections introduce the different pieces to produce the overall story
for asynchronous I/O integration in Swift Concurrency.

### The operation scheduler protocol

An operation scheduler owns the *identity and control* of in-flight operations.
It is deliberately *not* an executor and never runs jobs, since that's the
executor's role. Every operation an operation scheduler services shares one
common lifecycle:

> **Submit, then complete or cancel, then deliver a typed result**, with
> priority carried throughout.

Cancelling an operation and escalating its priority apply to every operation
regardless of what it reads or writes, so they form a small, resource-agnostic
baseline. In addition to this lifecycle, many operation schedulers need a small
amount of private state per in-flight operation, at a stable address. Some
examples are an `aiocb` for POSIX asynchronous I/O or an `OVERLAPPED` for
Windows overlapped I/O. The exact size differs per backend. The operation
scheduler therefore declares the state as an associated type, and the caller
allocates it on the task's stack for the duration of the operation.

```swift
public protocol OperationScheduler: AnyObject {
  // Per-operation state for this scheduler.
  associatedtype OperationState: ~Copyable

  func cancel(_ registration: OperationRegistration)
  func escalatePriority(
    of registration: OperationRegistration,
    to newPriority: TaskPriority
  )
}

// An opaque and stable identity for one in-flight operation.
public struct OperationRegistration: Sendable, Hashable { ... }
```

### Resource-specific operation scheduler protocols

A resource family such as clocks, files, sockets, or processes is modeled
through a protocol that *refines* `OperationScheduler` and adds that family's
operations as concretely-typed `submit` methods. Each takes a `Continuation`
carrying that operation's result and error type, the scheduler's per-operation
state, and returns an `OperationRegistration` synchronously so the caller can
wire up cancellation and escalation.

Returning from a `submit` method does not mean the operation is live in the
kernel. An operation scheduler is allowed to record the operation and defer the
syscall that starts it, which allows a backend to batch several operations into
a single syscall: an `io_uring` backend writes submission queue entries and
delays `io_uring_enter`, and a `kqueue` backend accumulates a changelist and
applies it in one `kevent`. The only guarantee `submit` makes is that the
operation has
been accepted and that the returned registration identifies it.

```swift
public protocol FileOperationScheduler: OperationScheduler {
  associatedtype FileHandle

  func submitOpen(
    _ continuation: consuming Continuation<FileHandle, IOError>,
    state: inout OutputSpan<OperationState>,
    at path: FilePath,
    options: FileOpenOptions
  ) -> OperationRegistration

  func submitRead(
    _ continuation: consuming Continuation<Int, IOError>,
    state: inout OutputSpan<OperationState>,
    handle: FileHandle,
    into buffer: inout OutputRawSpan,
    offset: Int64?
  ) -> OperationRegistration

  // submitWrite, submitClose, submitSync, ...
}
```

The core Swift libraries are expected to ship resource-specific operation
scheduler protocols for the common resources outlined in [Resource
types](#resource-types).

Because a resource family is just a protocol refining `OperationScheduler`, a
package or a platform can add a new kind of I/O by defining its own refinement
and vending an operation scheduler that conforms to it, without any change to
the core primitives:

```swift
// A package that drives a bespoke device defines its own resource protocol.
public protocol CustomDeviceOperationScheduler: OperationScheduler {
  func submitDeviceRead(
    _ continuation: consuming Continuation<Int, IOError>,
    state: inout OutputSpan<OperationState>,
    device: CustomDevice,
    into buffer: inout OutputRawSpan
  ) -> OperationRegistration
}
```

### Discovering an operation scheduler

How a resource type finds the operation scheduler that will service it is
dynamically discovered. Operation schedulers form a stack of preferences pushed
for a dynamic scope, much like a task executor preference, so different parts of
a program can run their I/O on different operation schedulers, e.g. one task on
an `epoll` operation scheduler and another on `io_uring`, independently of which
executor either runs on.

A resource resolves its operation scheduler once at creation by walking, in
order:

1. The pushed operation scheduler stack, from the innermost scope outward,
   taking the first operation scheduler that services the operation's resource.
2. The current executor, when it is itself an operation scheduler for that
   resource, first the active serial executor and then the task executor
   preference.
3. The default operation scheduler.

```swift
// `withOperationScheduler` pushes an operation scheduler as a preference for the dynamic extent of its body
try await withOperationScheduler(IOUringOperationScheduler()) {
  // Operations here use io_uring for the resources it supports.
  try await withOperationScheduler(EpollOperationScheduler()) {
    // Here epoll is used. A resource that epoll cannot service falls back
    // outward to io_uring, and finally to the default operation scheduler.
  }
}
```

A resource type stores the resolved operation scheduler to ensure all later
operations are submitted to the same operation scheduler it was created on.

#### Overriding the default operation scheduler

Scoped preferences pick an operation scheduler for part of a program. However, a
program can also replace the process-wide default.

```swift
struct MyOperationSchedulerFactory: OperationSchedulerFactory {
  static var defaultOperationScheduler: IOUringOperationScheduler { IOUringOperationScheduler() }
}

typealias DefaultOperationSchedulerFactory = MyOperationSchedulerFactory
```

### Solving submission races

The resource-specific operation scheduler protocols take continuations that they
resume once an operation completes. Today continuations in Swift are created
using `await withContinuation { ... }` which couples the creation and awaiting
of the continuation in one method. The closure for the `withContinuation` method
is synchronous, which forces the setup of cancellation and priority escalation
handlers to happen *before* the continuation is created. This leads to various
race conditions such as cancellation happening before the continuation was
created.

```swift
// The continuation is created and awaited inside the same synchronous closure, so
// the registration is "trapped" there. The cancellation handler has to wrap the
// await from the outside, where the registration is not yet in scope.
try await withTaskCancellationHandler {
  try await withContinuation { continuation in
    let registration = operationScheduler.submitRead(continuation, handle: handle, into: &buffer)
    // `registration` cannot escape this closure.
  }
} onCancel: {
  // We cannot cancel the in-flight operation: `registration` is not visible here,
  // and the operation may even complete before this handler is installed or this
  // handler is called before the operation is submitted.
}
```

A split interface that separates creating the continuation from awaiting it lets
`async` code, such as setting up task cancellation handlers, run inside the body
of the continuation, which resolves the races above.

```swift
// The split primitive hands the body two halves: a `Continuation` resume half to
// give to the operation scheduler, and a `ContinuationAwaiter` the task keeps and awaits.
public nonisolated(nonsending) func withContinuation<Success: ~Copyable, Failure: Error>(
  of: Success.Type = Success.self,
  throwing: Failure.Type,
  _ body: (
    consuming Continuation<Success, Failure>,
    consuming ContinuationAwaiter<Success, Failure>
  ) async throws(Failure) -> Success
) async throws(Failure) -> Success
```

Because the body is `async`, the operation can be submitted and its registration
kept in scope. The cancellation and escalation handlers are then installed on
the await itself, so they can refer to that registration. Passing them to the
await also lets the runtime publish both handlers and suspend the task in a
single update to the task's state, rather than one update per handler followed
by a third to suspend:

```swift
try await withContinuation(of: Int.self, throwing: IOError.self) { continuation, awaiter in
  // The continuation is handed to the operation scheduler before the task
  // suspends, so the registration it returns is in scope for the handlers.
  let registration = operationScheduler.submitRead(
    continuation, handle: handle, into: &buffer, offset: offset
  )

  // Publishes both handlers and suspends in one step.
  return try await awaiter.wait(
    onCancel: { operationScheduler.cancel(registration) },
    onEscalate: { newPriority in
      operationScheduler.escalatePriority(of: registration, to: newPriority)
    }
  )
}
```

### Child-task-free multi-await

Awaiting several operations at once requires a task group with one child task
per pending operation. Because the split continuation already separates
submission from awaiting, the runtime could instead offer a first-wins `select`
over a variadic set of awaiters that returns whichever completes first.

```swift
// One-shot and heterogeneous: the first to complete wins.
switch await select(timer.awaiter, connection.readAwaiter) {
case .first: break                    // the timer fired
case .second(let bytes): use(bytes)   // the read completed
}
```

### Combining the operation scheduler and the executor

For maximum performance an operation scheduler and an executor might be combined
in the same object. Today continuations offer multiple `resume` methods. Each of
them puts the value into the buffer of the suspended task and then enqueues the
task to run on the executor again. While this works, it means that every
resumption always leads to an additional enqueue. For a combined operation
scheduler and executor this is unnecessary since they would rather donate their
current thread to resume the task synchronously.

New APIs on continuations that allow a thread-donating resume enable highly
performant combined executors and operation schedulers:

```swift
extension Continuation {
  consuming func resumeSynchronously(
    isolatedTo serialExecutor: UnownedSerialExecutor,
    taskExecutor: UnownedTaskExecutor,
    with result: consuming Result<Success, Failure>
  )
}
```

## Coexisting with blocking code

Swift Concurrency expects that tasks always make forward progress to allow
executors to run work on a small, fixed pool of threads, potentially sized to
the core count. A synchronous, potentially-blocking I/O call breaks that
contract: the thread sits in a syscall making no progress and cannot be
reclaimed. Blocking one thread mostly means you have lost a fraction of the
machine's concurrency. Blocking all of them results in stalls where nothing
makes any progress.

Because the APIs in this vision are asynchronous, the blocking calls a program
still makes come from elsewhere: C libraries, older Swift APIs, and code written
before concurrency existed. On the surface this seems like an easy problem to
fix: just don't call blocking methods from asynchronous methods; however, the
blocking call is almost never at the call site but somewhere buried deep inside
the call stack. Those calls are often doing trivial work such as reading a
config from a slow mount, a log flush, or a synchronous DNS lookup buried a few
layers down a library you depend on. Those calls are almost impossible to find
during code review, they pass testing on fast local disks, and only become
noticeable during live workloads.

There are four ways to keep asynchronous contexts from calling such blocking
methods, and they trade off reliability against cost:

* A new function color "noasync": Making the existing `noasync` availability
  attribute viral so that any method calling a `noasync` method must also be
  marked as `noasync`. The cost of such a new function color is high, requiring
  the new effect to be propagated through closures, generics and protocols.
* A runtime trap: The blocking operations detect at runtime that they are
  running in an asynchronous context and trap. This needs no language changes,
  and catches the real misuse no matter how deep it is buried. The
  cost is that it adds a small check per call.
* Linting: A separate tool flags synchronous I/O reachable from asynchronous
  code.
  It is better than nothing, but it is not part of the compiler, it is opt-in,
  and it fails at diagnosing this across module boundaries, dynamic dispatch,
  closures, and protocol witnesses.
* By convention: Rely on documentation and on the naming of the blocking
  APIs. That makes the choice visible at the call site, but it enforces nothing
  and relies on the discipline of every developer.

While a new function color is the most correct, it has such wide-reaching impact
on the language that the runtime trap is the best answer right now, trading
off language complexity against a small runtime cost. The check itself can be
generated with a new `@blocking` macro that marks an operation that may block
and rewrites its body to ask the runtime first:

```swift
@blocking
public func read(into buffer: inout OutputRawSpan) throws(IOError) -> Int {
  try self.readSyscall(into: &buffer)
}
```

The trap fires when a blocking operation actually runs in an asynchronous
context, so it catches misuse in practice without being a static guarantee.
There are also legitimate reasons to make a blocking call from an asynchronous
context such as a program that has taken responsibility for not starving its
executor, or code running on a dedicated executor to isolate blocking calls. To
enable such use-cases we need a way to dynamically opt-out of the runtime traps:

```swift
withBlockingAllowed {
  // synchronous, potentially-blocking I/O that would otherwise trap
  try file.read(into: &buffer)
}
```

## Alternatives considered

### Eventing as a property of the executor

The most direct design makes the executor itself the thing that waits for I/O,
so there is only ever one component and never a hop. We rejected making that the
*only* model. Welding eventing to the executor forecloses the configurations
that motivate the split: an in-memory operation scheduler swapped in for
deterministic tests, a single shared `io_uring` operation scheduler serving
several executors, or a bare I/O service that has no business scheduling
arbitrary jobs. The operation scheduler is therefore a separate capability that
*may* be combined with an executor.

### A single data-driven operation type

The resource protocols expose each operation as a concretely typed method, for
example `submitRead` and `submitWrite`. The alternative is a single data-driven
`submit(operation)` core, the shape of an `io_uring` SQE or a `uv_req_t`, where
the operation kind and its arguments are packed into one value. We chose the
typed methods: they buy static result typing, discovery by protocol conformance,
extensibility, and no dispatch on an operation kind. The cost is a larger
protocol surface, since a new operation is a new method every conforming backend
implements.

### Asynchronous operation scheduler methods

The operation scheduler's `submit` methods could themselves be `async` and
simply return the result. An `async` method would also remove the need for a
separate per-operation state primitive, since the state could be a local
variable in the operation scheduler's own frame, sized exactly for that
backend.

We kept them continuation-taking and synchronous for two reasons. An `async`
method cannot be started without being awaited, so a task can only ever have one
operation in flight per suspension, which makes [child-task-free
multi-await](#child-task-free-multi-await) impossible. Second, it keeps the
suspension,
the cancellation handler and the escalation handler in one place rather than
requiring every backend to reimplement them.

## Prior art

Other ecosystems have solved asynchronous I/O in ways worth contrasting, since
the choices in this vision are based on lessons from those models.

### Go

Go integrates I/O into the runtime. A goroutine that does a blocking-looking
read is parked by the runtime's network poller and resumed when the descriptor
is ready, and `io.Reader` and `io.Writer` are the universal streaming currency.
This is close in spirit to putting eventing on the executor, and the
reader/writer duo maps onto the streaming protocols here. The difference is that
Go hides the scheduler entirely. There is no notion of a user-provided executor,
so a program cannot choose or specialize how its I/O is serviced.

### Rust

Rust splits the world into an executor, which is an async runtime, and a
reactor, which is an `epoll` or `io_uring` wrapper, connected by futures and
wakers. Crucially, Rust standardi`es only the *waker* side and leaves the
reactor concrete and per-runtime: there is no `Reactor` trait, only
`Future::poll` and the `Waker` vtable of `clone` / `wake` / `wake_by_ref` /
`drop`. A leaf future stashes the `Waker` and returns `Pending`. The runtime's
own reactor such as `mio`, tokio's I/O driver, or `async-io`, later calls `wake`
to have the task polled again. That interface is runtime-agnostic, but minimal
in two consequential ways: the `wake` carries no result, so the value must be
recovered by re-polling, and its only option is to *reschedule*. On top of that
the ecosystem fragments along concrete runtimes for two further reasons:

- Runtime-specific *leaf* types: A library that uses `tokio::spawn`,
`tokio::time::sleep`, or `tokio::net::TcpStream`, or the
`AsyncRead`/`AsyncWrite` traits is welded to Tokio, and does not interoperate
with other runtimes such as the `futures` crate without shims.
- The completion-model buffer problem: Because a Rust future can be dropped or
leaked at any await and destructors cannot be relied upon, completion-based I/O
could not keep the borrowed-buffer `AsyncRead`/`AsyncWrite` model, and
`tokio-uring`, `monoio`, and `glommio` each retreated to a different,
incompatible owned-buffer API of `(result, buffer)` pairs, splitting the
ecosystem again along the completion axis.

### C++

C++ P2300 `std::execution` models async work as *senders*. Connecting a sender
to a receiver produces an operation-state, and `start` begins it, with results
flowing back through typed `set_value`, `set_error`, and `set_stopped` channels,
and cancellation carried by a `stop_token`. This is the same split this vision's
[split continuation](#solving-submission-races) makes, where you create the
suspension object first, begin it, then let a typed result flow back. The three
completion channels line up with typed success, typed `IOError`, and
`IOError.cancelled`.

### .NET

.NET is the closest sibling to the low-level primitive here. Its
`IValueTaskSource` is a reusable, poolable backing store that splits creating an
awaitable from awaiting it and carries a typed result, which is exactly the
[split continuation](#solving-submission-races)'s shape.

Like this vision, .NET presents a completion-shaped model regardless of backend:
on Windows its async socket and file I/O map directly onto I/O completion ports
(IOCP), while on Unix it wraps a readiness-based `epoll` engine behind the same
completion-style `Task`/`ValueTask` surface, the same "respell readiness as
completion" approach taken here.

### Grand Central Dispatch

Grand Central Dispatch is Apple's prior art for delivering I/O as work items
onto a queue, using dispatch sources over `kqueue` and `DispatchIO` channels.
This vision keeps GCD's good idea of eventing delivered onto the executor while
making that combination optional and preserving the non-blocking
forward-progress contract GCD lacked.

### Java

Java's Project Loom makes blocking I/O on virtual threads cheap. The runtime
parks a virtual thread on a blocking call and resumes it on completion, so
ordinary synchronous-looking code scales to many concurrent operations. This
again validates runtime-integrated eventing, but with a different surface: Loom
keeps a synchronous, blocking API and moves the asynchrony underneath it,
whereas Swift already has `async`/`await` and structured concurrency.
