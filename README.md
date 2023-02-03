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
