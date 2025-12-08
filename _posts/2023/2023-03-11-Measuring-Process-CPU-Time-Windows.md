---
title: "Measuring Process CPU Time On Windows"
tags:
    - C++
    - Performance
    - Windows
toc: true
---

I normally develop on and for Linux based systems, but recently I've been
drafted in to help rescue a Windows based software project that was embarking on
a [death
march](https://en.wikipedia.org/wiki/Death_march_\(project_management\)). As is
often the case, the current state of the product functions reasonably OK but its
performance is widely missing its target KPIs. My job is to provide some rigour
and guidance for performance analysis and optimization and move the team away
from "I think this is the problem ..." opinion based approach and towards "The
data says that this is the problem" data driven approach. As usual, there was
some resistance, but walking the team through some examples and demonstrating
measured improvements has helped to get them on board.

Anyhow, back to the topic. The product is a C++ library that must perform a task
within a CPU cycle budget. The library is being exercised in a multi-threaded
test application and the existing methodology for measuring the CPU cycle budget
was to try and note the system CPU utilisation while the test application was
running (and the rest of the system was reasonably idle) and then work backwards
to figure out the CPU cycles used per operation. It was error prone, not very
automatable or repeatable and was contributing to the products performance woes.
I wanted to change this methodology to use something programmatic that could be
measured within the test application itself, across multiple threads and without
incurring overhead on the code being measured. The benefit of this is that the
data is available as part of the test application output report each time it is
run and hence automatable.

The question was, how do I do this on Windows for a C++ application. The code
in question consists of multiple threads, running on different CPU cores, that
co-operate to perform an operation so an approach of using simple timestamp
counters around a code block wasn't going to work. After some searching, I found
`QueryProcessCycleTime()`, `QueryThreadCycleTime()`, `GetProcessTime()` and
`GetThreadTime()`. The Query functions return the number of CPU cycles that the
process (i.e. application) used across all CPU cores and the `Get*Time()`
functions report the user and kernel time, in 10ns units, for each
process/thread. Both of these sets of functions offered a solution as they could
be called at the beginning of the big block of code to be executed and again at
the end and the difference represents the cycles and time take by the code being
measured.

Digging a little further into choosing whether to choose CPU cycles or
user/kernel time, I found [this
analysis](http://blog.kalmbachnet.de/?postid=28) of the differences between the
approaches. It highlights that the `GetProcessThreadTimes` functions are based
on retrieving information from a threads thread-information-block and this
information is only updated when the scheduler interrupt reschedules the process
thread. Most importantly this information may be incorrect if the thread
voluntarily sleeps. Based on this analysis, we chose the
`QueryProcessCycleTime()` and `QueryThreadCycleTime()` functions and after a
week of using them, they are proving valuable in measuring our progress towards
the target CPU cycle budget. This was a good learning exercise!

## Example

```cpp
#include <iostream>
#include <Windows.h>

int main() {
    uint64_t startCycles;
    uint64_t endCycles;

    QueryProcessCycleTime(GetCurrentProcess(), &startCycles);

    // Insert code to be measured

    QueryProcessCycleTime(GetCurrentProcess(), &endCycles);

    uint64_t elapsed = endCycles - startCycles;
    std::cout << "Elapsed: " << elapsed << " clock cycles" << std::endl;
}
```
