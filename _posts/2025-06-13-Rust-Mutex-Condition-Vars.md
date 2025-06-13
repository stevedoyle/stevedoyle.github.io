---
title: "Rust Concurrency: Mutexes and Condition Variables"
toc: true
tags:
    - Rust
---

Rust has first class support for concurrency via `std::sync`. This post looks at
two elements of this, **mutexes** and **condition variables**.

## The Gatekeeper: `std::sync::Mutex`

At its core, concurrent programming is about managing access to shared
resources. With a `std::sync::Mutex` as a
gatekeeper for a piece of data, only one thread can hold the key to the gate at
a time.

A `std::sync::Mutex` provides two primary methods:

- `lock()`: A thread calls this to wait for the key. If another thread has it,
  this thread blocks until the key is available. Once the key is available it is
  returned wrapped in an
[RAII](https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization)
idiom (Resource Acquisition Is Initialization) guard.
- `try_lock()`: A thread attempts to acquire the key. If another thread has it,
  an `Err` is returned, otherwise, an RAII guard is returned.

Notice that there is no explicit `unlock()` method. When the RAII guard returned
from `lock()` or `try_lock()` goes out of scope, the mutex will be unlocked.
This is an important aspect of Rusts concurrency features where RAII semantics
always apply.

```rust
use std::sync::{Arc, Mutex};
use std::thread;

let mutex = Arc::new(Mutex::new(0));
let c_mutex = mutex.clone();

thread::spawn(move || {
    *c_mutex.lock().unwrap() = 10;
}).join().expect("thread::spawn failed");
assert_eq!(*mutex.lock().unwrap(), 10);
```

## Beyond Locks: `std::sync::Condvar`

Mutexes are great for preventing threads from stepping on each other's toes. But
what if a thread needs to wait for something to *happen*? What if a "consumer"
thread needs to wait for a "producer" thread to put an item in a shared queue or
to indicate some condition?

You could use a mutex and have the consumer lock, check the queue, unlock, and
repeat. This is called "busy-waiting" or "spinning," and it's horribly
inefficient—it burns CPU cycles doing nothing.

The elegant solution is `std::sync::Condvar`. It allows a thread to go to
sleep efficiently and wait for a specific condition to become true.

### Working Together: Mutex and Condition Variable

These two primitives work as a team:

1. **The Mutex**: Protects the shared data (the "condition").
2. **The Condition Variable**: Orchestrates the waiting and notifying.

Here’s the pattern:

- **The Waiting Thread** (e.g., a consumer):
    1. Acquires a `lock()` on the mutex.
    2. Checks the condition (e.g., `is the queue empty?`).
    3. If it needs to wait, it calls `cv.wait(lock)`. This function *atomically*
       releases the lock and puts the thread to sleep.
    4. When woken up, it re-acquires the lock and can check the condition again.

- **The Notifying Thread** (e.g., a producer):
    1. Acquires a lock on the same mutex.
    2. Changes the state (e.g., adds an item to the queue).
    3. Calls `cv.notify_one()` or `cv.notify_all()` to wake up one or all
       waiting threads.

## Navigating the Pitfalls

As with any programming language or concurrency library, using condition
variables requires you to be aware of two specific phenomena.

### Spurious Wakeups

Sometimes, a waiting thread can wake up for no reason at all — no `notify` was
called. This is a "spurious wakeup." It’s a known behavior of threading
libraries on many operating systems.

**The Solution:** Always wait inside a loop that re-checks your condition.

```rust
let pair = Arc::new((Mutex::new(false), Condvar::new()));
let (lock, cv) = &*pair;
let mut started = lock.lock().unwrap();
while !* started {
    started = cv.wait(started).unwrap();
}
````

### Lost Wakeups

This is a logic bug where a `notify` is sent *before* a thread starts waiting,
causing the notification to be lost and the thread to wait forever.

**The Solution:** Strictly follow the correct protocol. The waiting thread must
lock the mutex *before* checking the condition. The `cv.wait()` call then
ensures the lock is released atomically as the thread goes to sleep, preventing
any race condition where a notification could be missed.

## Tying it all Together: The Producer-Consumer Example

This classic problem is the perfect showcase. One thread produces data and puts
it in a queue; another consumes it.

```rust
use std::collections::VecDeque;
use std::sync::{Arc, Condvar, Mutex};
use std::thread;
use std::time::Duration;

struct Mailbox {
    queue: VecDeque<i32>,
    finished: bool,
}

impl Mailbox {
    fn new() -> Self {
        Mailbox {
            queue: VecDeque::new(),
            finished: false,
        }
    }

    fn send(&mut self, value: i32) {
        self.queue.push_back(value);
    }

    fn receive(&mut self) -> Option<i32> {
        if self.queue.is_empty() && self.finished {
            None
        } else {
            self.queue.pop_front()
        }
    }

    fn finish(&mut self) {
        self.finished = true;
    }
}

fn producer(mailbox: Arc<(Mutex<Mailbox>, Condvar)>) {
    let (mailbox, condvar) = &*mailbox;
    for i in 0..10 {
        println!("Producing {}", i);
        let mut mb = mailbox.lock().unwrap();
        mb.send(i);
        condvar.notify_one(); // Notify the consumer that a new item is available
        thread::sleep(Duration::from_millis(100));
    }
    let mut mb = mailbox.lock().unwrap();
    mb.finish();
    println!("Producer done");
}

fn consumer(mailbox: Arc<(Mutex<Mailbox>, Condvar)>) {
    let (mailbox, condvar) = &*mailbox;
    loop {
        let mut mb = mailbox.lock().unwrap();
        while mb.queue.is_empty() && !mb.finished {
            println!("Mailbox empty, waiting");
            mb = condvar.wait(mb).unwrap();
        }

        // The mailbox is locked and either has items or is finished.
        if let Some(value) = mb.receive() {
            println!("Received {}", value);
        } else if mb.finished {
            break;
        }
    }
}

fn main() {
    let mailbox = Arc::new((Mutex::new(Mailbox::new()), Condvar::new()));

    // Producer thread
    let producer_mailbox = Arc::clone(&mailbox);
    let producer_thread = thread::spawn(move || {
        producer(producer_mailbox);
    });

    // Consumer thread
    let consumer_mailbox = Arc::clone(&mailbox);
    let consumer_thread = thread::spawn(move || {
        consumer(consumer_mailbox);
    });

    producer_thread.join().unwrap();
    consumer_thread.join().unwrap();
    println!("All done");
}

```

## Comparison with C++

Using mutexes and condition variables in Rust is a little different compared to
using [mutexes and condition variables in
C++](/Cpp-Mutex-Condition-Vars).

- In Rust the mutex wraps the data that it is protecting while in C++ the mutex
and the data are separate.

    ```cpp
    std::mutex mtx;
    int shared_counter = 0;

    void increment() {
        mtx.lock();
        // Critical Section: Only one thread can be here at a time.
        shared_counter++;
        mtx.unlock();
    }
    ```

    ```rust
    let shared_counter = Arc::new(Mutex::new(0));

    *shared_counter.lock().unwrap() += 1;
    ```

- In Rust, a `std::sync::Mutex` always uses RAII guards. It is closer to the C++
  `std::unique_lock` rather than the `std::mutex` which does not have RAII
  semantics.
