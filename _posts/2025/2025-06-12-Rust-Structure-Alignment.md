---
title: "Data Structure Memory Alignment in Rust"
tags:
    - Rust
toc: true
---

Rust’s memory management is very efficient, and it’s no surprise that it also helps with both structure alignment and structure field alignment, which plays a major role in the overall memory layout of a program. A well-aligned structure helps with caching and also improves overall performance.

## Structure Field Alignment for Rust

Structure field alignment in Rust refers to how the fields within a struct are arranged in memory. The Rust compiler automatically handles structure field alignment to ensure that each field is placed at a memory address that is a multiple of its alignment requirement. This is crucial for performance and correctness because many hardware architectures require data to be aligned in specific ways for efficient access.

### Why is Alignment Important?

- **Performance:** Processors often read and write data in "chunks" (e.g., 4 bytes, 8 bytes). If a data item is not aligned to these chunk boundaries, the processor might need to perform multiple memory accesses or more complex operations to retrieve the data, slowing down your program. Properly aligned data can be accessed in a single, efficient operation.
- **Correctness:** Some hardware architectures will simply crash or behave unexpectedly if you try to access unaligned data. While Rust generally protects you from this at a high level, understanding alignment is essential when interacting with low-level code, FFI (Foreign Function Interface), or raw pointers.
- **Cache Efficiency:** Modern CPUs use caches to speed up memory access. When data is properly aligned, it's more likely to fit neatly within cache lines, leading to fewer cache misses and faster execution. On Intel / x86 CPUs, the cache line is 64 bytes while on Apple M1 CPUs the cache line is 128 bytes.

### How to Check The Alignment of Rust Types

There are two common approaches, using the compiler to generate a report and programmatically using `std::mem`.

### Compiler Report

The `rustc -Z print-type-sizes` compiler option generates a report that describes the size and alignment of all types used.

This can also be passed to cargo via: `cargo rustc -- -Z print-type-sizes`

Example output:

```text
print-type-size type: `MyStruct`: 8 bytes, alignment: 4 bytes
print-type-size     field `.b`: 4 bytes
print-type-size     field `.c`: 2 bytes
print-type-size     field `.a`: 1 bytes
print-type-size     end padding: 1 bytes
```

#### Programmatically using std::mem

`std::mem::size_of` and `std::mem::align_of` can be used to inspect these properties:

```rust
use std::mem;

struct MyStruct {
    a: u8,
    b: u32,
    c: u16,
}

fn main() {
    println!("Size of MyStruct: {} bytes", mem::size_of::<MyStruct>());
    println!("Alignment of MyStruct: {} bytes", mem::align_of::<MyStruct>());
}
```

### How Rust Handles Alignment

By default, Rust automatically handles the alignment of struct fields. It ensures that:

- Each field is placed at an offset that is a multiple of its own alignment.
- The overall struct's alignment is the maximum alignment of its fields.
- Padding bytes are inserted between fields as needed to satisfy alignment requirements.
- The order of fields may be rearranged to reduce the amount of padding inserted thereby optimising the overall size of the structure.

Let's consider an example:

```rust
struct MyStruct {
    a: u8,    // 1 byte
    b: u32,   // 4 bytes, alignment 4
    c: u16,   // 2 bytes, alignment 2
}
```

Without any special considerations, you might expect this struct to take `1 + 4 + 2 = 7` bytes. However, due to alignment, its memory layout will likely be more like this:

```text
print-type-size type: `MyStruct`: 8 bytes, alignment: 4 bytes
print-type-size     field `.b`: 4 bytes
print-type-size     field `.c`: 2 bytes
print-type-size     field `.a`: 1 bytes
print-type-size     end padding: 1 bytes
```

The total size of `MyStruct` would be 8 bytes.

- The structure fields were rearranged by the compiler to minimise padding. This is typically achieved by ordering fields from largest to smallest.
- `c` (2 bytes, alignment 2) needs to start at an address that's a multiple of 2. After `b` (4 bytes), the current offset is 4. `c`can start at offset 4.
- Since `a` (1 byte, alignment 1) can start at any address, it can be placed at the current offset which is offset 6.
- Finally, the _total_ size of the struct must be a multiple of its largest alignment requirement (which is 4 for `u32`). After `a` ends at offset 7, 1 more padding byte is added to make the total size 8 bytes, which is a multiple of 4.

### Controlling Alignment with `#[repr()]`

While Rust's default alignment is usually optimal, there are scenarios where you might need explicit control, especially when interoperating with C code or hardware.

#### C Compatible Alignment with `#[repr(C)]`

This is the most common `repr` attribute. It tells Rust to lay out the struct fields in the same order as a C compiler would, respecting C's alignment rules. This is crucial for FFI. It doesn't guarantee _identical_ padding to C compilers, but it guarantees the order and similar alignment rules.

```rust
#[repr(C)]
struct CCompatibleStruct {
	x: u32,
	y: u8,
	z: u16,
}
```

