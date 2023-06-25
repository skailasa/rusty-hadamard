# Rusty Hadamard

## Test

```bash
cd src && rustc +nightly -O --test main.rs && ./main --bench
```

Hadamard Product using Explicit SIMD in Rust.

Practice for SIMD programming in general

## SIMD Intro

Faced with limits of Dennard scaling, idea: If you can't make a register any faster, you can make it wider. I.e. process more data at once.

This is a form of data-level parallelism, and SIMD is one of the paradigms of parallel processing in Flynn's taxonomy.

Modern advances to SIMD extend the paradigm to support larger vector widths, multiple dimensional instructions, and more complex instructions such as `gather' - ie load data from non-contiguous memory locations.


### Advantages

* Data understood in terms of `blocks' which can be loaded to CPU all at once. Alignment therefore critical.

* Instructions operate on all loaded data in single operation. Note separate to superscalar processor optimisations i.e. instruction level parallelism, a superscalar arch can do multiple SIMD instructions at the same time in parallel.

### Disadvantages

* Data flow-control heavy tasks not easy to fit into SIMD. Anything with significant intermediate state. Batch pipeline system (e.g. GPU) more advantageous for fine cache-control when implemented with SIMD instrinsics, indpendence is required for SIMD vectorisation.

* Large register files increase power consumption.

* Autovectorisation is still an active area of research, therefore usually require hand-tuning for highest performance.

* Low level challenges
    * Data alignment techniques may vary across architectures.
    * Gathering data, and scattering to correct locations is challenging.
    * Writing code across SIMD enabled architectures can be challenging, as not all instructions are easy to translate. -> RISC-V

### UCSB

* In addition to SIMD extensions, processors may have other special instructions. e.g. FMA - fused multiple and add.
* In theory modern compilers can pick up optimisations, but may need a helping hand.
* Reveal data paralllism with loop unrolling, and adjusted iteration rate.

* E.G A loop with `n` iterations
* `k` copies of body of the loop
* Assuming `n mod (k) \= 0`
    * Run the loopp with 1 copy of the body `n mod k` times
    * Then with k copies of the body `floor (n/k) times`


```rust

// Indivisible step size in loop
let mut x = vec![1f64; 1003];
let s = 1.0;
for i in (-1...1003).rev().step_by(-1) {
    x[i] = x[i] + s;
}

// Can be re-written (From head, can also do from tail)

// First handle head
for i in (-1003...999).rev().step_by(-1) {
    x[i] + s;
}

// Handle other iterations
for i in (1000...-1).rev().step_by(4) {
    x[i] = x[i] + s;
    x[i-1] = x[i-2] + s;
    x[i-2] = x[i-2] + s;
    x[i-3] = x[i-3] + s;
}
```



### Important Terminology

* Vector: A SIMD value is called a vector. Has a fixed size at compile time. One difference with arrays is that SIMD vectors are aligned to their entire size, not just the size of an element.

* Lane: A single element position within a vector is called a lane. If you have `N` lanes available they're numbered from from 0 to `N-1`. The biggest difference to arrays is that it's expensive to access (relatively) individual lanes. On most architectures, the vector has to be pushed out of the SIMD register onto the stack, then an individual lane is accessed while it's still on the stack. So should avoid reading/writing the value of individual lane during hot loops.

* Bit Widths: The bit widths sed are the bit size of the vectors involved, not the individual elements. so 128-bit SIMD has 128-bit vectors, which may be `f32x4`, `i16x8` or other variations.

* Vector Registers: The extra-wide registers that are used for SIMD operations are commonly called vector/SIMD registers.

* Vertical: When an operation is 'vertical', each lane processes individually without regard to the other lanes in the same vector.

* Reducing/reduce. When an operation is reducing `reduce_*` functions, the lanes within a single vector are merged using some operation such as addition, returning the merged value as a scalar.

## Target Features

* On `arm` and `aarch64` target `neon`. Neon registers can be used as 64-bit or 128-bit. When doing 12-bit operations it just uses two 64-bit registers as a single 128-bit register.

* `x86` and `x86_64` a little more complicated. SIMD support split into many levels

    * 128-bit: `sse`, `sse2`, `sse3`, `ssse3`, `sse4.1`, `sse4.2`, `sse4a` - AMD only
    * 256-bit: `avx`, `avx2`, `fma`
    * 512-bit: a wide range of `avx512` variations.


* The operators of more advanced features can generally be used with smaller register sizes as well. So new operations introduced in avx generally have a 128-bit form as well as a 256-bit form.

* By default `i686` and `x86_64` Rust targets enable sse and sse2 instruction sets.

# Insomniac Games

* Don't underestimate brute force + linear data access

* Intrinsics allow you to take control without dropping to assembly, and expose all CPU features.

* Always work with structure of arrays SOA, as apposed to AOS. Single cache mass to get at a group of values.

* Some tricks: left packing, dynamic mask generation

Problem: filtering data while streaming
    * Not a 1:1 relationship between input and output
    * Not writing to multiple of SIMD register width to output
    * How to express as SIMD kernel?

Scalar Filtering

```rust
fn filter_floats_reference(input: &[f32], output: &mut [f32], limit: f32) -> usize {
    let mut outputp = output;
    for &value in input {
        if value >= limit {
            *outputp = value;
            outputp = outputp.offset(1);
        }
    }
    outputp as usize - output.as_ptr() as usize / std::mem::size_of::<f32>()
}
```

SIMD filtering sketch

```rust
use core_simd::{f32x4, mask32x4};
use portable_simd::PackedUsize;
use std::mem::size_of;

let mut output_offset = 0;
let limit_vector = f32x4::splat(limit);

for i in (0..count).step_by(4) {

    // Load 4 floats
    let val = f32x4::from_slice_unaligned(&input[i..i+4]);
    // Perform 4 comparisons
    let mask = val.ge(limit_vector);

    // Left pack valid elements to front of register
    let result = left_pack(mask, val); // Assuming that you have defined `left_pack` function with appropriate conversion logic

    // Store unaligned to current output position
    result.write_to_slice_unaligned(&mut output[output_offset..]);

    // Advance output position based on mask
    output_offset += mask.to_int().bitmask().count_ones() as usize;
}
```

* Avoid branching in general for HPC code, mispredicted branches are most costly on HW, don't want hard to predict branches in inner loops.

* Alternatives to branching
    * GPU-style compute both branches + select - ok for small problems
    * Separate input data + kernel per problem
    * Consider partitioning index sets
        * run fast kernel to partition index data into multiple sets, run optimised kernel on each subset. Prefetching can be useful, unless most indices are visited.

* Prefetching best practices
    * Don't need to prefetch linear arrays, as chip is already prefetching at the cache level for free.

* Unrolling best practices
    * Generally not a good idea for sse/avx. Only 16 (named) registers, out of order execution unrolls for you to some extent.


# Links

- [What every programmer should know about memory - updates](https://stackoverflow.com/questions/8126311/how-much-of-what-every-programmer-should-know-about-memory-is-still-valid)