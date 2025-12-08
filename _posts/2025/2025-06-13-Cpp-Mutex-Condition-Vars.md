---
title: "Mastering C++ Concurrency: Mutexes and Condition Variables"
toc: true
tags:
    - C++
---

With the C++11 standard, C++ officially joined the concurrent programming party,
giving us a powerful, cross-platform toolkit to manage multiple threads. Before
C++11, we were stuck with platform-specific APIs or third-party libraries. Now,
we have standard, portable tools right in our back pocket.

But with great power comes great responsibility. When multiple threads share the
same data, chaos is always just one race condition or deadlock away. This post
looks at two of the most fundamental tools for bringing order to that
chaos: **mutexes** and **condition variables**.

## The Gatekeeper: `std::mutex`

At its core, concurrent programming is about managing access to shared
resources. Think of a `std::mutex` (short for **mut**ual **ex**clusion) as a
gatekeeper for a piece of data. Only one thread can hold the key to the gate at
a time.

Introduced in **C++11** in the `<mutex>` header, a `std::mutex` has two primary
jobs:

- `lock()`: A thread calls this to wait for the key. If another thread has it,
  this thread blocks until the key is available.
- `unlock()`: The thread that has the key calls this to release it, allowing
  another waiting thread to take its turn.

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

### The Problem with Manual Locking

Looks simple, right? But manually calling `lock()` and `unlock()` is a classic
C++ pitfall. What happens if an exception is thrown inside the critical section?
Or the critical section has multiple early returns and one of them is missing an
`unlock()` call. The `unlock()` call is skipped, and the mutex remains locked
forever. Any other thread waiting for it will be blocked indefinitely. This is a
deadlock.

This is where the
[RAII](https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization)
idiom (Resource Acquisition Is Initialization) saves the day.

## RAII: The C++ Way to Lock Safely

RAII is a core C++ concept: tie a resource's lifetime to an object's lifetime.
When the object is created, it acquires the resource. When it's destroyed (goes
out of scope), it releases the resource, even if exceptions are thrown.

### `std::lock_guard`: The Simple, Safe Lock

For locking, C++11 gave us `std::lock_guard`. It's a simple RAII wrapper. It
locks the mutex in its constructor and unlocks it in its destructor. It's
foolproof.

```cpp
void safe_increment() {
    // The lock is acquired when 'guard' is created.
    std::lock_guard<std::mutex> guard(mtx);
    shared_counter++;
} // The lock is automatically released when 'guard' goes out of scope.
```

`std::lock_guard` is fantastic for simple, scope-based locking. But sometimes,
you need more control.

### `std::unique_lock`: The Flexible, Powerful Lock

Enter `std::unique_lock`, also from C++11. It's a more powerful RAII wrapper
that gives you:

- **Explicit Control**: You can call `lock()` and `unlock()` on the
  `unique_lock` object itself.
- **Deferred Locking**: You can create it without locking the mutex immediately.
- **Movability**: You can transfer ownership of the lock from one `unique_lock`
  to another.

Most importantly, `std::unique_lock` is **essential for working with condition
variables**.

## Going Beyond Locks: `std::condition_variable`

Mutexes are great for preventing threads from stepping on each other's toes. But
what if a thread needs to wait for something to *happen*? What if a "consumer"
thread needs to wait for a "producer" thread to put an item in a shared queue?

You could use a mutex and have the consumer lock, check the queue, unlock, and
repeat. This is called "busy-waiting" or "spinning," and it's horribly
inefficient—it burns CPU cycles doing nothing.

The elegant solution is `std::condition_variable`. It allows a thread to go to
sleep efficiently and wait for a specific condition to become true.

### Working Together: Mutex, Unique Lock, and Condition Variable

These three primitives work as a team:

1. **The Mutex**: Protects the shared data (the "condition").
2. **The Unique Lock**: Manages the mutex, providing the flexibility needed for
   the `wait` operation.
3. **The Condition Variable**: Orchestrates the waiting and notifying.

Here’s the pattern:

- **The Waiting Thread** (e.g., a consumer):
    1. Acquires a `std::unique_lock` on the mutex.
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

Using condition variables requires you to be aware of two specific phenomena.

### Spurious Wakeups

Sometimes, a waiting thread can wake up for no reason at all—no `notify` was
called. This is a "spurious wakeup." It’s a known behavior of threading
libraries on many operating systems.

**The Solution:** Always wait inside a loop that re-checks your condition.

```cpp
std::unique_lock<std::mutex> lock(mtx);
while (queue.empty()) { // Always re-check the condition in a loop!
    cv.wait(lock);
}
````

Even better, use the version of `wait` that takes a predicate (a lambda or
function). It handles the loop for you!

```cpp
// This is the preferred, robust way to wait.
cv.wait(lock, []{ return !queue.empty(); });
```

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

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <queue>

std::mutex mtx;
std::condition_variable cv;
std::queue<int> data_queue;
bool finished = false;

void producer() {
    for (auto i = 0; i < 10; ++i) {
        {
            std::lock_guard<std::mutex> lock(mtx);
            data_queue.push(i);
            std::cout << "Produced: " << i << std::endl;
        } // Lock is released here.
        cv.notify_one(); // Wake up one consumer.
    }

    {
        std::lock_guard<std::mutex> lock(mtx);
        finished = true;
    }
    cv.notify_all(); // Wake up all consumers to check the finished flag.
}

void consumer() {
    while (true) {
        std::unique_lock<std::mutex> lock(mtx);
        // Wait until the queue is NOT empty OR production is finished.
        cv.wait(lock, []{ return !data_queue.empty() || finished; });

        // If production is done and the queue is empty, we can exit.
        if (data_queue.empty() && finished) {
            break;
        }

        int data = data_queue.front();
        data_queue.pop();
        std::cout << "Consumed: " << data << std::endl;
    } // Lock is released here after each iteration when it goes out of scope.
}

int main() {
    std::thread prod_thread(producer);
    std::thread cons_thread(consumer);

    prod_thread.join();
    cons_thread.join();

    return 0;
}
```

In this example, the `mutex` protects the queue and the `finished` flag.
The `condition_variable` allows the consumer to sleep efficiently when the queue
is empty, and the producer to signal it when new data is ready. Notice how the
scope blocks are used to manage the lock lifetime and hence the mutex
acquisition and release.