```text
print-type-size type: `CCompatibleStruct`: 12 bytes, alignment: 4 bytes
print-type-size     field `.a`: 1 bytes
print-type-size     padding: 3 bytes
print-type-size     field `.b`: 4 bytes, alignment: 4 bytes
print-type-size     field `.c`: 2 bytes
print-type-size     end padding: 2 bytes
```

#### Packed Alignment with `#[repr(packed)]`

This attribute tells Rust to lay out the struct fields without any padding between them. This can reduce memory usage but might lead to performance penalties or even undefined behavior on some architectures if you access unaligned fields. Use with extreme caution.

```rust
#[repr(packed)]
struct PackedStruct {
    a: u32,
	b: u8,
	c: u16,
}
```

```text
print-type-size type: `PackedStruct`: 7 bytes, alignment: 1 bytes
print-type-size     field `.a`: 1 bytes
print-type-size     field `.b`: 4 bytes
print-type-size     field `.c`: 2 bytes
```

#### Structure Alignment with `#[repr(align(N))]`

This allows you to specify a minimum alignment for the entire struct. The struct's alignment will be at least `N`, and it will be padded at the end to ensure its size is a multiple of `N`. This can be useful to help align structures on cache line boundaries.

```rust
#[repr(align(16))]
struct SixteenByteAligned {
    a: u8,
    b: u32,
    c: u16,
}
```

```text
print-type-size type: `SixteenByteAligned`: 16 bytes, alignment: 16 bytes
print-type-size     field `.b`: 4 bytes
print-type-size     field `.c`: 2 bytes
print-type-size     field `.a`: 1 bytes
print-type-size     end padding: 9 bytes
```

#### Multiple Alignment Specifiers

While the previous examples all used a single `#[repr()]` specified, it is possible to use multiple specifiers. The example below aligns the overall structure on a 16-byte boundary and the individual fields are compatible with a C structure layout.

```rust
#[repr(C, align(16))]
struct SixteenByteCAligned {
    a: u8,
    b: u32,
    c: u16,
}
```

```text
print-type-size type: `SixteenByteCAligned`: 16 bytes, alignment: 16 bytes
print-type-size     field `.a`: 1 bytes
print-type-size     padding: 3 bytes
print-type-size     field `.b`: 4 bytes, alignment: 4 bytes
print-type-size     field `.c`: 2 bytes
print-type-size     end padding: 6 bytes
```

## Cache Line Alignment

Unless `#[repr(packed)]` is used, the compiler alignment of structure fields ensures that an individual field won't span a cache line boundary which is typically 64 bytes for x86 and 128 bytes for Apple M. However, there are cases where a structure may span a cache line boundary. Two common situations are large structures and arrays of structures.

Looking closer at the array of structures case, consider an example where we have an array of 10 `CCompatibleStruct` structures. From the earlier analysis, this structure is 12 bytes and so the array size is 120 bytes. On an x86 system with a 64 byte cache line, this results in one of the structures in the array spanning the cache line boundary.

```rust
#[repr(C)]
struct CCompatibleStruct {
	x: u32,
	y: u8,
	z: u16,
}

// Array of 10 CCompatibleStructs. Each struct is 12 bytes.
let array = [CCompatibleStruct; 10];
```

This is not always a problem, but there are scenarios where it could be. For example, in a multi-threaded application where different threads operate on different sets of structures in the array, then it is desirable for the same cache line not to be shared by different threads. If the different threads were operating on different CPU cores and trying to access different structures that shared a cache line then poor performance will result. In such a scenario, this can be performance optimised by ensuring that each thread operated on a separate cache line and that the a structure didn't span a cache line boundary, using `#[repr(align(N))]` where N is an integer divisor of the cache line size. In the above example, the use of `#[repr(C, align(16))]` ensures that a structure within the array does not span a cache line boundary.

This learning also applies to the large structure scenario. By making large structures a multiple of the cache line size, it helps to optimise cache performance.

## Best Practices

- **Prefer `#[repr(C)]` for FFI:** When interacting with C, always use `#[repr(C)]` to ensure compatible memory layouts.
- **Avoid `#[repr(packed)]` unless absolutely necessary:** The performance and correctness implications can be severe. Only use it when you have a strong reason and understand the risks, such as very specific hardware constraints or extremely memory-constrained environments where you've profiled and confirmed the benefit.
- **Order fields by size (largest to smallest):** While Rust handles alignment for you, arranging fields from largest to smallest can sometimes lead to more compact struct layouts by reducing the need for padding. This is often a good general principle for C-like structs too.
- **Use `rustc -Z print-type-sizes` or `mem::size_of` and `mem::align_of` for introspection:** These functions are invaluable for understanding how your structs are laid out in memory and for debugging alignment issues.
- **Make the size of large structures a cache line multiple**: This avoids multiple structures sharing the same cache line which may cause performance issues in multi-threaded applications.

## Conclusion

Understanding structure alignment and structure field alignment is a key aspect of writing efficient and correct Rust code, especially when dealing with low-level programming or interoperating with other languages. While Rust's default behavior usually does the right thing, the `#[repr()]` attributes provide the necessary control for more advanced scenarios. By being mindful of alignment, you can unlock better performance and avoid subtle bugs in your applications.