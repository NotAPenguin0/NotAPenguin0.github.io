---
layout: post
title: "Writing an event system in Rust"
categories: [Rust]
tags: [rust]
author: NotAPenguin
---

## Introduction

While working on my current Rust project, I eventually started to have a need for a better way to handle individual systems communicating
with each other. Passing around references and calling functions manually when needed just didn't do it anymore. I decided to handle this
by using an event-based approach. The plan was to implement an `EventBus` struct that would be used as the central interface
into the event system. This `EventBus` can be used to publish events and register new listeners. The final interface will look a little
like this:

```rust
struct MyEvent;
impl Event for MyEvent {}

struct MySystem {
  pub counter: u64,
}

impl System for MySystem {
  fn initialize(bus: &EventBus, system: &SyncSystem<Self>) {
    bus.subscribe(system, handle_event);
  }
}

fn handle_event(system: &mut MySystem, event: &MyEvent) {
  system.counter += 1;
}
```

## What is an event?

The first question to ask ourselves is what sort of types we can use as events. Ideally, we impose as few restrictions as possible. With that in
mind, an initial definition of an `Event` could be ...

```rust
pub trait Event {}
```

... nothing! This may seem redundant, but if we want to add properties to events later, such as a `Result` type that is returned from
its handler, we can easily do this without having to add trait bounds everywhere.

## Systems and event handlers

A *System* is some object that listens to events on the event bus. Once again, we'll start by defining an empty trait that systems
must implement.

```rust 
pub trait System {}
```

Easy! Next up, let's define what it takes for something to be an event handler for a system. 

```rust
pub trait Handler<S: System, E: Event> {
  fn handle(&self, system: &mut S, event: &E);
}
```

So, a handler is something that we can call `handle()` on, and give it the system's state and an event to handle. Note that even though the
trait doesn't actually use the fact that `S` is a `System` and `E` an `Event`, for completeness' sake it makes sense to add this anyway.
An example of something that could implement `Handler` is the following function:

```rust
fn handle_my_event(system: &mut MySystem, event: &MyEvent) {
  println!("Received event!");
}
```

With that in mind, it makes sense to implement `Handler` for every function with a signature that matches `(&mut S, &E) -> ()` 

```rust
impl<S: System, E: Event, F: Fn(&mut S, &E)> Handler<S, E> for F {
  fn handle(&self, system: &mut S, event: &E) {
    self(system, event)
  }
}
```

The next sensible thing to do is to create some sort of `InternalSystem` type that holds the system's state and it's handlers.

```rust
struct InternalSystem<S: System> {
  state: S,
  handlers: Vec<???>
}
```

Except it turns out it's not that simple. Because each `Handler` accepts completely different generic parameters, there's no common
interface to them, and we can't store them this easily. Enter our new friend: Type erasure.

### Intermezzo: Type erasure

Runtime type erasure in Rust all revolves around `dyn` trait objects. They are comparable to abstract classes or interfaces in languages
like Java and C++. When you want to store a collection of different types that implement the same trait, you can use a 
`Vec<Box<dyn Trait>>`. You can then use `Trait`'s methods on these, and dynamic dispatch will take care of calling the correct
implementation.

However, we don't have such a common trait here, so that won't work. Thankfully, the standard library provides us with the `Any` trait,
which is, unsurprisingly, implemented for (not quite) every type. The final component we will need is `TypeId`. This is a unique, 
hashable identifier for each type. We can now use this to create our collection:

```rust
struct ErasedStorage {
  items: HashMap<TypeId, Box<dyn Any>>,
}

impl ErasedStorage {
  pub fn put<T: 'static>(&mut self, item: T) {
    self.items.insert(TypeId::of::<T>(), Box::new(item));
  }
  
  pub fn get<T: 'static>(&self) -> Option<&T> {
    let any = self.items.get(&TypeId::of::<T>());
    // Downcast back to concrete type    
    any.map(|value| value.downcast_ref::<T>().unwrap())
  }
}
```

