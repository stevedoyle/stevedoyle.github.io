---
title: "Understanding the UnionFind Algorithm: A First-Time Experience"
date: 2025-12-08
tags: [algorithms, data-structures, advent-of-code, rust]
toc: true
---

## Introduction

While solving [Advent of Code 2025 Day 8](https://adventofcode.com/2025/day/8), ([my solution](https://github.com/stevedoyle/advent-of-code/blob/main/2025/src/bin/day08.rs)), I encountered a problem that required efficiently grouping elements and determining connectivity between them. This led me to discover the Union Find (also known as Disjoint Set Union or DSU) data structure, an algorithm I hadn't used before. This post explores what UnionFind is, how it works, and why it's so useful for certain types of problems.

*Disclaimer: I used Claude AI to help generate the initial draft for explaining the Union Find algorithm. I made edits to this draft to fit my style, to improve the diagrams, to add additional explanations and to reference the example back to my AoC Day 8 solution.*

## What is UnionFind?

Union Find is a data structure that efficiently handles two primary operations on disjoint sets:

1. **Find**: Determine which set a particular element belongs to
2. **Union**: Merge two sets together

The data structure excels at answering questions like "Are these two elements in the same group?" or "How many separate groups exist?" with near-constant time complexity.

## The Problem Context

In [Advent of Code 2025 Day 8](https://adventofcode.com/2025/day/8), I needed to identify connected components in a 3D space. Elements were connected based on specific rules, and I had to determine how many distinct groups existed and whether certain elements belonged to the same group.

This is a classic use case for UnionFind: dynamic connectivity problems where you're repeatedly merging groups and checking group membership.

## How UnionFind Works

### Core Concept

UnionFind maintains a forest of trees, where each tree represents a disjoint set. Each element points to its parent, and the root of each tree serves as the representative (or "leader") of that set.

```text
Initial State (5 elements, each in its own set):
Element:     0    1    2    3    4
             │    │    │    │    │
Parent:     [0]  [1]  [2]  [3]  [4]  (each element is its own parent/root)

After union(0, 1) and union(2, 3):
(roots: 1, 3, 4)
    1         3         4
    │         │         │
    0         2         4

After union(1, 3):
(roots: 3, 4)
        3
       ╱ ╲
      1   2
      │
      0         4
```

### Basic Structure

```rust
struct UnionFind {
    parent: Vec<usize>,
    rank: Vec<usize>,  // Upper bound on tree height
}

impl UnionFind {
    fn new(size: usize) -> Self {
        UnionFind {
            parent: (0..size).collect(),  // Each element is its own parent initially
            rank: vec![0; size],           // Single-node trees have height 0
        }
    }
}
```

Initially, each element is in its own set, acting as its own parent. The `rank` tracks an upper bound on the tree height, starting at 0 for single-node trees.

### The Find Operation

The `find` operation locates the root (representative) of the set containing a given element:

```rust
fn find(&mut self, x: usize) -> usize {
    if self.parent[x] != x {
        // Path compression: make x point directly to root
        self.parent[x] = self.find(self.parent[x]);
    }
    self.parent[x]
}
```

**Path Compression** is the key optimization here. As we traverse up the tree to find the root, we update each node to point directly to the root. This flattens the tree structure, making future operations faster.

```text
Before Path Compression:        After find(4):
        0                              0
        │                          ┌───┼───┐
        1                          1   2   4
        │
        2
        │
        4

Path: 4→2→1→0                   All nodes point directly to root
```

### The Union Operation

The `union` operation merges two sets by connecting their roots:

```rust
fn union(&mut self, x: usize, y: usize) -> bool {
    let root_x = self.find(x);
    let root_y = self.find(y);

    if root_x == root_y {
        return false;  // Already in same set
    }

    // Union by rank: attach smaller tree under larger tree
    if self.rank[root_x] < self.rank[root_y] {
        self.parent[root_x] = root_y;
    } else if self.rank[root_x] > self.rank[root_y] {
        self.parent[root_y] = root_x;
    } else {
        self.parent[root_y] = root_x;
        self.rank[root_x] += 1;
    }

    true
}
```

**Union by Rank** ensures we attach the smaller tree to the larger one, keeping trees shallow and operations fast.

```text
Without Union by Rank:          With Union by Rank:
    0                               0
    │                           ┌───┼───┐
    1                           1   2   3
    │                           │
    2                           4
    │
    3
    │
    4
Height: 4                       Height: 2
```

## Key Optimizations

### Path Compression

Without path compression, repeatedly finding elements in a deep tree would be slow. Path compression flattens the tree during `find` operations, ensuring subsequent operations are nearly constant time.

### Union by Rank

By always attaching the smaller tree to the larger tree's root, we maintain balanced trees. This prevents worst-case scenarios where trees become linear chains.

**Why rank starts at 0:** The `rank` field represents an upper bound on the tree height. A single-node tree has height 0, so rank starts at 0. The rank only increases when merging two trees of equal rank:

- **Different ranks**: Attach lower-rank tree to higher-rank root (no height increase)
- **Equal ranks**: Attach one to the other and increment the resulting tree's rank (height increases by 1)

This ensures the rank accurately reflects the tree structure and maintains the logarithmic height bound.

## Time Complexity

With both optimizations:

- **Find**: O(α(n)) amortized, where α is the inverse Ackermann function
- **Union**: O(α(n)) amortized

The inverse Ackermann function grows so slowly that for all practical purposes, these operations are effectively constant time (α(n) < 5 for any realistic input size).

## Common Use Cases

UnionFind excels at:

1. **Connected Components**: Finding clusters in graphs or grids
2. **Cycle Detection**: Determining if adding an edge creates a cycle
3. **Kruskal's Algorithm**: Finding minimum spanning trees
4. **Network Connectivity**: Determining if nodes are connected
5. **Image Processing**: Identifying connected regions in images

## Visual Example: Grid Connectivity

Consider a grid where we want to connect adjacent cells with the same value:

```text
Grid:           Initial Sets:       After Connecting Same Values:
A A B           0 1 2               0─1     2
A C B           3 4 5               │       │       4
D C C           6 7 8               3       5       │
                                                    7─8
                                    6

Final Groups:
- Group 1 (A): {0, 1, 3} - 0─1 horizontal, 0─3 vertical
- Group 2 (B): {2, 5}    - 2─5 vertical
- Group 3 (C): {4, 7, 8} - 4─7 vertical, 7─8 horizontal
- Group 4 (D): {6}       - isolated
```

## Conclusion

The UnionFind algorithm is an elegant solution to dynamic connectivity problems. Using it for [solving AoC Day 8](https://github.com/stevedoyle/advent-of-code/blob/main/2025/src/bin/day08.rs) showed me how the right data structure can turn a complex problem into a straightforward implementation. The combination of path compression and union by rank provides near-constant time operations, making it practical for real-world applications and large data sets.

## Further Reading

- [Algorithms, 4th Edition by Robert Sedgewick and Kevin Wayne](https://algs4.cs.princeton.edu/15uf/)
- [CP-Algorithms: Disjoint Set Union](https://cp-algorithms.com/data_structures/disjoint_set_union.html)
- [Visualization of UnionFind](https://visualgo.net/en/ufds)
