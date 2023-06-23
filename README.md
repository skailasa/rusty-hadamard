# Rusty Hadamard

## Test

```bash
cd src && rustc +nightly -O --test main.rs && ./main --bench
```

Hadamard Product using Explicit SIMD in Rust.

Practice for SIMD programming in general

## SIMD Intro

Faced with limits of Dennard scaling, idea: If you can't make a register any faster, you can make it wider. I.e. process more data at once.

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

