You probably should avoid putting lifetime parameters on traits
===============================================================

Having a lifetime parameter attached to a trait makes it much easier to accidentally create an unusable trait - I only realized after a lot of hardship fighting with the borrow checker.

Here is an example I encountered recently.

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

It's a head-scratcher, isn't it? Indeed, this error took me quite sometime to decipher, but I think I can explain what it actually means.

First, let me put the elided lifetime parameter back into `fn lock`:

```rust
fn lock<'a>(&'a self) -> Self::Guard<'a>;
```

When we call `t.lock()`, the compiler must figure out a lifetime to assign to `'a`. This lifetime is used in the return type `Self::Guard<'a>`, and all the compiler know about `Self::Guard<'a>` is that it implements `Locked<'a>`. `Locked<'a>` is a trait, which means it is invariant w.r.t lifetime `'a`. (If you don't know what variance is, see [here](https://doc.rust-lang.org/reference/subtyping.html), and [here](https://doc.rust-lang.org/nomicon/subtyping.html).)

And here lies the problem, in the context of our function `test`:

```rust
fn test<'lifetime_of_t, T: Lockable>(t: &'lifetime_of_t T) {
    let x = t.lock();
    // ...
}
```

This mean the type of `x` is actually `T::Guard<'lifetime_of_t>`, which implements `Locked<'lifetime_of_t>`. And since `Locked<'_>` is invariant w.r.t the lifetime, `Guard` implementing `Locked<'lifetime_of_t>` doesn't mean it implements `Locked<'shorter>` for any `'shorter` lifetime. When we call `x.iter()`, it can only return `Locked::Iter: 'lifetime_of_t`. Which means `x` is actually borrowed for `'lifetime_of_t`! It's borrowed for a lifetime that is actually longer than its *own* lifetime! (I think it's fair to say rustc's diagnostic here can use some polish.)

And once we figured this out, the solution is simple. One way is do away the lifetime parameter on `Locked`. For example, we can make it:

```rust
type Guard<'a>: Locked + 'a;
```

The whole example:

```rust
trait Lockable {
    // Unlike `Trait<'a>`, `Trait + 'a` is covariant w.r.t. `'a`.
    type Guard<'a>: Locked + 'a where Self: 'a;
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

And looking back, it's now clear our `Locked<'a>::iter` function was wrong: the `Locked` iterator yields items that lives as long as the `Lockable` type, yet if you think about the semantic of a lock, `Locked` really should only yield item that live as long as the `Guard`.

(If you really want to keep the lifetime on `Locked<'a>`, there is a way:
```rust
trait Lockable {
    type Guard<'a>: Locked<'a> where Self: 'a;
    fn lock(&self) -> Self::Guard<'_>;
}
trait Locked<'a> {
    type Iter<'b>: Iterator<Item = &'b u32> + 'b where 'a: 'b, Self: 'b;
    fn iter<'b>(&'b self) -> Self::Iter<'b> where 'a: 'b;
}
```
I am not trying to show lifetime parameter is absolutely not workable, I just want to say it often is an unexpected trap for new comers. Plus, there are other ways this invariance can be annoying.)

I ended up not needing such a trait at all, but I think this is a really good example why having lifetime parameter on a trait might not be a very good idea. In fact, most of the traits in `std` doesn't have an explicit lifetime parameter. There are cases where the use of a lifetime parameter can be justified, `serde::de::Deserialize` is such an example. But in general, you probably should think twice before using it.
