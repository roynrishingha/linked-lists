<h1 align="center">LINKED LISTS<h1>

## NOTES

### ITERATOR

```rs
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

`type Item` is declaring that every implementation of Iterator has an associated type called Item. 
In this case, this is the type that it can spit out when you call next.

The reason Iterator yields `Option<Self::Item>` is because the interface coalesces the `has_next` and `get_next` concepts.
When you have the next value, you yield `Some(value)`, and when you don't you yield `None`. 
This makes the API generally more ergonomic and safe to use and implement, while avoiding redundant checks and logic between `has_next` and `get_next`.

#### 3 DIFFERENT KINDS OF ITERATOR EACH COLLECTION SHOULD ENDEAVOUR TO IMPLEMENT :

1. **IntoIter - T**
2. **IterMut - &mut T**
3. **Iter - &T**

### Lifetime

A lifetime is the name of a region (~block/scope) of code somewhere in a program.
#### Lifetimes solve both of these problems:

- Holding a pointer to something that went out of scope
- Holding a pointer to something that got mutated away

```rs
// Only one reference in input, so the output must be derived from that input
fn foo(&A) -> &B; // sugar for:
fn foo<'a>(&'a A) -> &'a B;

// Many inputs, assume they're all independent
fn foo(&A, &B, &C); // sugar for:
fn foo<'a, 'b, 'c>(&'a A, &'b B, &'c C);

// Methods, assume all output lifetimes are derived from `self`
fn foo(&self, &B, &C) -> &D; // sugar for:
fn foo<'a, 'b, 'c>(&'a self, &'b B, &'c C) -> &'a D;
```

## PERSISTENT SINGLY LINKED LIST

### LAYOUT

```rs
list1 = A -> B -> C -> D
list2 = tail(list1) = B -> C -> D
list3 = push(list2, X) = X -> B -> C -> D
```

**We want the memory to look like this:**

```
list1 -> A ---+
              |
              v
list2 ------> B -> C -> D
              ^
              |
list3 -> X ---+
```

#### THE PROBLEM

This just can't work with `Boxes`, because ownership of `B` is shared. Who should free it? If I drop `list2`, does it free `B`? With boxes we certainly would expect so!

Functional languages — and indeed almost every other language — get away with this by using garbage collection. 
With the magic of garbage collection, B will be freed only after everyone stops looking at it.

Rust doesn't have anything like the garbage collectors these languages have. 
They have tracing GC, which will dig through all the memory that's sitting around at runtime and figure out what's garbage automatically. 
Instead, all Rust has today is `reference counting`.
Reference counting can be thought of as a very simple GC. 
For many workloads, it has significantly less throughput than a tracing collector, 
and it completely falls over if you manage to build cycles. 

#### THE SOLUTION - Rc

`Rc` is just like Box, but we can duplicate it, 
and its memory will only be freed when all the Rc's derived from it are dropped.

Unfortunately, this flexibility comes at a serious cost:
we can only take a shared reference to its internals. This means 
we can't ever really get data out of one of our lists, nor can we mutate them.

#### NOTE

Note that we can't implement `IntoIter` or `IterMut` for this type.
We only have shared access to elements.

### Arc

One reason to use an immutable linked list is to share data across threads.
In order to be thread-safe, we need to fiddle with reference counts atomically. 
Otherwise, two threads could try to increment the reference count, and only one would happen. 
Then the list could get freed too soon!

In order to get thread safety, we have to use Arc. 
Arc is completely identical to Rc except for the fact that reference counts are modified atomically. 
This has a bit of overhead if you don't need it, so Rust exposes both.

#### how do we know if a type is thread-safe or not? Can we accidentally mess up?


No! You can't mess up thread-safety in Rust!
Rust models thread-safety in a first-class way with two traits:
`Send` and `Sync`.
That is, if `T` is `Sync`, `&T` is `Send`.

- A type is Send if it's safe to move to another thread. 
- A type is Sync if it's safe to share between multiple threads.

Almost every type is Send and Sync. 
Most types are Send because they totally own their data.
Most types are Sync because the only way to share data across threads is to put them behind a shared reference, which makes them immutable!

However there are special types that violate these properties: 
those that have interior mutability.

Interior mutability types violate this: 
they let you mutate through a shared reference.
There are two major classes of interior mutability:
- cells, which only work in a single-threaded context;
- locks, which work in a multi-threaded context. 

For obvious reasons, cells are cheaper when you can use them.
There's also atomics, which are primitives that act like a lock.

#### So what does all of this have to do with Rc and Arc?

Well, they both use interior mutability for their reference count.
Worse, this reference count is shared between every instance! 
- Rc just uses a cell, which means it's not thread safe. 
- Arc uses an atomic, which means it is thread safe. 

Of course, you can't magically make a type thread safe by putting it in Arc.
Arc can only derive thread-safety like any other type.

## BAD BUT SAFE DOUBLY LINKED DEQUE

This means each node has a pointer to the previous and next node.
Also, the list itself has a pointer to the first and last node. 
This gives us fast insertion and removal on both ends of the list.

Each node should have exactly two pointers to it. 
Each node in the middle of the list is pointed at by its predecessor and successor, while the nodes on the ends are pointed to by the list itself.

### LAYOUT

The key to our design is the `RefCell` type. 
The heart of `RefCell` is a pair of methods:

```rs
fn borrow(&self) -> Ref<'_, T>;
fn borrow_mut(&self) -> RefMut<'_, T>;
```

The rules for `borrow` and `borrow_mut` are exactly those of `&` and `&mut`:
you can call `borrow` as many times as you want, 
but `borrow_mut` requires exclusivity.

Rather than enforcing this statically, RefCell enforces them at runtime.
If you break the rules, RefCell will just panic and crash the program.

### RefCell

Shareable mutable containers.

Values of the Cell<T> and RefCell<T> types may be mutated through shared references (i.e. the common &T type), whereas most Rust types can only be mutated through unique (&mut T) references. We say that Cell<T> and RefCell<T> provide 'interior mutability', in contrast with typical Rust types that exhibit 'inherited mutability'.

Cell types come in two flavors: Cell<T> and RefCell<T>. Cell<T> provides get and set methods that change the interior value with a single method call. Cell<T> though is only compatible with types that implement Copy. For other types, one must use the RefCell<T> type, acquiring a write lock before mutating.

RefCell<T> uses Rust's lifetimes to implement 'dynamic borrowing', a process whereby one can claim temporary, exclusive, mutable access to the inner value. Borrows for RefCell<T>s are tracked 'at runtime', unlike Rust's native reference types which are entirely tracked statically, at compile time. Because RefCell<T> borrows are dynamic it is possible to attempt to borrow a value that is already mutably borrowed; when this happens it results in thread panic.


#### When to choose interior mutability?

The more common inherited mutability, where one must have unique access to mutate a value, is one of the key language elements that enables Rust to reason strongly about pointer aliasing, statically preventing crash bugs. 
Because of that, inherited mutability is preferred, and interior mutability is something of a last resort. 
Since cell types enable mutation where it would otherwise be disallowed though, there are occasions when interior mutability might be appropriate, or even must be used:
- Introducing inherited mutability roots to shared types.
- Implementation details of logically-immutable methods.
- Mutating implementations of `Clone`.

#### Inherited mutability roots to shared types

Shared smart pointer types, including Rc<T> and Arc<T>, provide containers that can be cloned and shared between multiple parties. Because the contained values may be multiply-aliased, they can only be borrowed as shared references, not mutable references. Without cells it would be impossible to mutate data inside of shared boxes at all!

It's very common then to put a RefCell<T> inside shared pointer types to reintroduce mutability:

```rs
use std::collections::HashMap;
use std::cell::RefCell;
use std::rc::Rc;

