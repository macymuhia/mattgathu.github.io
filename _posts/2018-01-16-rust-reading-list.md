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
4. [Jan-18-2018 - A Journey into Iterators](#jan-18-2018)
5. [Jan-19-2018 - Error Handling in Rust](#jan-19-2018)
6. [Jan-20-2018 - Pretty State Machine Patterns in Rust](#jan-20-2018)
7. [Jan-21-2018 - Finding Closure in Rust](#jan-21-2018)
8. [Jan-22-2018 - Why is Rust difficult?](#jan-22-2018-1)
9. [Jan-22-2018 - Macros](#jan-22-2018-2)
10. [Jan-23-2018 - Macros in Rust pt1](#jan-23-2018)
11. [Jan-24-2018 - Macros in Rust pt2](#jan-24-2018)
12. [Jan-25-2018 - Macros in Rust pt3](#jan-25-2018)
13. [Jan-26-2018 - Macros in Rust pt4](#jan-26-2018)

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


## Jan-19-2018

**Title:** [Error Handling in Rust][11]

Error handling is divided into two broad categories: exceptions and return values. Error handling
in Rust is implemented via return values.

To `unwrap` something is to say, "Gimme the result, else just panic and stop execution". The
`Option` and `Result` types implement the `unwrap` method.

The `Option` type is a way to use Rust's type system to express the **possibility of absence**.
```rust
enum Option<T> {
    None,
    Some(T),
}
```
Case analysis (through [pattern matching][12]) is used to get the value stored inside an
`Option<T>`. The `unwrap` method abstracts away the case analysis.

The `Option` trait defines combinators that are useful in get rid of case analysis. A great example
is `map` that maps a closure to the value inside of an `Option<T>`

```rust
let maybe_some_string = Some(String::from("Hello, World!"));
let maybe_some_len = maybe_some_string.map(|s| s.len());
assert_eq!(maybe_some_len, Some(13));
```
The `unwrap_or` combinator is useful for providing a default value when an `Option` value is
`None`. 

```rust
pub fn unwrap_or(self, def: T) -> T {
        match self {
            Some(x) => x,
            None => def,
        }
}
```
The `and_then` combinator makes is to compose distinct computations, essentially by chaining.
```rust
fn square(x: u32) -> Option<u32> { Some(x * x) }
assert_eq!(Some(2).and_then(square).and_then(square), Some(16));
```

The `Result` type is a richer version of `Option`. It expresses the **possibility of error**.
```rust
enum Result<T, E> {
   Ok(T),
   Err(E),
}
```
Just like `Option`, `Result` implements lots of combinators including `map`, `map_err`, `unwrap_or`
and `and_then`. The `Result` type also supports aliases when dealing with many references to one
`Result`. A common alias is `io::Result`.
```rust
pub type Result<T> = result::Result<T, Error>;
```

Multiple errors can be handled by having a custom `enum` to represent **one of many
possibilities**.
```rust
#[derive(Debug)]
enum CliError {
    Io(io::Error),
    Parse(num::ParseIntError),
}
```
The standard library defines two traits for error handing: `std::error::Error` and
`std::convert::From`. The first one for describing errors and the latter for conversion.
```rust
impl From<io::Error> for CliError {
    fn from(err: io::Error) -> CliError {
        CliError::Io(err)
    }
}
```

The [`try!`][13] macro is useful for encapsulating case analysis, control flow and error type
conversion.

## Jan-20-2018

**Title:** [Pretty State Machine Patterns in Rust][14]

A State Machine is any **machine** which has a set of **states** and **transitions** defined between them.
When we talk about a machine we’re referring to the abstract concept of something which does something.

States are a way to reason about where a machine is in its process. For example, we can think about a 
bottle filling machine. The machine is in a **waiting** state when it is waiting for a 
new bottle. Once it detects a bottle it moves to the **filling** state. Upon detecting the bottle is 
filled it enters the **done** state. After the bottle has left the machine we return to the **waiting** state.

```
  +---------+   +---------+   +------+
  |         |   |         |   |      |
  | Waiting +-->+ Filling +-->+ Done |
  |         |   |         |   |      |
  +----+----+   +---------+   +--+---+
       ^                         |
       +-------------------------+
```
When designing a state machine in Rust, we ideally want these characteristics:

* Can only be in one state at a time.
* Each state should able have its own associated values if required.
* Transitioning between states should have well defined semantics.
* It should be possible to have some level of shared state.
* Only explicitly defined transitions should be permitted.
* Changing from one state to another should consume the state so it can no longer be used.
* We shouldn't need to allocate memory for all states. 
* Any error messages should be easy to understand.
* We shouldn't need to resort to heap allocations to do this. 
* The type system should be harnessed to our greatest ability.
* As many errors as possible should be at compile-time.

An approach to achieve is use a combination of generics, `enum`s and shared values.

```rust
fn main() {
    let mut the_factory = Factory::new();
    the_factory.bottle_filling_machine = the_factory.bottle_filling_machine.step();
}

// This is our state machine for our Bottle Filling Machine
struct BFM<S> {
    shared_value: usize,
    state: S
}

// The following states can be the 'S' in StateMachine<S>
struct Waiting {
    waiting_time: std::time::Duration,
}

struct Filling {
    rate: usize,
}

struct Done;

// Our Machine starts in the 'Waiting' state.
impl BFM<Waiting> {
    fn new(shared_value: usize) -> Self {
        BFM {
            shared_value: shared_value,
            state: Waiting {
                waiting_time: std::time::Duration::new(0, 0),
            }
        }
    }
}

// The following are the defined transitions between states.
impl From<BFM<Waiting>> for BFM<Filling> {
    fn from(val: BFM<Waiting>) -> BFM<Filling> {
        BFM {
            shared_value: val.shared_value,
            state: Filling {
                rate: 1,
            }
        }
    }
}

impl From<BFM<Filling>> for BFM<Done> {
    fn from(val: BFM<Filling>) -> BFM<Done> {
        BFM {
            shared_value: val.shared_value,
            state: Done,
        }
    }
}

impl From<BFM<Done>> for BFM<Waiting> {
    fn from(val: BFM<Done>) -> BFM<Waiting> {
        BFM {
            shared_value: val.shared_value,
            state: Waiting {
                waiting_time: std::time::Duration::new(0, 0),
            }
        }
    }
}


// Here is we're building an enum so we can contain this state machine in a parent.
enum MachineWrapper {
    Waiting(BFM<Waiting>),
    Filling(BFM<Filling>),
    Done(BFM<Done>),
}

// Defining a function which shifts the state along.
impl MachineWrapper {
    fn step(mut self) -> Self {
        self = match self {
            MachineWrapper::Waiting(val) => MachineWrapper::Filling(val.into()),
            MachineWrapper::Filling(val) => MachineWrapper::Done(val.into()),
            MachineWrapper::Done(val) => MachineWrapper::Waiting(val.into()),
        };
        self
    }
}

// The structure with a parent.
struct Factory {
    bottle_filling_machine: MachineWrapper,
}

impl Factory {
    fn new() -> Self {
        Factory {
            bottle_filling_machine: MachineWrapper::Waiting(BFM::new(0)),
        }
    }
}
```

## Jan-21-2018

**Title:** [Finding Closure in Rust][15]

A **closure** is a function that can directly use variables from the scope in which it is defined.
This is often described as the closure _closing over_ or _capturing_ variables.

Rust has C++11 inspired closures using the trait system, allowing for:
- allocation-less statically dispatched closures
- choice to opt-in to type-erasure and dynamic dispatch

Syntactically, a closure in Rust is an anonymous function value defined similar to Ruby, with
pipes. 
```rust
fn main() {
    let mut v = [5, 4, 1, 3, 2];
    v.sort_by(|a, b| a.cmp(b)); // closure `|arguments...| body`
    assert!(v == [1, 2, 3, 4, 5]);
}
```
How do closures work? The definition of `Option::map` (an example closure implementation) is:
```rust
impl<X> Option<X> {
    pub fn map<Y, F: FnOnce(X) -> Y>(self, f: F) -> Option<Y> {
        match self {
            Some(x) => Some(f(x)),
            None => None
        }
    }
}
```

The `FnOnce` trait and its close cousins `Fn` and `FnMut` represent how the variables are captured by the
closure:

* `Fn`: **by reference**
* `FnMut`: **by mutable reference**
* `FnOnce`: **by value**

 By default, the compiler looks at the closure body to see how captured variables are used, and uses 
 that to infers how variables should be captured, that is deciding between `Fn`, `FnMut` and
 `FnOnce`.

 The `move` keyword is used to define an **escaping** closure, one that might leave the stack frame
 where it is created.
```rust
use std::thread;
thread::spawn(move || {
    // some work here
    });
```

The use of traits for closures allows one to opt-in into dynamic dispatch via trait objects:
```rust
let mut closures: Vec<Box<Fn()>> = vec![];

let text = "second";

closures.push(Box::new(|| println!("first")));
closures.push(Box::new(|| println!("{}", text)));
closures.push(Box::new(|| println!("third")));

for f in &closures {
    f(); // first / second / third
}
```

## Jan-22-2018-1

**Title:** [Why is Rust difficult?][16]

Rust is considered difficult to learn by many people. However, it's not necessarily a bad thing if
you get something in return for the investment.

Rust aims to **solve hard problems**; it's therefore harder than languages that solve simpler problems,
such as Lua.

> On the other hand, if you declare upfront that your language needs to be able to solve any hard 
> problem anyone thinks of, run fast and be safe to use then you’ll probably get your wish, but 
> it’ll hardly be simple.

The Rust language is **honest**. It does not abstract away problems from the programmer; a beginner
therefore has to grasp the complete complexity of the problem.

Rust is **different**. Features such as ownership, traits and lifetimes makes Rust stand out from
other languages.

The compiler is very **strict**. The Rust compiler requires your code to be typed, memory safe and
be free of data races. Even having dead code is an issue! 

> But at the same time, it makes learning it a bit harder, because it insists on you learning
> everything needed to write a good program. An average is not acceptable.

There is also a general **perception** that Rust is hard. 

Despite all this, it easy to quickly come up to speed, find the language interesting, become a
better programmer (the compile is a strict teacher) and meet some very nice people in the
community.

## Jan-22-2018-2

**Title:** [Macros][17]

Macros are a syntactic language feature. A macro use is expanded according to a macro definition.
They are like functions, however, their expansions happens entirely at compile time. In Rust this
after the parsing step of compilation.

Rust has a hygienic macro system - it preserves the scoping of the macro definition preventing
shadowing of variables during macro expansion.

Macros can be implemented as simple textual substitution, by manipulating tokens after lexing, or
by manipulating the AST (Abstract Syntax Tree) after parsing. This also depends on how macros are
defined. `macro_rules` in Rust allow for pattern matching of arguments in the macro definition.
Rust macros are lexed into tokens before expansion and parsed afterwards.

In a **procedural macro system**, a macro is defined as a program. When its use is encountered, the
macro is executed with the macro arguments as input. The macro use is then replaced by the result
of the execution.

```rust
proc_macro! foo(x) {  
    let mut result = String::new();
    for i in 0..10 {
        result.push_str(x)
    }
    result
}

fn main() {  
    let a = "foo!(bar)"; // Hand-waving about string literals.
}
```
will expand to
```rust
fn main() {  
    let a = "barbarbarbarbarbarbarbarbarbar";
}
```
> A procedural macro is a generalisation of the syntactic macros described so far. One could imagine 
> implementing a syntactic macro as a procedural macro by returning the text of the syntactic macro 
> after manually substituting the arguments.


## Jan-23-2018

**Title:** [Macros in Rust pt1][18]

Rust offers an array of macro features:
* `macro_rules!` macros
* procedural macros
* built-in macros such as `println!` and `#[derive()]`

`macro_rules!` lets you write syntactic macros based on pattern matching. They are used in a
function like style, the `!` distinguishes them from regular functions.

```rust
macro_rules! hello {  
    () => { println!("hello world"); }
}

fn main() {  
    hello!();
}
```
The above gets expanded into:

```rust
fn main() {  
    println!("hello world");
}
```

To illustrate pattern matching in macros:

This code snippet
```rust

macro_rules! my_print {  
    (foo <> $e: expr) => { println!("FOOOOOOOOOOOOoooooooooooooooooooo!! {}", $e); };
    ($e: expr) => { println!("{}", $e); };
    ($i: ident, $e: expr) => {
        let $i = {
            let a = $e;
            println!("{}", a);
            a
        };
    };
}

fn main() {  
    my_print!(x, 30 + 12);
    my_print!("hello!");
    my_print!(foo <> "hello!");
}
```
Outputs
```
42
hello!
FOOOOOOOOOOOOoooooooooooooooooooo!! hello!
```

## Jan-24-2018

**Title:** [Macros in Rust pt2][19]

Procedural macros are implemented using pure Rust code at the meta level. They are powerful since
the output can be anything you can express as an Abstract Syntax Tree.

At the moment procedural macros are used to write custom derive. They also need to be in their own
crate of the `proc-macro` crate type.

An example procedural macro implementation:

```rust
extern crate proc_macro;
extern crate syn;
#[macro_use]
extern crate quote;

use proc_macro::TokenStream;

#[proc_macro_derive(HelloWorld)]
pub fn hello_world(input: TokenStream) -> TokenStream {
    // Construct a string representation of the type definition
    let s = input.to_string();
    
    // Parse the string representation
    let ast = syn::parse_derive_input(&s).unwrap();

    // Build the impl
    let gen = impl_hello_world(&ast);
    
    // Return the generated impl
    gen.parse().unwrap()
}
```

The [`syn`][20] and [`qoute`][21] crates make it easier to write procedural macros.
> Syn is a parsing library for parsing a stream of Rust tokens into a syntax tree of Rust source code.

The `qoute!` macro is used to turn Rust syntax tree data structures into tokens of source code.


Error handling inside procedural macros is through the `panic!` macro.
```rust
fn impl_hello_world(ast: &syn::DeriveInput) -> quote::Tokens {
    let name = &ast.ident;
    // Check if derive(HelloWorld) was specified for a struct
    if let syn::Body::Struct(_) = ast.body {
        // Yes, this is a struct
        quote! {
            impl HelloWorld for #name {
                fn hello_world() {
                    println!("Hello, World! My name is {}", stringify!(#name));
                }
            }
        }
    } else {
        //Nope. This is an Enum. We cannot handle these!
       panic!("#[derive(HelloWorld)] is only defined for structs, not for enums!");
    }
}
```

This post also borrows from the [rusk book][22]

## Jan-25-2018

**Title:** [Macros in Rust pt3][23]

This post is about **macro hygiene in Rust**

A hygienic macro system preserves the scoping of a macro definition. The following examples
illustrate that:

```rust
macro_rules! foo {  
    () => {
        let x = 0;
    }
}
fn main() {  
    let mut x = 42;
    foo!();
    println!("{}", x);
}
```
The above code prints `42` as the output. Here the `x` defined in `main` and the one in `foo!` are
disjoint.

```rust
macro_rules! foo {  
    ($x: ident) => {
        $x = 0;
    }
}
fn main() {  
    let mut x = 42;
    foo!(x);
    println!("{}", x);
}
```
This prints `0`. `x` is passed in to the macro and then modified.

Rust is hygienic with respect to variables scoping, labels, feature gates and stability checks.

There are some limitations:
* hygiene only works on expression variables and labels.
* no hygiene is applied to lifetimes or type variables or types themselves.
* no hygiene with respect to privacy and safety i.e. in `unsafe` blocks.

The implementation of hygienic macro system is a bit complex.
Much of the work happens during the **macro expansion** and **name resolution** phases of Rust
code compilation.

The macro expansion phase is hygienic. During name resolution syntactic names are resolved into
definitions. For unhygienic system, this resolutions is basically string equality. In Rust
however, we consider identifiers(names) and a syntax context. To check if two identifiers are
equal, they are resolved in the respective syntax contexts and the results compared.
The syntax context is added to an identifier during the macro expansion phase.

The macro hygiene algorithm used in Rust comes from the [Macros That Work Together][24] paper. This
mtwt algorithm is used during macro expansion. The whole syntax tree of a crate is walked,
expanding macro uses and applying the algorithm to all identifiers.

Mtwt has two key concepts - marking and renaming. Marking is applied when we expand a macro and
renaming when we enter a new scope. A syntax context under mtwt consists of a series of marks and
renames.

## Jan-26-2018

**Title:** [Macros in Rust pt4][25]

This post is about stuff around the import, export and modularisation of macros.

The order of items matters for macros. You can only refer to a macro after it is declared.

Macros defined inside a module cannot be used unless the module is annotated with `#[macro_use]`. Macros defined before and outside a module can be used without importation.

Macros are encapsulated by crates - they must be explicitly imported and exported. When importing macros from a crate, you annotate `extern crate` with `#[macro_use]`.

Program representations that matter in macros:
* source text
* token trees
* abstract syntax tree

**Source text** is the text passed to the compiler. It's stored in a codemap and is immutable. Once it's lexed, it hardly used by the compiler. It's main use is in error messages.

**Lexing** is the first stage of compilation where source text is transformed into **tokens**.

Parsing a token tree gives an **abstract syntax tree(AST)**. An AST is a tree of nodes representing the source text in a concrete way. 

The three main phases of compilation are:
* parsing and expansion
* analysis, and
* code generation

**libsyntax** implements lexing, parsing and macro expansion. This is purely syntactic. The result of this is an AST. During lexing and parsing, macro use is left as is in the AST. The macro expansion phase walks the AST and performs substitutions while maintaining hygiene.

**Spans** (_a span identifies a section of code in the source text_)  are used to tracing macros in order to highlight source code in error messages. Due to the possibility of nested macros, the are also **expansion traces** table that holds a trace of each macro expansion.


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
[11]: http://blog.burntsushi.net/rust-error-handling/
[12]: https://doc.rust-lang.org/book/second-edition/ch18-03-pattern-syntax.html
[13]: https://doc.rust-lang.org/std/macro.try.html
[14]: https://hoverbear.org/2016/10/12/rust-state-machine-pattern/
[15]: http://huonw.github.io/blog/2015/05/finding-closure-in-rust/
[16]: https://vorner.github.io/difficult.html
[17]: https://www.ncameron.org/blog/macros/
[18]: https://www.ncameron.org/blog/macros-in-rust-pt1/
[19]: https://www.ncameron.org/blog/macros-in-rust-pt2/
[20]: https://docs.rs/syn/0.12.10/syn/
[21]: https://docs.rs/quote/0.4.2/quote/
[22]: https://doc.rust-lang.org/book/first-edition/procedural-macros.html
[23]: https://ncameron.org/blog/macros-in-rust-pt3/
[24]: https://www.cs.utah.edu/plt/publications/jfp12-draft-fcdf.pdf
[25]: https://ncameron.org/blog/macros-in-rust-pt4/
