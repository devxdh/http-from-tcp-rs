# http-from-tcp-rs

A minimal multi-threaded HTTP/1.1 server built in Rust using standard library networking primitives (`std::net::TcpListener`, `std::net::TcpStream`) and a custom thread pool.

This is a learning project based on the final chapter of "The Rust Programming Language" book, designed to demonstrate raw TCP handling, HTTP response framing, and thread synchronization primitives in Rust.

## What the Project Does

1. Binds a TCP listener to `127.0.0.1:8080`.
2. Initializes a worker thread pool of fixed capacity (default: 4 threads).
3. Listens for incoming TCP connections in a loop.
4. Dispatches each connection to the thread pool as an executable task.
5. Parses the request line from the incoming raw TCP byte buffer:
   - `GET / HTTP/1.1` -> returns `200 OK` with contents of `hello.html`.
   - `GET /sleep HTTP/1.1` -> sleeps for 5 seconds to simulate high-latency work, then returns `200 OK` with `hello.html`.
   - Any other request -> returns `404 NOT FOUND` with `404.html`.
6. Formats an HTTP response with status line, `Content-Length` header, and body, writes it back to the stream, and flushes the socket.

The `/sleep` route demonstrates why concurrency is required: without multi-threading, a single slow request blocks every subsequent incoming connection.

## Multi-Threading Architecture

The project implements a bounded worker pool in `src/lib.rs` to handle requests concurrently without unbounded thread spawning.

### 1. Job Representation

```rust
type Job = Box<dyn FnOnce() + Send + 'static>;
```

- `FnOnce()`: Each job is an owned closure that runs once (handling one connection).
- `Send`: The closure can be transferred across thread boundaries safely.
- `'static`: The closure owns its data or only contains references valid for the entire program lifetime.

### 2. Work Distribution (`Arc<Mutex<Receiver<Job>>>`)

Rust standard library `std::sync::mpsc` provides a multi-producer, single-consumer channel. Because there are multiple worker threads needing to consume from the same channel:

- The `Receiver<Job>` is wrapped in a `Mutex` to synchronize access so only one worker reads from the channel at a time.
- The `Mutex<Receiver<Job>>` is wrapped in an `Arc` (atomic reference count) to share ownership across all workers.

```
                  +-------------------+
                  | ThreadPool::sender|
                  +---------+---------+
                            |
                     (mpsc::Sender)
                            |
                            v
             +------------------------------+
             | Arc<Mutex<mpsc::Receiver>>   |
             +--------------+---------------+
                            |
        +-------------------+-------------------+
        |                   |                   |
        v                   v                   v
+---------------+   +---------------+   +---------------+
| Worker 0      |   | Worker 1      |   | Worker N...   |
| loop { recv } |   | loop { recv } |   | loop { recv } |
+---------------+   +---------------+   +---------------+
```

### 3. Worker Execution Loop and Lock Granularity

Inside `Worker::new`:

```rust
loop {
    let message = receiver.lock().unwrap().recv();

    match message {
        Ok(job) => {
            println!("Worker {id} got a job; executing.");
            job();
        }
        Err(_) => {
            println!("Worker {id} disconnected; shutting down.");
            break;
        }
    }
}
```

Lock acquisition is scoped specifically to `receiver.lock().unwrap().recv()`. The temporary `MutexGuard` is dropped as soon as the `let message` statement completes. This ensures a worker does not hold the mutex while executing `job()`, allowing other idle workers to immediately pick up incoming requests.

### 4. Graceful Shutdown (`Drop` implementation)

`ThreadPool` implements `Drop` to ensure all active work completes and threads shut down deterministically:

1. `drop(self.sender.take())`: Drops the channel sender. When the sender is dropped, all remaining `receiver.recv()` calls fail with an `Err(_)`.
2. Workers receive `Err(_)`, break out of their processing loops, and exit their threads.
3. The pool iterates through workers, calls `worker.thread.take().unwrap().join().unwrap()`, waiting for each thread to finish.

## Project Structure

- `src/main.rs`: TCP listener loop, request parsing, routing, and HTTP response serialization.
- `src/lib.rs`: `ThreadPool` and `Worker` implementation, thread coordination, and shutdown logic.
- `hello.html`: Static HTML served for 200 OK responses.
- `404.html`: Static HTML served for unrecognized routes.

## Running the Server

Start the server:

```bash
cargo run
```

Test normal response:

```bash
curl -i http://127.0.0.1:8080/
```

Test slow response (demonstrating non-blocking behavior for concurrent clients):

```bash
curl -i http://127.0.0.1:8080/sleep
```

While `curl http://127.0.0.1:8080/sleep` is running, open a separate terminal and run `curl http://127.0.0.1:8080/`. The root request resolves immediately because worker threads handle requests independently.