The `'static` bound on `T` is required by the bounds on `Any` and `TypeId::of`. This just means that our erased storage
can't store any type that holds non-`'static` references. For us, this won't be a major limitation.

### Storing handlers

Great, now that we know how to store any type inside an erased container, we can freely store our `dyn Handler<S, E>` objects, right?
Well, not exactly. Currently, the `put` and `get` functions have an implicit `T: Sized` bound. However, `dyn Trait` objects do not
have a known size at compile time. Unfortunately, relaxing this bound with `T: ?Sized` is not possible, because `Any::downcast_ref::<T>`
requires `T: Sized`.

> Oh, I know, I'll just `std::mem::transmute` the `Box<dyn Any>` to `Box<T>`. They're both pointers, and I know that it really was a `T`,
> initially, so this should work!

![Clueless](/assets/img/ferris_clueless.png)

So, let's try it on the Rust playground and run it through Miri 
[here](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=c477bcd07ad30ef0f90a59d12ac031cc).

```rust
// Assumes T is a `dyn Trait` type, and the original type inside the box implements `Trait`.
unsafe fn to_dyn_trait<T: ?Sized>(erased: &Box<dyn Any>) -> &Box<T> {
     std::mem::transmute::<_, &Box<T>>(erased)
}
```

> Error: Undefined Behavior: `dyn` call on a pointer whose vtable does not match its type
{: .prompt-danger }

*Oh. Right. Okay*. What's going on here? As it turns out, if `T` is not `Sized`, `Box<T>` stores a vtable on the stack.
This means that the memory layout of `Box<T>` depends on `T`!
Luckily for us, there is a way around this. Using the nightly channel of the Rust compiler, 
we can enable the `thin_box` feature and use `ThinBox<T>` instead. This struct is similar to `Box<T>`, except the 
vtable and other metadata is stored in another heap allocation, meaning its memory layout is independent of `T`, 
and we can "safely" transmute like we just tried to. 
Armed with this knowledge, we can extend `ErasedStorage` with two new methods:

```rust
struct ErasedStorage {
  items: HashMap<TypeId, Box<dyn Any>>,
  dyn_items: HashMap<TypeId, ThinBox<dyn Any>>,
}

impl ErasedStorage {
  pub fn put_dyn<T: ?Sized + 'static>(&mut self, item: impl Unsize<T>) { ... }
  
  pub fn get_dyn<T: ?Sized + 'static>(&self) -> Option<&T> {
    let any = self.dyn_items.get(&TypeId::of::<T>());
    // SAFETY: Using `TypeId` as a key guarantees that the type we transmute 
    // to is the same as original type used to construct this ThinBox.
    any.map(|any| unsafe { std::mem::transmute::<_, &ThinBox<T>>(any) }.deref() )
  }
}
```

Using the `Unsize` trait, we can ensure that if we insert a struct of type `T` as a `dyn Foo`, that `T: Foo`.

Let's go back to the `InternalSystem` struct from before and complete it.
```rust
struct InternalSystem<S: System> {
  state: S,
  handlers: ErasedStorage,
}

impl<S: System + 'static> InternalSystem<S> {
  pub fn try_handle<E: Event + 'static>(&mut self, event: &E) {
    // Look up the handler in our erased storage, and call its handle
    // function if it exists.
    let handler = self.handlers.get_dyn::<dyn Handler<S, E>>();
    handler.inspect(|handler| handler.handle(&mut self.state, event));
  }
  
  // Register an event handler for this system
  pub fn subscribe<E: Event + 'static>(&mut self, handler: impl Handler<S, E> + 'static) {
    self.handlers.put_dyn::<dyn Handler<S, E>>(handler);
  }
}
```

As a last step, we'll make our systems thread safe by wrapping them in an `Arc<Mutex<T>>`. This way we can give each event bus a 
copy of the systems that are listening to its events.

```rust
struct SyncSystem<S: System>(Arc<Mutex<InternalSystem<S>>>);
// impl<S> Clone for SyncSystem<S> ...
// impl<S> Deref for SyncSystem<S> ...
```

## Storing systems

Now that we have a way to store a system with all its handlers, it's time to think about how we will store the systems themselves.
Ideally, we would have a trait that is generic over them all, and store that as a trait object. Let's try to write such a trait:

```rust
trait Caller {
  fn call<E: Event + 'static>(&self, event: &E);
}

struct SystemList {
  systems: Vec<Box<dyn Caller>>,
}
```

Hit compile, and ...

> Error[E0038]: the trait `Caller` cannot be made into an object [...]
> because method `call` has generic type parameters.
{: .prompt-danger }

... bummer. The compiler won't make it that easy. One way to go around this would be to move the `E` generic parameter to the
trait definition. This does mean we'll need to store a list of these `dyn Caller` objects for each event type.

```rust
trait Caller<E: Event + 'static> {
  fn call(&self, event: &E);
}

impl<S: System + 'static, E: Event + 'static> Caller for SyncSystem<S> {
  fn call(&self, event: &E) {
    self.0.lock().expect("Poisoned mutex").handle(event);
  }
}
```

Now we can create an event bus parametrized on the event type

```rust
struct TypedEventBus<E: Event + 'static> {
  listeners: Vec<Box<dyn Caller<E>>>,
}

