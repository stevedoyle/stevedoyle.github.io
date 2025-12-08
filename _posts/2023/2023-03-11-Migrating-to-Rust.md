---
title: "Migrating to Rust"
tags:
    - Rust
---

Hearing of companies migrating from C/C++ to Rust has been quite common in
recent years. The safety guarantees provided by the language outweigh its [steep
learning
curve](https://blog.rust-lang.org/2020/04/17/Rust-survey-2019.html#rust-adoption---a-closer-look
"steep learning curve"). I was quite surprised to read that [Turborepo are
making the transition to Rust from
Go](https://vercel.com/blog/turborepo-migration-go-rust)!

Turborepo primary reason for the migration is a preference towards languages
that prioritize up front correctness, which Rust clearly does. It is not that Go
doesn't prioritize correctness, it just that Go is targeted towards fast network
workloads in a data center environment where fixes for errors caught at runtime
can be quickly rolled out and Turborepo's focus is on software that is installed
by a user and hence the quick rollout of bug-fixes are slower and more
problematic.

Turborepo also have a need to integrate with existing C/C++ libraries and have
found that doing this via Rust is easier than in Go.

I'm curious to see if other companies / products migrate to Rust from
programming languages other than C/C++, especially from modern languages.
Personally, I think its a case of picking the right language to suit your
software product and deployment model.
