---
title: "First Impressions Programming In Rust"
tags:
    - Rust
---

A couple of years ago I had skimmed through the [Rust
book](https://doc.rust-lang.org/book/) but I hadn't really done any Rust
programming until a few months ago where I started using it to do some
experiments with WebAssembly and WASI interfaces using
the [Wasmtime](https://wasmtime.dev/) runtime which is written in Rust. I was
also keen to do something with Rust since it was announced last year that [Rust
has made it into the Linux
kernel](https://thenewstack.io/rust-in-the-linux-kernel/).

I've captured some of my first impressions of programming with Rust. As a little
background, I've been programming in C and C++ since the mid 1990's and in
Python since early 2000's. I've also dabbled in Java, Ruby and a little Go, but
the bulk of my professional development has been in C/C++.

First off, the Rust toolchain, and in particular Cargo, is a dream to work with.
Having spent many an hour tweaking makefiles and autoconf settings when working
with C/C++, Cargo is so much easier and more intuitive. Overall the whole Rust
toolset, including rustup is really well thought out and everything that you
would expect from a modern day language toolchain.

One of the main advantages of Rust over C/C++ is it's improved memory safety.
It's strict rules about lifetimes, borrowing and mutable references make a lot
of sense when thinking about how they improve memory safety, but they do take a
while to get used to. I've certainly spent a lot of time deciphering compiler
errors because I had more than one mutable reference to a variable in scope at
the same time or that I used the incorrect syntax when passing a Vector to a
function that was expecting a mutable slice. At the time it was frustrating to
work through these and I thought longingly about programming in Python where
things just worked. But it was also an opportunity to get to know the language
better and forced me to take the time to fully understand how these features
worked and why they are the way that they are. For others learning to program in
Rust, I highly recommend that they take the time to really study the chapters on
borrowing, references and slices and to practice using these in simple programs
to get the hang of them.

Error handling, Options and Results are much different compared to C and C++ and
even Python. Out-of-band exceptions are gone and errors are now passed in band
in Result structures. This also takes some getting used to and apart from
learning some idioms for using Result, appending a "?" to a function call, or to
test a returned variable to see if it is an Error and to unwrap it if not, I
don't think that I have fully mastered this feature yet.

Rust generics and traits also look very interesting. While generics remind me of
C++ templates and traits to Python decorators, I haven't really had the
opportunity to create too many new generic structures or to create new traits so
my experience has been limited to using existing types in the various crates
that I've used. I'll reserve judgement on these until I've had a chance to use
them more.

So there are some of my first impressions of programming in Rust. While it has
been a little frustrating to begin with I'm a fan of the language and can
certainly see the advantage of it's memory safety features and I think it, or a
derivative of it will become the dominant low level system programming language
in the not too distant future.