fn main() {
    let shared_map: Rc<RefCell<_>> = Rc::new(RefCell::new(HashMap::new()));
    shared_map.borrow_mut().insert("africa", 92388);
    shared_map.borrow_mut().insert("kyoto", 11837);
    shared_map.borrow_mut().insert("piccadilly", 11826);
    shared_map.borrow_mut().insert("marbles", 38);
}
```

**NOTE:**
This example uses `Rc<T>` and not `Arc<T>`. `RefCell<T>`s are for single-threaded scenarios. 
Consider using `Mutex<T>` if you need shared mutability in a multi-threaded situation.

## OK BUT UNSAFE SINGLY LINKED QUEUE

### LAYOUT

So what's a singly-linked queue like? 
Well, when we had a singly-linked stack we pushed onto one end of the list, and then popped off the same end. 
The only difference between a stack and a queue is that a queue pops off the other end. 
So from our stack implementation we have:

```
input list:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)

stack push X:
[Some(ptr)] -> (X, Some(ptr)) -> (A, Some(ptr)) -> (B, None)

stack pop:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)
```

### UNSAFE RUST

The long and the short of it is that every language is actually unsafe as soon as you allow calling into other languages, because you can just have C do arbitrarily bad things. Yes: Java, Python, Ruby, Haskell... everyone is wildly unsafe in the face of Foreign Function Interfaces (FFI).

Rust embraces this truth by splitting itself into two languages: Safe Rust, and Unsafe Rust.

Unsafe Rust is a superset of Safe Rust. It's completely the same as Safe Rust in all its semantics and rules, you're just allowed to do a few extra things that are wildly unsafe and can cause the dreaded Undefined Behaviour that haunts C.

### MIRI

An experimental interpreter for Rust's mid-level intermediate representation (MIR). It can run binaries and test suites of cargo projects and detect certain classes of undefined behavior, for example:
- Out-of-bounds memory accesses and use-after-free
- Invalid use of uninitialized data
- Violation of intrinsic preconditions (an unreachable_unchecked being reached, calling copy_nonoverlapping with overlapping ranges, ...)
- Not sufficiently aligned memory accesses and references
- Violation of some basic type invariants (a bool that is not 0 or 1, for example, or an invalid enum discriminant)
- Experimental: Violations of the Stacked Borrows rules governing aliasing for reference types
- Experimental: Data races (but no weak memory effects)

On top of that, Miri will also tell you about memory leaks: when there is memory still allocated at the end of the execution, and that memory is not reachable from a global static, Miri will raise an error.
However, be aware that Miri will not catch all cases of undefined behavior in your program, and cannot run all programs.

#### **TL;DR:** 

It interprets your program and notices if you break the rules at runtime and Do An Undefined Behaviour. This is necessary because Undefined Behaviour is generally a thing that happens at runtime. If the issue could be found at compile time, the compiler would just make it an error!

```sh
cargo +nightly miri test
```

### MEMORY MODEL

- Rust conceptually handles reborrows by maintaining a "borrow stack"
- Only the one on the top of the stack is "live" (has exclusive access)
- When you access a lower one it becomes "live" and the ones above it get popped
- You're not allowed to use pointers that have been popped from the borrow stack
- The borrowchecker ensures safe code code obeys this
- Miri theoretically checks that raw pointers obey this at runtime

