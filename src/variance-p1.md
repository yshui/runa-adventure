You probably should avoid putting lifetime parameters on traits
===============================================================

Having a lifetime parameter attached to a trait makes the trait so much harder to work with - I only realized after a lot of hardship fighting with the borrow checker.

Here is an example I recently encountered.

I tried to make my code generic over some sort of "lockable" types. Like a `Mutex<T>`, which you can call `.lock()` on, and after you locked it there is a set of methods you can call on `T`. I didn't want to lock the user into a predetermined `Mutex` type, so I came up with this trait:

```rust
trait Lockable {
    type Guard<'a>: Locked<'a> where Self: 'a;
    fn lock(&self) -> Self::Guard<'_>;
}
```

(Quick side note, if you don't know, this `where Self: 'a` bound is forced by the compiler, whether it's actually needed or not. You can see [this issue](https://github.com/rust-lang/rust/issues/87479) for more details.)

And for the `Locked<'a>`, it implements a set of predefined methods. Naively, I wrote down this trait for it:

```rust
trait Locked<'a> {
    type Iter: Iterator<Item = &'a u32> + 'a;

    /// A scapegoat example to illustrate the problem I am going to have.
    /// This is not that weird a method to have - imagine a `Mutex<HashMap>`,
    /// after locking it you might want to iterate over its keys.
    fn iter(&'a self) -> Self::Iter;
}
```

OK, this seems innocent enough, what would happen if we try to use it?

```rust
fn test<T: Lockable>(t: &T) {
    let x = t.lock();
    x.iter();
}
```

Should work, right?

No. Instead, we get this very confusing error:

```
error[E0597]: `x` does not live long enough
  --> src/lib.rs:19:5
   |
19 |     x.iter();
   |     ^^^^^^^^ borrowed value does not live long enough
20 | }
   | -
   | |
   | `x` dropped here while still borrowed
   | borrow might be used here, when `x` is dropped and runs the destructor for type `<T as Lockable>::Locked<'_>`
```

What is going on here? If we take the error message at face value, it doesn't make a lot of sense: `x` is dropped while still borrowed, OK. Why was it still borrowed? Because it was used later. For what? For its destructor, because it was dropped later. What?

It's a head-scratcher, isn't it? Indeed, this error took me quite sometime to decipher, but I think I can explain what this actually means.

First, lets put the elided lifetime parameter back into `fn lock`:

```rust
fn lock<'a>(&'a self) -> Self::Guard<'a>;
```

When we call `t.lock()`, the compiler must figure out a lifetime to assign to `'a`. This lifetime is used in the return type `Self::Guard<'a>`, and all the compiler know about `Self::Guard<'a>` is that it implements `Locked<'a>`. `Locked<'a>` is a trait, which means it is invariant w.r.t lifetime `'a`. (If you don't know what variance is, see [here](https://doc.rust-lang.org/reference/subtyping.html), and [here](https://doc.rust-lang.org/nomicon/subtyping.html).)

And here lies the problem, the compiler doesn't have the flexibility to lengthen or shorten this lifetime, so it use `'a` from `&'a self` as is. In the context of our function `test`:

```rust
fn test<'lifetime_of_t, T: Lockable>(t: &'lifetime_of_t T) {
    let x = t.lock();
    // ...
}
```

This mean the type of `x` is actually `T::Guard<'lifetime_of_t>`, which implements `Locked<'lifetime_of_t>`. And when we call `x.iter()`, it returns `Locked::Iter: 'lifetime_of_t`. Which means `x` is actually borrowed for `'lifetime_of_t`! It's borrowed for a lifetime that is actually *longer* than its own lifetime! (I think it's fair to say rustc's diagnostic here can use some polish.)

And once we figured this out, the solution is simple. We need to make the return type of `lock()` covariant w.r.t the lifetime of `&self`. For example, we can make it:

```rust
fn lock<'a>(&'a self) -> &'a Self::Guard;
```

Of course, the lock guard usually won't be as simple as just a reference. More practically, we can use something like this:

```rust
trait Lockable {
    type Target: Locked;
    // Unlike `Trait<'a>`, `Trait + 'a` is covariant w.r.t. `'a`.
    type Guard<'a>: std::ops::DerefMut<Target = Self::Target> + 'a where Self: 'a;
    fn lock(&self) -> Self::Guard<'_>;
}
trait Locked {
    type Iter<'a>: Iterator<Item = &'a u32> + 'a where Self: 'a;
    fn iter(&self) -> Self::Iter<'_>;
}
fn test<T: Lockable>(t: &T) {
    let x = t.lock();
    x.iter();
}
```

I ended up not needing such a trait at all, but I think this is a really good example why having lifetime parameter on a trait might not be a very good idea. In fact, most of the traits in `std` doesn't have an explicit lifetime parameter. There are cases where the use of a lifetime parameter can be justified, `serde::de::Deserialize` is such an example. But in general, you probably should think twice before using it.
