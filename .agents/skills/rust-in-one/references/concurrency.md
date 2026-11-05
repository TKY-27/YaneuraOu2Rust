# Concurrency, async, and services

Read this before writing a network service, async code, or nontrivial multithreaded code.

## Choose the model

- **Synchronous code**: CLIs and batch tools that read files, plus most libraries. Simplest to test and reason about.
- **Threads**: a few long-lived workers or blocking work. Use `std::thread::scope` to borrow data without `Arc`, and `rayon` for data-parallel loops over collections. Accumulate results per thread and merge them, rather than sharing a locked accumulator.
- **Async (`tokio`)**: many concurrent sockets, servers, clients issuing parallel requests, or timers. Async does not make CPU work faster.

Make runtime requirements explicit. Reusable async APIs normally use the caller's runtime; a blocking facade may own a runtime when its lifecycle and calling restrictions are deliberate. Tokio I/O requires Tokio runtime facilities even when an API returns a generic future. Do not nest `Runtime::block_on` or `Handle::block_on` inside an async execution context: these calls panic there.

## Async rules

- **Never block the executor.** `std::fs`, `std::thread::sleep`, synchronous HTTP or database clients, heavy computation, and long-held `std::sync::Mutex` locks stall every task on that worker thread. Use async equivalents, or move finite blocking work to `tokio::task::spawn_blocking` (a CPU-heavy batch can go to `rayon` instead). Bound concurrent CPU jobs explicitly: Tokio's blocking pool has a high default thread limit suited for blocking I/O, not CPU saturation. Started blocking tasks cannot be aborted; cancellation and shutdown need cooperation from the work itself. See [spawn_blocking](https://docs.rs/tokio/latest/tokio/task/fn.spawn_blocking.html).
- **Locks and `.await`.** A `std::sync::Mutex` is fine for short critical sections, but never hold its guard across `.await`: scope the guard in a block. An async mutex can serialize access to an I/O resource across `.await`; prefer a dedicated owner task when it simplifies cancellation and backpressure. See [Tokio's mutex guidance](https://docs.rs/tokio/latest/tokio/sync/struct.Mutex.html).
- **Backpressure and limits.** Use bounded channels (`mpsc::channel(n)`). Limit fan-out with `Semaphore`, `JoinSet` plus a cap, or `buffer_unordered(n)`. Unbounded channels, spawning, and queues can exhaust memory when producers outpace consumers.
- **Timeouts and size limits at each I/O boundary.** A read timeout does not bound a response write to a peer that stops reading. Bound reads, writes, and relevant whole-operation deadlines; ensure cancellation or timeout actually releases worker slots and resources. Test peers that stop reading responses as well as clients that send requests slowly or stall. Configure outbound connection, read, and overall request deadlines, ensuring that stalled request writes are also bounded. `reqwest` has no overall request timeout by default. On servers, `tower_http::timeout::TimeoutLayer` bounds response creation, while body transfer and transport I/O need their own deadlines. For body-size limits, `axum::extract::DefaultBodyLimit` applies only to extractors that use it; `RequestBodyLimitLayer` applies regardless of the extractor.
- **Cancellation.** A future can be dropped at any `.await`, for example when a losing `select!` branch or a timeout drops it. A client disconnect does not automatically cancel all server work; trace the framework and ownership behavior. Keep state consistent across await points. In `tokio::select!` loops, use only cancellation-safe operations, as stated in each method's "Cancel safety" docs (`mpsc::Receiver::recv` is cancellation safe; `AsyncReadExt::read_exact` is not), or keep the in-progress future outside the loop and poll it by reference.
- **Task failures.** A `JoinHandle` exposes task completion, returned errors, and panics; a panic hook may also emit a message. Join tasks (`JoinSet`), propagate or log their errors, and do not leak detached tasks that outlive shutdown.
- **Graceful shutdown.** Handle `tokio::signal::ctrl_c()` and, on Unix, SIGTERM (`tokio::signal::unix`). Stop accepting work with `axum::serve(...).with_graceful_shutdown(...)` and enforce a separate deadline for in-flight work. Define what happens to work still running at that deadline, propagate cancellation to background tasks (`tokio_util::sync::CancellationToken`), then flush logs and state.

## Shared state

- Prefer ownership transfer (channels, one task owning a resource and serving requests) over shared mutable state.
- For a service's shared state, use `Arc<AppState>` with fine-grained interior locks or atomics for the parts that change. Keep critical sections short and acquire multiple locks in one consistent order. In both synchronous and async code, release application-state locks before unrelated I/O; copy or extract the needed data while locked.
- Use atomics only for simple counters and flags, unless you can state the ordering argument. `Relaxed` is fine for independent statistics counters. Publishing data to another thread needs `Release`/`Acquire` or a lock. Independently review custom lock-free structures; use `loom` tests where the relevant concurrency can be modeled.
- `Rc` is neither `Send` nor `Sync`; `Cell<T>` and `RefCell<T>` are not `Sync` but are `Send` when `T: Send`. Do not add an unjustified `unsafe impl Send` or `unsafe impl Sync` just to silence compiler errors.

## Service checklist

- Read configuration from flags, environment variables, or files and validate it at startup. On invalid configuration, exit with a clear error message and a nonzero status rather than panicking.
- Structured logs (`tracing`) include request identifiers and avoid secrets.
- Health and readiness endpoints reflect dependency state where the deployment uses them.
- Input validation, size limits, and timeouts apply on every route that accepts external input, including response writes and streaming bodies. A full worker pool must recover after stalled peers time out.
- Tests start the real server on `127.0.0.1:0` and issue real requests, including malformed ones and shutdown during a request.

## Testing concurrent code

- `#[tokio::test]` uses a current-thread runtime by default. Use `flavor = "multi_thread"` when the behavior under test depends on parallelism.
- For isolated timer logic use `start_paused = true` and `tokio::time::advance` with Tokio's `test-util` feature and a current-thread runtime. Use real deadlines for real socket/process tests.
- Run concurrency tests repeatedly; a single pass proves little. Use `loom` for custom synchronization primitives, and ThreadSanitizer (nightly, `-Zsanitizer=thread`) where available.
