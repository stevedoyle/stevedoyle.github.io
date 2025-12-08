---
title: "Using Rust in Advent-of-Code 2023"
tags:
  - Rust
  - AoC
---

This is the first year that I participated in the [Advent of Code](https://adventofcode.com/2023). I decided to take on this years AoC puzzles using Rust. I figured it would be a good way to practice using Rust and to possibly use some new Rust crates. Here are some thoughts after [completing](https://github.com/stevedoyle/advent-of-code) this years puzzles.

- It was fun but it consumed a lot of time. Some days were easier than others. I got satisfaction from solving the puzzles each day, especially if I found the solution using an elegant approach.
- There is a big difference between good programming practices and hacking together a solution to quickly solve the puzzles. For the early puzzles I started with a "good programming practices" approach and spent time on error handling, creating well named data structures to represent the data and problem domain and fairly exhaustive unit tests. I found that I was spending a lot of time on writing "clean" code that wasn't actually making much progress towards solving the puzzle. As the days wore on, my approach shifted towards doing just enough to solve that days puzzle. The input was always well formatted so there was no real need for adding extra code to handle error cases that wouldn't be encountered in the puzzle. Nicely named data structures were a nice to have but not essential as I could cobble together a tuple or a vector of vectors or something similar to process the data. The net result is a lot of "one shot" code which solves the specific problem but is not very readable or maintainable or useful for other purposes. My takeaway here is that while AoC puzzles are great for learning algorithms, it doesn't necessarily foster the use of good programming practices that are essential for writing clean production code.
- Most of the AoC puzzles are prime candidates for using Iterators and chains of iterators. Using for loops now feels wrong. Or at least makes me think that I'm missing a more idiomatic way using iterators.
- AoC's well formed input can lead to short cuts in input validation and error checking. My error checks got fewer and fewer as the days wore on. My regex parsers more and more specific to the exact input string expected without any allowances for badly formed input. It works for AoC but it can lead to laziness and one shot code.
- The `anyhow` crate simplifies error handling. Or at a minimum it improves code readability by being able to use `Result<()>` instead of `Result<(), Box<dyn Error>>`.
- Cycle detection and memoization. Some puzzles required this to be able to find a solution in a reasonable running time. This was not something that I had encountered when writing production code. Mainly because I wasn't writing production code that could have leveraged this to reduce runtimes.
- Graph algorithms are a weak area for me. I struggled a little with the puzzles that required graph algorithms for their solution. I had to spend some time researching these algorithms in order to be able to implement a solution. This is an area where I need to invest some time to improve my knowledge. 
- Overall, using Rust was a good choice. It didn't get in the way of solving the puzzles and the speed of development was good, better than if I had used C or C++ and close to the speed of development using Python.

