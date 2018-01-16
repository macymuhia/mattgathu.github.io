---
layout: post
title: "Rust 2018 Reading Journal"
date: January 16, 2018
categories:
- rust
- code
description: 'summary of posts about Rust that I read in 2018'
keywords: rust, posts, journal, 2018

---

1. [Jan-15-2018 - Peeking inside Trait Objects](#jan-15-2018)
2. [Jan-16-2018 - What does Rust's "Unsafe" mean?](#jan-16-2018)

## Jan-15-2018

**Title:** [Peeking inside Trait Objects][1]

Rust's trait system forms the basis of the generic system and polymorphic functions and
types. Traits allow abstraction over behavior that types can have in common.

Example

```rust
// trait definition
trait Foo {
    fn method(&self) -> String;
}

// implementation of trait Foo for type u8
impl Foo for u8 {
    fn method(&self) -> String { format!("u8: {}", *self) }
}
```
**Trait objects** are normal values that store a value of any type that implements a given trait,
where the precise type can only be known at runtime.

Trait objects are obtained by **casting** and **coercions**.

Representation

```rust
pub struct TraitObject {
    pub data: *mut (), // data pointer
    pub vtable: *mut (), // vtable pointer
}
```
The **data pointer** addresses the data that the trait object is storing, while the **vtable pointer**
points to the virtual method table corresponding to the trait implementation.

## Jan-16-2018

**Title:** [What does Rust's "Unsafe" mean?][2]

Rust aims to be memory safe, so that code cannot crash due to dangling pointers or iterator
invalidation. There are things that cannot fit the type system e.g. interaction with the OS and
system libs. Rust fills these gaps using the `unsafe` keyword.

There are two ways of opting into unsafe behavior: with an `unsafe` block or with an `unsafe`
function

```rust
// calling some C functions imported via FFI
unsafe fn foo() {
    some_c_function();
}
fn bar() {
    unsafe {
        another_c_function();
}
```
The `unsafe` context allows one to:
- call unsafe functions
- dereference raw pointers
- access a mutable global variable
- use inline assembly

> An `unsafe` context is the programmer telling the compiler that the code is guaranteed to be safe
> due to invariants impossible to express in the type system, and that it satisfies the [invariants
> that Rust itself imposes][3].

[1]: http://huonw.github.io/blog/2015/01/peeking-inside-trait-objects/
[2]: http://huonw.github.io/blog/2014/07/what-does-rusts-unsafe-mean/
[3]: https://doc.rust-lang.org/nightly/reference/behavior-considered-undefined.html
