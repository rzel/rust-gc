# rust-gc
[![Build Status](https://travis-ci.org/Manishearth/rust-gc.svg?branch=master)](https://travis-ci.org/Manishearth/rust-gc)

Simple tracing (mark and sweep) garbage collector for Rust

Works, but still under construction.

The design and motivation is illustrated in [this blog post](http://manishearth.github.io/blog/2015/09/01/designing-a-gc-in-rust/), with a sketch of the code [in this gist](https://gist.github.com/mystor/fa1141bfb30643289597).

There is [another post](http://blog.zhenzhang.me/2016/02/18/cgc.html) about the initial design of `cgc`, its experimental concurrent branch.

## How to use
To include in your project, add the following to your Cargo.toml:

```
[dependencies]
gc = "*"
gc_plugin = "*"
```

This can be used pretty much like `Rc`, with the exception of interior mutability.

While this can be used pervasively, this is intended to be used only when needed, following Rust's "pay only for what you need" model. Avoid using `Gc` where `Rc` or `Box` would be equally usable.

Types placed inside a `Gc` must implement `Trace`. The easiest way to do this is to use the existing plugin:

```rust
#![feature(plugin, custom_derive)]

#![plugin(gc_plugin)]
extern crate gc;

use gc::Gc;

#[derive(Trace)]
struct Foo {
  x: Gc<Foo>,
  y: u8,
  // ...
}

// now, `Gc<Foo>` may be used
```

For types defined in the stdlib, please file an issue on this repository (use the `unsafe_ignore_trace` method shown below to make things work in the meantime).

Note that `Trace` is only needed for types which transitively contain a `Gc`, if you are sure that this isn't the case, you may use the `unsafe_empty_trace!` macro on your types. Alternatively, use the `#[unsafe_ignore_trace]` annotation on the struct field. Incorrect usage of `unsafe_empty_trace` and `unsafe_ignore_trace` may lead to unsafety.

```rust
#![feature(plugin, custom_derive, custom_attribute)]

#![plugin(gc_plugin)]
extern crate gc;

extern crate bar;

use gc::Gc;
use bar::Baz;

#[derive(Trace)]
struct Foo {
  x: Gc<Foo>,
  #[unsafe_ignore_trace]
  y: Baz, // we are assuming that `Baz` doesn't contain any `Gc` objects
  // ...
}
```

To use `Gc`, simply call `Gc::new`:

```rust
let x = Gc::new(1_u8);
let y = Gc::new(Box::new(Gc::new(1_u8)));

#[derive(Trace)]
struct Foo {
 a: Gc<u8>,
 b: u8
}

let z = Gc::new(Foo {a: x.clone(), b: 1})
```

Calling `clone()` on a `Gc` will create another garbage collected reference to the same object. For the most part, try to use borrowed references to the inner value instead of cloning the `Gc` wherever possible -- `Gc` implements `Deref` and is compatible with borrowing.

`Gc` is an immutable container. Much like with `Rc`, to get mutability, we must use a cell type. The regular `RefCell` from the stdlib will not work with `Gc` (as it does not implement `Trace`), instead, use `GcCell`. `GcCell` behaves very similar to `RefCell`, except that it internally helps keep track of GC roots.

```rust
#[derive(Trace)]
struct Foo {
 cyclic: GcCell<Option<Gc<Foo>>>,
 data: u8,
}

let foo1 = Gc::new(Foo {cyclic: GcCell::new(None), data: 1});
let foo2 = Gc::new(Foo {cyclic: GcCell::new(Some(foo1.clone())), data: 2});
let foo3 = Gc::new(Foo {cyclic: GcCell::new(Some(foo2.clone())), data: 3});
*foo1.cyclic.borrow_mut() = Some(foo3.clone());
```


## Known issues

- Destructors should not access `Gc`/`GcCell` values. We may add finalizers in the future, but we'd need to figure out a way to prevent this.
- There needs to be a better story for cross-crate deriving, but we may have to wait for the middle IR to land to work on this.
- The current GC is not concurrent and the GCed objects are confined to a thread. There is an experimental concurrent collector [in this pull request](https://github.com/Manishearth/rust-gc/pull/6).


## Related projects
* [RuScript](https://github.com/izgzhen/RuScript): Uses single-thread `rust-gc` to allocate memory for various objects
