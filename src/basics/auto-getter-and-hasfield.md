# Auto Getter and HasField

This chapter explains two pieces that often appear together:

- `#[derive(HasField)]`
- `#[cgp_auto_getter]`

Short version:

- `HasField` gives a context a generic way to read fields by field name.
- `cgp_auto_getter` uses `HasField` to auto-implement simple getter traits.

If `HasField` is the raw field access engine, `cgp_auto_getter` is the user-friendly wrapper.

## Why this exists

You usually want provider code to depend on traits like `HasName`, not on concrete struct fields.

That keeps providers reusable:

- any context with a matching getter trait can use the provider,
- the provider does not care about the concrete context type.

`HasField` + `cgp_auto_getter` is the easiest way to create those getter traits.

## Step 1: `#[derive(HasField)]` on a context

```rust
use cgp::prelude::*;

#[derive(HasField)]
pub struct Person {
    pub name: String,
    pub age: u32,
}
```

After this derive, `Person` can expose its fields through CGP's generic field-access trait machinery.

In plain terms: the derive teaches CGP that `Person` has a `name` field and an `age` field, and what their types are.

## Step 2: define getter traits with `#[cgp_auto_getter]`

```rust
use cgp::prelude::*;

#[cgp_auto_getter]
pub trait HasName {
    fn name(&self) -> &str;
}

#[cgp_auto_getter]
pub trait HasAge {
    fn age(&self) -> &u32;
}
```

`#[cgp_auto_getter]` auto-generates blanket getter implementations using `HasField`.

In plain terms:

- if a context has a `name` field compatible with the `name()` return type, it gets `HasName` automatically.
- if a context has an `age` field compatible with the `age()` return type, it gets `HasAge` automatically.

## Full working example

```rust
use cgp::prelude::*;

#[cgp_auto_getter]
pub trait HasName {
    fn name(&self) -> &str;
}

#[cgp_auto_getter]
pub trait HasAge {
    fn age(&self) -> &u32;
}

#[derive(HasField)]
pub struct Person {
    pub name: String,
    pub age: u32,
}

fn print_profile<C: HasName>(person: &C) {
    println!("name={}", person.name());
}

fn print_age<C: HasAge>(person: &C) {
    println!("age={}", person.age());
}

fn main() {
    let person = Person {
        name: "Ada".into(),
        age: 36,
    };

    print_profile(&person);
    print_age(&person);
}
```

No manual impl blocks were needed for `HasName` or `HasAge`.

## What exactly gets matched

For `#[cgp_auto_getter]`, the getter method name is important:

- `fn name(&self) -> ...` expects a `name` field.
- `fn age(&self) -> ...` expects an `age` field.

And the return type must be compatible with the field type.

Example:

- `name: String` can satisfy `fn name(&self) -> &str`.
- `age: u32` satisfies `fn age(&self) -> &u32`.

## How this helps providers

Providers can depend on getter traits, not concrete structs:

```rust
use cgp::prelude::*;

#[cgp_component(Greeter)]
pub trait CanGreet {
    fn greet(&self);
}

#[cgp_auto_getter]
pub trait HasName {
    fn name(&self) -> &str;
}

#[cgp_impl(new GreetHello)]
impl Greeter
where
    Self: HasName,
{
    fn greet(&self) {
        println!("Hello, {}!", self.name());
    }
}
```

This provider works with any context that has `HasName`, whether that context is `Person`, `AdminUser`, or something else.

## Setting a field with `HasFieldMut`

`#[derive(HasField)]` also enables mutable field access via `HasFieldMut`.

That lets you update a field in generic code without naming the concrete context type.

```rust
use core::marker::PhantomData;
use cgp::prelude::*;

#[derive(HasField)]
pub struct Person {
    pub name: String,
    pub age: u32,
}

fn rename<C>(context: &mut C, new_name: String)
where
    C: HasFieldMut<Symbol!("name"), Value = String>,
{
    *context.get_field_mut(PhantomData) = new_name;
}

fn main() {
    let mut person = Person {
        name: "Ada".into(),
        age: 36,
    };

    rename(&mut person, "Grace".into());
    assert_eq!(person.name, "Grace");
}
```

How to read this:

- `HasFieldMut<Symbol!("name")>` means "this context has a mutable `name` field".
- `get_field_mut(PhantomData)` returns `&mut String` for that field.
- We assign through that mutable reference.

So a practical rule is:

- use `#[cgp_auto_getter]` for read-focused traits,
- use `HasFieldMut` when you need generic field updates.

## Mental model

- `HasField` = low-level field lookup capability generated from your struct fields.
- `cgp_auto_getter` = easy getter traits built on top of `HasField`.
- Provider constraints (like `Self: HasName`) = reusable requirements that many contexts can satisfy.

This is one of the main CGP patterns: use small getter traits as reusable dependencies.
