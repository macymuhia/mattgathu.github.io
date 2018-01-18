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
3. [Jan-17-2018 - What's Tokio and Async IO All About?](#jan-17-2018)
4. [Jan-18-2018 - A Journey into Interators](#jan-18-2018)

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
> due to invariants impossible to express in the type system, and that it satisfies the 
> [invariants that Rust itself imposes][3].


## Jan-17-2018

**Title:** [What's Tokio and Async IO All About?][4]

One of Rust's key features if "**fearless concurrency**". But the kind of concurrency required for
handling a large amount of I/O bound tasks is absent from Rust.

"_Handling N things at once_" is best done by using threads. However, OS threads do not scale when
**N** is large: each thread needs to allocate a stack, setting up a thread involves a bunch of
syscalls, and context switching is expensive. We need "lighter" threads.

The Go language has lightweight threads, called "**goroutines**". They are spawn using the `go`
keyword.

```go
listener, err = net.Listen(...)
// handle err
for {
    conn, err := listener.Accept()
    // handle err

    // spawn goroutine:
    go handler(conn)
}
```
This is a loop that waits for new TCP connections, and spawns a new goroutine to handle each connection.
The spawned goroutine will shut down when `handler` finishes. The main loop keeps executing since
it runs in a different goroutine. 

Go implements an **M:N threading model** with a scheduler that
swaps goroutines in and out, much like the OS scheduler. Rust _used_ to support lightweight threads
pre-1.0 (feature got removed since it wasn't _zero cost_).

Two building blocks of lightweight threading are **Async I/O** and **futures**.

In regular blocking I/O, when you request I/O your thread will not be allowed to run until the
operation is done. In Async (non-blocking) I/O a thread queues a request for I/O with the OS and
continues execution. The I/O request is executed at some later point by the kernel. The OS provides
system calls like `epoll` for Async I/O tasks.

The Rust library [mio][5] is a platform-agnostic wrapper around non-blocking I/O and interfaces
such as epoll, kqueue etc. It forms a building block for Async I/O.

**Futures** are another building block. A Future is the promise of eventually having a value.
Futures can be `wait`ed (blocking), `poll`ed and chained.

**[Tokio][6]** is a wrapper around mio that uses futures. It has a core event loop, and you feed it
closures that return futures. The event loop schedules the tasks (closures) passed to it. 

Rust's code that has futures isn't elegant/pretty. **Generators** and the **async/await** syntax
are experimental features aimed at reducing the boilerplate code, essentially
turning

```rust
fn foo(...) -> Future<ReturnType, ErrorType> {
    do_io().and_then(|data| do_more_io(compute(data)))
          .and_then(|more_data| do_even_more_io(more_compute(more_data)))
    // ......
}
```
to

```rust
#[async]
fn foo(...) -> Result<ReturnType, ErrorType> {
    let data = await!(do_io());
    let result = compute(data);
    let more_data = await!(do_more_io());
    // ....
```

## Jan-18-2018

**Title:** [A Journey into Iterators][7]

Rust supports iterators. They are a fast, safe, lazy way of working with data structures, streams,
and other more creative applications.

A Rust iterator implements the [`Iterator`][8] trait. There are also traits from conversion from
and into iterators.

Example
```rust
fn main() {
    let input = [1, 2, 3]; // define a set of values
    let iterator = input.iter(); // create an iterator over the them.
    let mapped = iterator.map(|&x| x * 2); // map iterator to a closure
    let output = mapped.collect::<Vec<usize>>(); // convert iterator into a collection
    println!("{:?}", output);
}
```
Iterators provide convenient methods such as `.next()` for single element iteration, `.take(n)` for
a batch pick and `.skip(n)` for discarding `n` elements.

Many of Rust's collection data structures support iterators e.g. Vectors, VecDeques, HashMaps.

Writing an iterator simply means implementing the `Iterator` trait.

```rust
struct CountUp {
    current: usize,
}

impl Iterator for CountUp {
    type Item = usize;
    // The only fn we need to provide for a basic iterator.
    fn next(&mut self) -> Option<usize> {
        self.current += 1;
        Some(self.current)
    }
}
```

Rust's ranges also implement the `Iterator` traits.

Iterators can be merged, chained, even split into other iterators. They can also be `filter`ed and
`fold`ed.

```rust
fn main() {
	let input = 1..10;
	let output = input
    	.filter(|&item| item % 2 == 0) // Keep Evens
    	.map(|item| item * 2) // Multiply by two.
    	.fold(0, |accumulator, item| accumulator + item);
	println!("{}", output);
}
```

Check out [**`std::collections`**][9] and [**`std::iter`**][10]

[1]: http://huonw.github.io/blog/2015/01/peeking-inside-trait-objects/
[2]: http://huonw.github.io/blog/2014/07/what-does-rusts-unsafe-mean/
[3]: https://doc.rust-lang.org/nightly/reference/behavior-considered-undefined.html
[4]: https://manishearth.github.io/blog/2018/01/10/whats-tokio-and-async-io-all-about/
[5]: https://github.com/carllerche/mio
[6]: https://github.com/tokio-rs/tokio-core
[7]: https://hoverbear.org/2015/05/02/a-journey-into-iterators/
[8]: https://doc.rust-lang.org/core/iter/index.html#traits
[9]: https://doc.rust-lang.org/beta/std/collections/
[10]: https://doc.rust-lang.org/beta/std/iter/