impl<E: Event + 'static> TypedEventBus<E> {
  pub fn subscribe<S: System + 'static>(
    &mut self, 
    system: SyncSystem<S>, 
    handler: impl Handler<S, E> + 'static) {
    system.lock().expect("Poisoned mutex").subscribe(handler);
    self.listeners.push(Box::new(system));
  }
  
  pub fn publish(&self, event: &E) {
    for caller in &self.listeners {
      caller.call(event);
    }
  }
}
```

Once again we will make a thread safe wrapper around this, this time using a `RwLock<T>`.

```rust
struct SyncEventBus<E: Event + 'static>(RwLock<TypedEventBus<E>>);
// impl<E> Deref for SyncEventBus<E> ...
```

## Putting it all together

Now we have all the components needed to create our final `EventBus`. To be able to store any `SyncEventBus<E>` and look it up
again later, we can use our new friend type erasure again.

```rust
#[derive(Clone)]
struct EventBus {
  buses: Arc<RwLock<ErasedStorage>>,
}
```

Let's create a helper function that given an event type `E` will give us the `SyncEventBus<E>` for that type, possibly inserting
it if it didn't exist yet.

```rust
impl EventBus {
  // Create a new event bus and call f with this bus.
  fn with_new_event_bus<E, F>(&self, f: F) 
  where
    E: Event + 'static,
    F: FnOnce(&SyncEventBus<E>) {
    let mut lock = self.buses.write().expect("Poisoned RwLock");
    lock.put(SyncEventBus::<E>::new());
    // Type inference saves us some typing
    let bus = lock.get().unwrap();
    f(bus);
  }
  
  // Grab the event bus of type E and call f with this bus.
  fn with_event_bus<E, F>(&self, f: F)
  where
    E: Event + 'static,
    F: FnOnce(&SyncEventBus<E>) {
    // Get a read lock to check if the bus exists
    let lock = self.buses.read().unwrap();
    let maybe_bus = lock.get(); 
    match maybe_bus {
      None => {
        // Release the lock and everything referencing it, 
        // because with_new_event_bus() needs a writer lock
        drop(maybe_bus);
        drop(lock);
        self.with_new_event_bus(f)
      }
      Some(bus) => f(bus),
    }
  }
}
```

With this in place, we can now implement `EventBus::publish`. This function will ask for an event bus of the correct type and simply
forward the event to it.

```rust
impl EventBus {
  pub fn publish<E: Event + 'static>(&self, event: &E) {
    self.with_event_bus(|bus| {
      let typed_bus = bus.read().unwrap();
      typed_bus.publish(event);
    })
  }
}
```

We're now just missing one component: Actually adding systems into the bus. Looking at our current code, it seems we need to call
`TypedEventBus::subscribe()` with a correct `SyncSystem` and a handler. We don't really want to expose too much of this detail to the user,
so let's start by wrapping over the `TypedEventBus::subscribe`.

```rust
impl EventBus {
  pub fn subscribe<S: System + 'static, E: Event + 'static>(
    &self,
    system: &SyncSystem<S>,
    handler: impl Handler<S, E> + 'static,
  ) {
    self.with_event_bus(|bus| {
      bus.write().expect("Poisoned RwLock").subscribe(system.clone(), handler);
    });
  }
}
```

Okay, not bad. But what about getting this `SyncSystem`. One way to handle this would be to make the user construct this
`SyncSystem` and then call all `subscribe` functions, but this feels a bit weird to me. Instead, let's look back at the `System` trait.
```rust 
trait System {
  fn initialize(event_bus: &EventBus, system: &SyncSystem<Self>) where Self: Sized;
}
```

Now we have a central place for users to register their handlers for a system. Additionally, we can now add a final method to our 
`EventBus`:

```rust
impl EventBus {
  fn add_system<S: System + 'static>(&self, system: S) {
    let system = SyncSystem::new(system);
    S::initialize(self, &system);
  }
}
```

And that's all we need! We now have a fully functional event bus that supports an arbitrary amount of event and system types.

## Conclusion

One drawback of this implementation is that we end up with a lot of allocations for references to the same system. However, since there
are usually not a ridiculously large amount of systems and these allocations only happen when registering the system, this is probably
fine. 

The full source code of this implementation can be found in the [scheduler](https://github.com/NotAPenguin0/andromeda-rs/tree/master/crates/scheduler)
crate of my engine, with a couple extensions:
- The `EventBus` also stores a type `T` with some arbitrary data (as long as it is `Clone + Send + Sync`).
- Each event handler also gets an `EventContext` with a reference to the event bus inside, to publish events from
  within events.
- The `Event` trait has a `Result` type that is returned from its handler. All these results are collected in a `Vec` that is
  returned from `publish`
