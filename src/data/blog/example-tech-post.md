---
author: Daiki Matsui
pubDatetime: 2025-09-21T15:00:00Z
title: Getting Started with Rust - My Learning Journey
slug: getting-started-with-rust
featured: false
draft: false
tags:
  - rust
  - programming
  - learning
description: My experience learning Rust programming language and why it's worth the effort.
---

## Why Rust?

Rust has been gaining popularity in recent years, and for good reason. It offers:

- **Memory safety** without garbage collection
- **Concurrency** without data races
- **Zero-cost abstractions**
- **Great tooling** with cargo

## My First Steps

Here's a simple "Hello, World!" program in Rust:

```rust
fn main() {
    println!("Hello, World!");
}
```

## Key Concepts I've Learned

### Ownership

One of Rust's unique features is its ownership system:

```rust
let s1 = String::from("hello");
let s2 = s1; // s1 is no longer valid here
// println!("{}", s1); // This would cause a compile error
println!("{}", s2); // This works
```

### Pattern Matching

Pattern matching in Rust is powerful and expressive:

```rust
match value {
    1 => println!("one"),
    2 | 3 => println!("two or three"),
    _ => println!("something else"),
}
```

## Resources

- [The Rust Book](https://doc.rust-lang.org/book/)
- [Rust by Example](https://doc.rust-lang.org/rust-by-example/)
- [Rustlings](https://github.com/rust-lang/rustlings)

## Next Steps

I'm planning to build a small CLI tool to practice what I've learned. Stay tuned!