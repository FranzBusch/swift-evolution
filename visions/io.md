# A Vision for I/O in Swift

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
* [Prohibiting synchronous I/O in asynchronous contexts](#prohibiting-synchronous-io-in-asynchronous-contexts)
* [Two surfaces with shared primitives](#two-surfaces-with-shared-primitives)
* [Streaming protocols](#streaming-protocols)
* [Integration with Swift Concurrency](#integration-with-swift-concurrency)
  * [The proactor protocol](#the-proactor-protocol)
  * [Resource-specific proactor protocols](#resource-specific-proactor-protocols)
  * [Stable buffer pointers](#stable-buffer-pointers)
  * [Discovering a proactor](#discovering-a-proactor)
    * [Overriding the default proactor](#overriding-the-default-proactor)
  * [Solving submission races](#solving-submission-races)
  * [Fusing the proactor and the executor](#fusing-the-proactor-and-the-executor)
  * [High-level types](#high-level-types)
  * [Clocks and deadlines](#clocks-and-deadlines)
* [Future directions](#future-directions)
  * [A linked-operation DSL](#a-linked-operation-dsl)
  * [Child-task-free multi-await](#child-task-free-multi-await)
  * [A `with` statement for scoped resources](#a-with-statement-for-scoped-resources)
  * [Static resource capability via an effects system](#static-resource-capability-via-an-effects-system)
* [Alternatives considered](#alternatives-considered)
  * [Eventing as a property of the executor](#eventing-as-a-property-of-the-executor)
  * [A single data-driven operation type](#a-single-data-driven-operation-type)
  * [One resource type with both sync and async methods](#one-resource-type-with-both-sync-and-async-methods)
  * [Unadorned names for the synchronous surface](#unadorned-names-for-the-synchronous-surface)
  * [Asynchronous proactor methods](#asynchronous-proactor-methods)
* [Prior art](#prior-art)
  * [Go](#go)
  * [Rust](#rust)
  * [C++](#c)
  * [.NET](#net)
  * [Grand Central Dispatch](#grand-central-dispatch)
  * [Java](#java)

## Introduction

Almost every non-trivial program does *input/output*. It reads and writes files,
talks over sockets, drives pipes and terminals, waits for timeouts, and waits on
child processes. Programs do this in two fundamentally different ways.
*Synchronously*, where a call blocks the current thread until the work is done,
and *asynchronously*, where the work is submitted and the caller is free to do
other things until it completes. Swift needs a first-class story for both, and
today it has a good story for neither. There is no coherent, portable set of I/O
APIs that feels like it belongs to the language.

This vision proposes a path forward, covering both synchronous and asynchronous
I/O and building them on shared primitives, so that the two surfaces differ
mostly by the `await`. The vision spans all of Swift's supported platforms:
Apple platforms, Linux, Android, FreeBSD, Windows, WebAssembly, and Embedded
Swift. I/O is fundamental everywhere Swift runs, and the model here is designed
to work everywhere, even where the underlying mechanism differs dramatically.

## Motivation

The absence of a unified story is a constant source of developer friction.
Synchronous I/O in Swift today means dropping down to the C library and
wrangling file descriptors, raw buffers, and `errno` by hand, with none of the
safety, portability, or ergonomics the rest of the language offers. Asynchronous
I/O means reaching for a framework such as `SwiftNIO` or `Dispatch` that bring
their own event loop, their own thread pool, and their own currency types,
running *beside* the concurrency runtime rather than within it. While this works
it doesn't feel like it belongs to Swift, and the choice between them is usually
made once, early, and is expensive to reverse.

Running an I/O layer beside the concurrency runtime also carries a cost. Each
framework brings its own eventing loop, so threads multiply, wakeups land at the
wrong priority because a generic reactor thread has no idea which task is
waiting, and libraries built on one framework do not interoperate with another.
The ecosystem fragments along whichever I/O layer a package happened to pick,
which is what a language-level story prevents.

That friction is felt every day during development and during production usage.
Because most APIs offer only synchronous I/O, blocking calls end up buried a few
layers down inside `async` code, where they block one of the limited concurrency
threads. A single synchronous file read backed by a slow networked filesystem,
reached transitively from an async request handler, is enough to spike a
service's latency or stall a daemon. A complete I/O story therefore has to do
two things:
- Make the asynchronous interfaces the natural one.
- Make it hard to accidentally block a concurrency thread with the synchronous
  one.

## Goals

The vision sets out goals that shape the concrete solutions:

* **Synchronous and asynchronous I/O, one set of types.** Both surfaces are
  first-class. They share the same currency types, the platform handles,
  buffers, socket addresses, open options, and errors, and differ only by the
  `await`.
* **Hard to block a concurrency thread by accident.** Reaching a
  potentially-blocking synchronous operation from an `async` context should be a
  diagnostic, rather than a latent production failure.
* **I/O can run on the executor.** The executor that schedules a task's jobs can
  also wait for that task's I/O, which allows for maximum performance where the
  two are fused.
* **One model, every platform.** The programming model is identical on every
  platform, even where the implementation is entirely different, for example an
  `io_uring` submission ring versus a thread pool draining blocking syscalls.
* **Progressive disclosure by audience.** Application and library authors use
  high level currency types and compose on a small set of streaming protocols.
  Only runtime and backend authors implement executors and operations.
* **Integrated cancellation and priority.** Cancelling a task cancels the
  *actual* in-flight operation and releases its resources, and escalating a
  task's priority re-prioritises its pending I/O. The model carries the
  cancellation and the priority all the way through to the operation rather than
  stopping at the suspension.
* **Extensible and executor-agnostic.** New kinds of I/O, or a whole new
  platform, can be added without changing the core primitives.

## Prohibiting synchronous I/O in asynchronous contexts

Swift Concurrency expects that tasks always make forward progress to allow
executors to run work on a small, fixed pool of threads, potentially sized to
the core count. A synchronous, potentially-blocking I/O call breaks that
contract: the thread sits in a syscall making no progress and cannot be
reclaimed. Blocking one thread mostly means you have lost a fraction of the
machine's concurrency. Blocking all of them results in stalls where nothing
makes any progress.

On the surface this seems like an easy problem to fix: just don't call blocking
methods from asynchronous methods; however, the blocking call is almost never at
the call site but somewhere buried deep inside the call stack. Those calls are
often doing trivial work such as reading a config from a slow mount, a log
flush, or a synchronous DNS lookup buried a few layers down a library you depend
on. Those calls are almost impossible to find during code review, they pass
testing on fast local disks, and only become noticeable during live workloads.

There are four ways to prohibit asynchronous contexts from calling such blocking
methods, and they trade off reliability against cost:

* **A new function color "noasync":** Making the existing `noasync` availability
  attribute viral so that any method calling a `noasync` method must also be
  marked as `noasync`. The cost of such a new function color is high, requiring
  the new effect to be propagated through closures, generics and protocols.
* **A runtime trap:** The blocking operations detect at runtime that they are
  running in an asynchronous context and trap. This needs no language changes,
  works today, and catches the real misuse no matter how deep it is buried. The
  cost is that it adds a small check per call.
* **Linting:** A separate tool flags synchronous I/O reachable from async code.
  It is better than nothing, but it is not part of the compiler, it is opt-in,
  and it fails at diagnosing this across module boundaries, dynamic dispatch,
  closures, and protocol witnesses.
* **By convention:** Rely on a visible type split, `File` versus `AsyncFile`,
  plus documentation. Naming makes the choice explicit at the call site, but it
  enforces nothing and relies on the discipline of every developer.

While a new function color is the most correct, it has such wide-reaching impact
on the language that the **runtime trap** is the best answer right now, trading
off language complexity against a small runtime cost.

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

## Two surfaces with shared primitives

Both the synchronous and asynchronous surface are built from one set of types,
so moving between them is easy.

At the bottom layer is a set of currency types that both surfaces build upon,
such as:
- A `PlatformHandle` that abstracts a POSIX file descriptor or a Windows
  `HANDLE`/`SOCKET`
- The span family (`RawSpan`, `MutableRawSpan`, `OutputRawSpan`, `InputRawSpan`,
  and a multi-span for vectored I/O)
- A `SocketAddress` for socket operations
- An `OpenOptions` for file opening operations
- An `IOError` that surfaces portable, well-known error cases with the raw
  platform code available as an escape hatch, plus a `cancelled` case.

On top of the currency types sit the resource types most programs use: files,
TCP and UDP sockets and listeners, pipes and terminals, and subprocesses. Each
comes in a synchronous and an asynchronous form.

Following the convention set by `Sequence` and `AsyncSequence`, the asynchronous
type carries the `Async` prefix and the synchronous one takes the plain name:
`File` and `AsyncFile`, `TCPConnection` and `AsyncTCPConnection`, `UDPSocket`
and `AsyncUDPSocket`, and so on. The expectation is that most users reach for
the asynchronous form.

Reading a file asynchronously is then just:

```swift
try await AsyncFile.open(at: path, options: .read) { file in
  try await file.read(into: &buffer)
}
```

and the synchronous version is the same shape with the `Async` and `await`s
removed:

```swift
try File.open(at: path, options: .read) { file in
  try file.read(into: &buffer)
}
```

Similarly for TCP sockets:

```swift
try await AsyncTCPConnection.connect(to: address) { connection in
  try await connection.write(from: &request)
  try await connection.read(into: &response)
}
```

and again the synchronous version:

```swift
try TCPConnection.connect(to: address) { connection in
  try connection.write(from: &request)
  try connection.read(into: &response)
}
```

## Streaming protocols

Most I/O is not a single read but a *stream* of bytes or elements that compose
with each other: piping one resource into another, wrapping a stream in a
decoder or a decompressor, and nesting those wrappers. For that to compose
across the ecosystem, streaming needs a small set of currency protocols that
every resource speaks. Without them the ecosystem fragments the same way it does
where different packages define their own stream abstraction protocols.

The protocols fall out of a 2×2×2 of **sync/async × direction × buffer
ownership**:
- synchronous versus asynchronous
- read versus write
- and whether the caller or the stream owns the buffer

In the caller-owned forms a reader fills the caller's span and a writer drains
it. In the stream-owned forms the stream hands the caller a mutating span to
drain or fill, which unlocks zero-copy paths. They come in synchronous and
asynchronous forms and they are element-generic. They are intended to evolve
`AsyncSequence`, keeping its ergonomics while solving some existing issues
such as the lack of a write-side, bulk-reading, support for `~Copyable` types
and more.

The high-level types simply conform, so copying a file into a socket is a single
pipe:

```swift
try await AsyncFile.open(at: path, options: .read) { file in
  try await AsyncTCPConnection.connect(to: address) { connection in
    try await file.pipe(into: &connection)
  }
}
```

The synchronous protocols mirror this:

```swift
try File.open(at: path, options: .read) { file in
  try TCPConnection.connect(to: address) { connection in
    try file.pipe(into: &connection)
  }
}
```

Sometimes the caller would rather the stream own the buffer. A buffered reader
lends a mutating view of its own bytes, so a chunk is consumed in place with no
copy into a caller buffer, and a buffered writer lends its buffer to fill
directly:

```swift
for mutating try await chunk in reader {
  handle(&chunk) // a mutating view into the reader's own buffer
}

var out = try await writer.next() // a mutable view into the writer's buffer
encode(message, into: &out)
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

A rough outline of the asynchronous forms is shown below. The synchronous forms
mirror them with the `await` removed. They are element-generic to cater to both
byte-based and arbitrary-element-based use-cases. The caller-owned forms take a
span the caller owns. The stream-owned forms lend a view of the stream's own
storage through a one-shot (`yields`) coroutine.

```swift
// Caller-owned buffer: the caller passes a span for the reader to fill or the
// writer to drain.
public protocol AsyncReader<Element, Failure>: ~Copyable, ~Escapable {
  associatedtype Element: ~Copyable
  associatedtype Failure: Error
  mutating func read(into buffer: inout OutputSpan<Element>) async throws(Failure)
}

public protocol AsyncWriter<Element, Failure>: ~Copyable, ~Escapable {
  associatedtype Element: ~Copyable
  associatedtype Failure: Error
  mutating func write(from buffer: inout InputSpan<Element>) async throws(Failure)
}

// Stream-owned buffer: the stream lends a view of its own storage via a one-shot
// coroutine, so a chunk is consumed or produced in place with no copy.
public protocol AsyncBufferedReader<Element, Failure>: ~Copyable, ~Escapable {
  associatedtype Element: ~Copyable
  associatedtype Failure: Error
  mutating func next() async throws(Failure) yields (inout InputSpan<Element>)
}

public protocol AsyncBufferedWriter<Element, Failure>: ~Copyable, ~Escapable {
  associatedtype Element: ~Copyable
  associatedtype Failure: Error
  mutating func next() async throws(Failure) yields (inout OutputSpan<Element>)
}
```

## Integration with Swift Concurrency

While the synchronous APIs are simple ergonomic wrappers around the platform's
syscalls, the asynchronous APIs need to integrate with Swift Concurrency. Swift
Concurrency already gave the language a model for *scheduling*: tasks are broken
into jobs, and executors decide when and where those jobs run. Asynchronous I/O,
though, has so far lived *beside* that runtime rather than within it: a separate
I/O thread or event loop that waits for I/O and then hands completions back
across a thread boundary.

Closing that gap starts with understanding how operating systems report I/O,
which comes in one of two shapes. *Readiness-based* systems tell you when a
descriptor is ready and you perform the syscall yourself, e.g. `select`, `poll`,
`epoll`, and `kqueue`. The component that waits for those signals and drives the
syscalls is commonly referred to as a **reactor**. In *completion-based* systems
you hand the system the whole operation and it tells you the result when it is
done, e.g. `io_uring`, `IOCP`, and overlapped I/O. That component is commonly
referred to as a **proactor**. Swift Concurrency is already completion-shaped
through its `async/await` and continuation model. A task submits work, suspends,
and is resumed with a result, making a proactor the natural fit. Readiness
mechanisms are easily respelled into a proactor API, by turning "read these
bytes" into "wait until readable, then read." Mapping readiness onto completion
is cheap whereas going the other way would give up the syscall batching a
completion interface allows.

The two shapes differ in who owns the buffer while the task is suspended, which
in turn shapes how cancellation works. On a readiness backend the syscall has
not run yet, so no buffer is shared with the kernel. Cancelling can resume the
waiting continuation immediately and simply drop the interest. On a completion
backend the kernel may be reading into or writing from the caller's buffer for
as long as the task is suspended, so the operation cannot just be abandoned. The
proactor has to submit a real kernel cancellation and keep the buffer alive
until the kernel confirms the operation completed or was cancelled. Some
backends cannot cancel an in-flight operation at all, so they just have to wait
until the operation completes.

Intuitively one wants to fold the proactor into the executor, so the object that
runs a task's jobs also waits for its I/O, resulting in no extra threads, no
hops, no priority inversion. Many executors already own everything a proactor
needs, whereas a standalone proactor has to duplicate all of that and coordinate
across a thread boundary for every completion. That makes fusing the two the
right *default* for maximum performance. But the fusing should be optional, not
required, because plenty of programs want the two roles apart. A test harness
might swap in an in-memory proactor to make I/O deterministic while its tasks
keep running on the ordinary executor. A server might route its socket I/O
through a single shared `io_uring` proactor for batched submission without
handing that proactor the whole process's scheduling. So this vision proposes to
treat the proactor and the executor as separate roles that *may* be fused for
maximum performance, rather than one thing that is always both.

The next sections introduce the different pieces to produce the overall story
for asynchronous I/O in Swift Concurrency.

### The proactor protocol

A proactor owns the *identity and control* of in-flight operations. It is
deliberately *not* an executor and never runs jobs, since that's the executor's
role. Every operation a proactor services shares one common lifecycle:

> **Submit, then complete or cancel, then deliver a typed result**, with
> priority carried throughout.

Cancelling an operation and escalating its priority apply to every operation
regardless of what it reads or writes, so they form a small, resource-agnostic
baseline: given a value that identifies one in-flight operation, a proactor can
make a best-effort attempt to cancel it or raise its priority.

```swift
public protocol Proactor: AnyObject {
  func cancel(_ registration: OperationRegistration)
  func escalatePriority(
    of registration: OperationRegistration,
    to newPriority: TaskPriority
  )
}

// An opaque, stable, non-reused identity for one in-flight operation.
public struct OperationRegistration: Sendable, Hashable {
  public var id: UInt64
}
```

### Resource-specific proactor protocols

A resource family such as files, sockets, clocks, or processes is a protocol that
*refines* `Proactor` and adds that family's operations as concretely-typed
`submit` methods. Each takes a `Continuation` carrying that operation's result
and error type, and returns an `OperationRegistration` synchronously so the
caller can wire up cancellation and escalation. Refining `Proactor` per resource
makes this model extensible, as packages or platforms can define their own
resource-specific proactor protocol that concrete proactor implementations can
conform to.

```swift
public protocol FileProactor: Proactor {
  func submitOpen(
    _ continuation: consuming Continuation<PlatformHandle, IOError>,
    at path: FilePath,
    options: OpenOptions
  ) -> OperationRegistration

  func submitRead(
    _ continuation: consuming Continuation<Int, IOError>,
    handle: PlatformHandle,
    into buffer: inout OutputRawSpan
  ) -> OperationRegistration

  // submitWrite, submitClose, submitSync, ...
}
```

The standard library is expected to ship resource-specific proactor protocols
for the common resources: a `FileProactor`, socket and listener proactors, a
pipe proactor, a clock proactor, and a process proactor, each refining
`Proactor` with that family's `submit` methods. Because a resource family is
just a protocol refining `Proactor`, a package or a platform can add a new kind
of I/O by defining its own refinement and vending a proactor that conforms to
it, without any change to the core primitives:

```swift
// A package that drives a bespoke device defines its own resource protocol.
public protocol CustomDeviceProactor: Proactor {
  func submitDeviceRead(
    _ continuation: consuming Continuation<Int, IOError>,
    device: CustomDevice,
    into buffer: inout OutputRawSpan
  ) -> OperationRegistration
}
```

### Stable buffer pointers

As noted above, in a completion-based backend, when a read or write is submitted,
the proactor hands the kernel a pointer into the caller's buffer, and the kernel
writes into that buffer directly at some point between submission and
completion, while the task is suspended. The pointer has to stay valid, and the
storage behind it has to stay in place, for the whole in-flight window rather
than only for the duration of the `submitRead` call.

The `OutputRawSpan` the caller passes owns that storage. The high-level type
borrows it `inout` for the entire `async` operation, so the borrow spans the
suspension and nothing else can move, mutate, or free the storage while the
operation is in flight. The proactor projects a stable pointer out of the span at
submission and holds it until it resumes the continuation:

```swift
func submitRead(
  _ continuation: consuming Continuation<Int, IOError>,
  handle: PlatformHandle,
  into buffer: inout OutputRawSpan
) -> OperationRegistration {
  let registration = nextRegistration()
  // A pointer into the caller's buffer that must stay valid until completion.
  let pointer = buffer.unsafeStablePointer
  // Hand the whole operation to the kernel. The kernel writes into the buffer
  // while the task is suspended in `awaiter.wait()`.
  submitToKernel(registration, handle: handle, into: pointer)
  return registration
}
```

The high-level `read` is what keeps that pointer valid. Its `buffer` parameter
is `inout`, so the borrow of the caller's storage lasts for the whole `async`
call, and it hands that same `buffer` to `submitRead` before suspending in
`awaiter.wait()`. Because the borrow outlives the suspension, the storage the
proactor's pointer refers to cannot move or be freed until the continuation
resumes and `read` returns:

```swift
mutating func read(into buffer: inout OutputRawSpan) async throws(IOError) -> Int {
  try await withContinuation(of: Int.self, throwing: IOError.self) { continuation, awaiter in
    // `buffer` is borrowed for the whole call, so the stable pointer the proactor
    // takes here stays valid until the await below resumes.
    let registration = proactor.submitRead(continuation, handle: handle, into: &buffer)
    return try await withTaskCancellationHandler {
      try await awaiter.wait()
    } onCancel: {
      proactor.cancel(registration)
    }
  }
}
```

### Discovering a proactor

How a resource finds the proactor that will service it is a scoped choice.
Proactors form a stack of preferences pushed for a dynamic scope, much like a
task executor preference, so different parts of a program can run their I/O on
different proactors, e.g. one task on an `epoll` proactor and another on
`io_uring`, independently of which executor either runs on. An operation resolves
its proactor by walking, in order: the pushed proactor stack from the innermost
scope outward, taking the first that supports its resource, then the current
executor, first the active serial executor, then the task executor preference,
when it is itself a proactor for the resource, and finally the default proactor.

```swift
// `withProactor` pushes a proactor as a preference for the dynamic extent of its body
try await withProactor(IOUringProactor()) {
  // Operations here use io_uring for the resources it supports.
  try await withProactor(EpollProactor()) {
    // Here epoll is used. A resource that epoll cannot service falls back
    // outward to io_uring, and finally to the default proactor.
  }
}
```

A resource resolves its proactor once, at creation, and stores it to ensure all
later operations are submitted to the same proactor it was created on.

```swift
public struct AsyncFile: ~Copyable, Sendable {
  let handle: PlatformHandle
  // Resolved once at `open` and stored
  let proactor: any FileProactor

  static func open<R: ~Copyable>(
    at path: FilePath,
    options: OpenOptions,
    _ body: (inout AsyncFile) async throws -> R
  ) async throws -> R {
    // Discover the FileProactor from the enclosing scope exactly once.
    let proactor = Task.currentProactor(conformingTo: FileProactor.self)

    let handle = try await withContinuation(of: PlatformHandle.self, throwing: IOError.self) { continuation, awaiter in
      let registration = proactor.submitOpen(continuation, at: path, options: options)
      return try await withTaskCancellationHandler {
        try await awaiter.wait()
      } onCancel: {
        proactor.cancel(registration)
      }
    }

    var file = AsyncFile(handle: handle, proactor: proactor)

    defer {
      try? await withTaskCancellationShield { try await file.close() }
    }

    return try await body(&file)
  }

  // A later write submits to the same stored proactor
  mutating func write(from buffer: inout InputRawSpan) async throws(IOError) -> Int {
    try await withContinuation(of: Int.self, throwing: IOError.self) { continuation, awaiter in
      let registration = proactor.submitWrite(continuation, handle: handle, from: &buffer)
      return try await withTaskCancellationHandler {
        try await awaiter.wait()
      } onCancel: {
        proactor.cancel(registration)
      }
    }
  }
}
```

#### Overriding the default proactor

Scoped preferences pick a proactor for part of a program. However, a program can
also replace the process-wide default. Defaults are installed before any task
runs by pointing a typealias at a factory:

```swift
struct MyProactorFactory: ProactorFactory {
  // Vends a proactor conforming to the resource-specific protocols it can service.
  static var defaultProactor: some Proactor { IOUringProactor() }
}

typealias DefaultProactorFactory = MyProactorFactory
```

Replacing the default proactor means taking responsibility for the resources it
must service: if it does not support a resource some dependency reaches, and
nothing else in scope does either, that operation traps.

### Solving submission races

The resource-specific proactor protocols take continuations that they resume
once an operation completes. Today continuations in Swift are created using
`await withContinuation { ... }` which couples the creation and awaiting of the
continuation in one method. The closure for the `withContinuation` method is
synchronous, which forces the setup of cancellation and priority escalation
handlers to happen *before* the continuation is created. This leads to various
race conditions such as cancellation happening before the continuation was
created. 

```swift
// The continuation is created and awaited inside the same synchronous closure, so
// the registration is "trapped" there. The cancellation handler has to wrap the
// await from the outside, where the registration is not yet in scope.
try await withTaskCancellationHandler {
  try await withCheckedContinuation { continuation in
    let registration = proactor.submitRead(continuation, handle: handle, into: &buffer)
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
// give to the proactor, and a `ContinuationAwaiter` the task keeps and awaits.
public nonisolated(nonsending) func withContinuation<Success: ~Copyable, Failure: Error>(
  of: Success.Type = Success.self,
  throwing: Failure.Type,
  _ body: (
    consuming Continuation<Success, Failure>,
    consuming ContinuationAwaiter<Success, Failure>
  ) async throws(Failure) -> Success
) async throws(Failure) -> Success
```

Because the body is `async`, the operation can be submitted, its registration
kept in scope, and the cancellation and escalation handlers installed *around*
the await:

```swift
try await withContinuation(of: Int.self, throwing: IOError.self) { continuation, awaiter in
  // The continuation is handed to the proactor before the task suspends.
  let registration = proactor.submitRead(continuation, handle: handle, into: &buffer)

  return try await withTaskPriorityEscalationHandler {
    try await withTaskCancellationHandler {
      try await awaiter.wait()
    } onCancel: {
      proactor.cancel(registration)
    }
  } onPriorityEscalated: { _, newPriority in
    proactor.escalatePriority(of: registration, to: newPriority)
  }
}
```

### Fusing the proactor and the executor

Today continuations offer multiple `resume` methods. Each of them puts the value
into the buffer of the suspended task and then enqueues the task to run on the
executor again. While this works, it means that every resumption always leads to
an additional enqueue. For a fused proactor and executor this is unnecessary
since they would rather donate their current thread to resume the task
synchronously.

Continuations therefore gain a synchronous, thread-donating resume alongside the
existing one:

```swift
extension Continuation {
  // Existing: store the value and enqueue the task on its executor (a hop).
  consuming func resume(with result: consuming Result<Success, Failure>)

  // New: donate the current (executor-owned) thread to run the resumed task
  // inline, re-establishing its isolation: no enqueue, no hop.
  consuming func resumeSynchronously(
    isolatedTo serialExecutor: UnownedSerialExecutor,
    taskExecutor: UnownedTaskExecutor,
    with result: consuming Result<Success, Failure>
  )
}
```

A fused proactor and executor drains completions on its own thread and, because
that thread is an executor thread, resumes each task inline instead of
enqueuing:

```swift
while running {
  for (continuation, result) in proactor.drainCompletedOperations() {
    continuation.resumeSynchronously(
      isolatedTo: self.asUnownedSerialExecutor(),
      taskExecutor: self.asUnownedTaskExecutor(),
      with: result
    )
  }
}
```

A standalone proactor, whose thread is not an executor thread, calls the
ordinary `resume(with:)` and takes the one hop back onto the task's executor.

### High-level types

Application and library authors are not expected to touch proactors or
continuations directly. As the `AsyncFile` above shows, each resource gets an
ergonomic high-level type, a `TCPConnection`, a `UDPSocket`, and so on, that
hides the discovery, storage, and continuation-and-handler dance behind ordinary
`async` methods, so a read is just `file.read(into:)`.

These high-level types conform to the streaming protocols, which lets them
compose. The `file.pipe(into: &connection)` shown earlier works because both
sides are an `AsyncReader`/`AsyncWriter`:

```swift
extension AsyncFile: AsyncReader, AsyncWriter {} // Element == UInt8, Failure == IOError
extension AsyncTCPConnection: AsyncReader, AsyncWriter {}
```

### Clocks and deadlines

Not every resource is handle-backed. A clock is the smallest proactor: it
services a single `sleep`, and because a sleep has no handle to store, it
resolves its proactor per call through the same discovery chain.

```swift
public protocol ContinuousClockProactor: Proactor {
  func submitSleep(
    _ continuation: consuming Continuation<Void, CancellationError>,
    until instant: ContinuousClock.Instant,
    tolerance: ContinuousClock.Duration?
  ) -> OperationRegistration
}
```

Clocks are not only used for sleeps but also for deadlines. A deadline is a
point in time where a certain scope needs to be cancelled. This can be
implemented by a clock proactor with a fire callback at that instant, and when
the callback fires it cancels the scope.

```swift
public protocol ContinuousClockProactor: Proactor {
  ...
  // A callback variant runs a bare callback at the instant.
  func submitSleep(
    _ callback: @escaping () -> Void,
    until instant: ContinuousClock.Instant,
    tolerance: ContinuousClock.Duration?
  ) -> OperationRegistration
}
```

`withDeadline` builds on the callback variant. It opens a cancellation scope,
schedules a bare callback for the instant that cancels that scope, and runs the
body. If the body finishes first the timer is dropped. If the instant is reached
first the callback cancels the scope, and that cancellation reaches whichever
proactor is servicing each in-flight operation.

```swift
public func withDeadline<Return>(
  _ instant: ContinuousClock.Instant,
  tolerance: ContinuousClock.Duration? = nil,
  _ body: () async throws -> Return
) async throws -> Return {
  let proactor = Task.currentProactor(conformingTo: ContinuousClockProactor.self)
  return try await withCancellationScope { scope in
    // A bare callback that cancels the scope when the instant is reached. No task
    // is parked on the timer.
    let registration = proactor.submitSleep(
      { scope.cancel() },
      until: instant,
      tolerance: tolerance
    )
    // Drop the timer if the body finishes before the instant is reached.
    defer { proactor.cancel(registration) }
    return try await body()
  }
}
```

## Future directions

### A linked-operation DSL

A declarative way to express fused chains, such as open-write-close, copy-range,
or accept-then-read, as a single submission on backends that support it, with
typed placeholders for descriptors that never surface to user space.

```swift
try await withLinkedOperations { chain in
  let file = chain.open(at: "/tmp/out", options: [.write, .create])
  chain.write(to: file, bytes)
  chain.close(file)
}
```

### Child-task-free multi-await

Awaiting several operations at once currently requires a task group with one
child task per pending operation. Because the split continuation already
separates submission from awaiting, the runtime could instead offer a first-wins
`select` over a variadic set of awaiters that returns whichever completes first.

```swift
// One-shot and heterogeneous: the first to complete wins.
switch await select(timer.awaiter, connection.readAwaiter) {
case .first: break                    // the timer fired
case .second(let bytes): use(bytes)   // the read completed
}
```

### A `with` statement for scoped resources

Asynchronous resources are handed out with scoped lifetimes rather than relying
on `deinit` for cleanup. An async-backed resource cannot do its real teardown in
`deinit` since a `deinit` cannot `await`, so cleanup has to be an explicit,
`await`-able step. The scoped `open` and `connect` closures express that.

Composing multiple resources leads to nesting:

```swift
try await AsyncFile.open(at: path, options: .read) { file in
  try await AsyncTCPConnection.connect(to: address) { connection in
    try await file.pipe(into: &connection)
  }
}
```

A first-class `with` statement could bind multiple scoped resources for the
remainder of its enclosing scope and run the resource's cleanup at scope exit,
without a nested closure:

```swift
with var file = try await AsyncFile.open(at: path, options: .read), var connection = try await AsyncTCPConnection.connect(to: address) {
  try await file.pipe(into: &connection)
}
```

## Alternatives considered

### Static resource capability via an effects system

An operation discovers its proactor at runtime and traps if nothing in scope
supports the resource. If the language ever gained an effects system, then this
could instead model "requires a `FileProactor` in scope" as an effect carried in
a function's signature, turning that runtime trap into a static guarantee. This
would not only solve the proactor discovery but also solve the problem
["prohibiting synchronous I/O in asynchronous
contexts"](#prohibiting-synchronous-io-in-asynchronous-contexts).

### Eventing as a property of the executor

The most direct design makes the executor itself the thing that waits for I/O,
so there is only ever one component and never a hop. We rejected making that the
*only* model. Welding eventing to the executor forecloses the configurations
that motivate the split: an in-memory proactor swapped in for deterministic
tests, a single shared `io_uring` proactor serving several executors, or a bare
I/O service that has no business scheduling arbitrary jobs. The proactor is
therefore a separate capability that *may* be fused with an executor.

### A single data-driven operation type

The resource protocols expose each operation as a concretely typed method, for
example `submitRead` and `submitWrite`. The alternative is a single data-driven
`submit(operation)` core, the shape of an `io_uring` SQE or a `uv_req_t`, where
the operation kind and its arguments are packed into one value. We chose the
typed methods: they buy static result typing, discovery by protocol conformance,
extensibility, and no dispatch on an operation kind. The cost is a larger
protocol surface, since a new operation is a new method every conforming backend
implements.

### One resource type with both sync and async methods

Rather than separate `File` and `AsyncFile`, a single type could carry both
surfaces and pick per call. We rejected it since an asynchronous resource has to
hold the proactor it was pinned to at creation while a synchronous one holds
nothing, so a unified type would carry an optional proactor and an ill-defined
answer for what a synchronous read does on an asynchronously-opened handle.
Separate types give each exactly the state it needs.

### Unadorned names for the synchronous surface

We could give the asynchronous surface the plain name (`File`) and qualify the
synchronous one (`SyncFile`), on the grounds that the asynchronous surface is
the common case and deserves the shorter name. We considered it but chose to
follow the precedent already set by `Sequence` and `AsyncSequence`, where it is
the asynchronous variant that carries the `Async` prefix. Matching that
convention keeps the surfaces predictable across the standard library.

### Asynchronous proactor methods

The proactor's `submit` methods could themselves be `async` and simply return
the result. We kept them continuation-taking and synchronous instead to avoid
having each proactor implement the continuation, cancellation and priority
escalation setup. More importantly, it makes further optimizations possible,
such as a child-task-free `select` and fused linked operations.

## Prior art

Other ecosystems have solved asynchronous I/O in ways worth contrasting, since
the choices in this vision are based on lessons from what those models get right
and where they fragment.

### Go

Go integrates I/O into the runtime. A goroutine that does a blocking-looking
read is parked by the runtime's network poller and resumed when the descriptor
is ready, and `io.Reader` and `io.Writer` are the universal streaming currency.
This is close in spirit to putting eventing on the executor, and the
reader/writer duo maps onto the streaming protocols here. The difference is that
Go hides the scheduler entirely. There is no notion of a user-provided executor,
so a program cannot choose or specialise how its I/O is serviced.

### Rust

Rust splits the world into an executor, which is an async runtime, and a
reactor, which is an `epoll` or `io_uring` wrapper, connected by futures and
wakers. Crucially, Rust standardises only the *waker* side of that seam and
leaves the reactor concrete and per-runtime: there is no `Reactor` trait, only
`Future::poll` and the `Waker` vtable of `clone` / `wake` / `wake_by_ref` /
`drop`. A leaf future stashes the `Waker` and returns `Pending`. The runtime's
own reactor such as `mio`, tokio's I/O driver, or `async-io`, later calls `wake`
to have the task polled again. That interface is runtime-agnostic, but minimal in
two consequential ways: the `wake` carries no result, so the value must be
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
`cancelled`.

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
onto a queue, using dispatch sources over `kqueue` and `DispatchIO`
channels. This vision keeps GCD's good idea of eventing delivered onto the
executor while making fusing optional and preserving the non-blocking
forward-progress contract GCD lacked.

### Java

Java's Project Loom makes blocking I/O on virtual threads cheap. The runtime
parks a virtual thread on a blocking call and resumes it on completion, so
ordinary synchronous-looking code scales to many concurrent operations. This
again validates runtime-integrated eventing, but with a different surface: Loom
keeps a synchronous, blocking API and moves the asynchrony underneath it,
whereas Swift already has `async`/`await` and structured concurrency. So this
vision expresses I/O as first-class asynchronous operations with typed results
and cancellation rather than hiding them behind a blocking facade.