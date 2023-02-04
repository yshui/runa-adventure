# The Rust borrow checker is annoying

OK, quick wayland primer. One of the central concepts in wayland is object. Each client connected to the wayland server has a set of objects bound to it. Client can invoke functions on those objects by sending a message to server with the right object ID.

Let's have a look at how we might implement this in Rust. First of all, a set of objects indexed via their object IDs, seems easy:

```rust
let objects: HashMap<u32, Object> = HashMap::new();
```

Now let's handle the messages:

```rust
let object_id = get_message_object_id();
let object = objects.get(message.object_id).unwrap();
// To decode a message, we need to know the type of the object.
let message = decode_message(object.interface);
object.handle_message(message);
```

Looks straightforward. But problem arises when our `handle_message` implementation needs access to the set of objects. For example, `wl_compositor.create_surface` creates new surfaces, so it needs to be able to insert into `objects`.

But we can't change `handle_message(message)` to `handle_message(&mut objects, message)`, because we are already borrowing `objects` immutably via `object`.

And it's not as simple as putting `objects` in a `RefCell` either:

```rust
let objects_borrow = objects.borrow();
let object = objects_borrow.get(message.object_id).unwrap();
object.handle_message(objects.borrow_mut(), message); // Fails, because a borrow already exists
```

Similarly, it's also a problem if `handle_message` wants to modify an object inside `objects`, simply because we can't pass a `&mut objects` to `handle_message`, so there is no way of getting mutable references to objects. This forces the object to implement interior mutably. Which isn't the end of the world, but also not very nice.

Hopefully you can see this is a genuine problem, not simply because I am terrible at writing Rust. Now let's look at some solutions I've come up with.

## Solutions

### Solution 1

Make `Object` `Rc<Object>`, i.e.

```rust
let objects: HashMap<u32, Rc<Object>> = HashMap::new();
```

This solves the first problem - because we can `clone` the `Rc<Object>` and thus don't need to keep borrowing `objects` - but doesn't solve the second.

This also make the lifetime of `Object`s unpredictable. Previously, objects will be freed when `objects` is dropped; now because they are behind `Rc`, they can hang around indefinitely.

### Solution 2

Make `handle_message` not take `self`, i.e.

```rust
Object::handle_message(&mut objects, message);
```

This solves both problems, but if `handle_message` needs access to the object itself, it will have to get it again from `objects`, essentially doing extra `HashMap` lookups.

### Solution 3

Cache modifications to `objects` while it was borrowed, i.e.

```rust
let mut objects_modifications = ObjectsModification::new();
object.handle_message(&mut objects_modifications, message);
objects.apply(objects_modifications);
```

This solves the first problem, but can be a little counter-intuitive, as modifications made to the set of objects won't be immediately visible.

And this doesn't solve the second problem.

## Conclusion

Which one is the best solution? I don't know. I am inclined to go with solution 2 as it's the least bad, but that's not the main point here - I wanted to present a case where the Rust's lifetime mechanism meets a real world use case, and the problems that came with it. Although not an unsolvable problem, all of the solutions presented have some awkwardness to them, and that is the kind of trade-off one might face when working with Rust.

I am not an expert on Rust, so _please_ let me now if you think there is a better solution that I missed.


