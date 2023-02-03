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
