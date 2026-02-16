# Defining Components with `#[cgp_component]`

This chapter explains how to define CGP components in v0.6.1 style.

Goal:

- understand the two `#[cgp_component]` forms,
- know what names are generated,
- learn practical naming and usage best practices.

If you remember one sentence, use this:

- **`#[cgp_component]` turns a consumer trait into a wireable CGP capability.**

## The two macro forms

`#[cgp_component]` has two forms.

### Form A (recommended default)

```rust
#[cgp_component(Greeter)]
pub trait CanGreet {
    fn greet(&self);
}
```

This is the form you should use most of the time.

### Form B (explicit key/value form)

```rust
#[cgp_component {
    name: GreeterComponent,
    provider: Greeter,
    context: Context,
}]
pub trait CanGreet {
    fn greet(&self);
}
```

For this example, Form B is equivalent to Form A.

## What gets generated

For `#[cgp_component(Greeter)]`:

- consumer trait: `CanGreet` (the trait you wrote),
- provider trait: `Greeter` (from macro argument),
- component key type: `GreeterComponent` (generated default name).

Default key naming rule:

- `` `ProviderName` + `Component` ``

So `Greeter` becomes `GreeterComponent`.

## Complete example

```rust
use cgp::prelude::*;

#[cgp_component(Greeter)]
pub trait CanGreet {
    fn greet(&self) -> String;
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
    fn greet(&self) -> String {
        format!("Hello, {}!", self.name())
    }
}

#[derive(HasField)]
pub struct Person {
    pub name: String,
}

delegate_components! {
    Person {
        GreeterComponent: GreetHello,
    }
}

fn main() {
    let person = Person { name: "Ada".into() };
    assert_eq!(person.greet(), "Hello, Ada!");
}
```

How to read this:

1. `CanGreet` defines what callers can ask for.
2. `Greeter` is the provider trait generated for implementations.
3. `GreeterComponent` is the key used in context wiring.
4. `GreetHello` is the concrete provider selected for `Person`.

## When to use the explicit form

Use Form B only when you need customization.

Typical reasons:

- custom component key name (`name: ...`),
- custom provider trait name (`provider: ...`),
- custom context generic identifier (`context: ...`),
- advanced dispatch derivation (`derive_delegate: [...]`).

For basic tutorials and most app code, Form A is clearer.

## Best practices (v0.6.1)

- Prefer Form A: `#[cgp_component(ProviderName)]`.
- Use a consumer trait name like `CanX` (`CanGreet`, `CanCompute`, etc.).
- Use a provider trait name that sounds like behavior role (`Greeter`, `Fetcher`, `Formatter`).
- Let generated `...Component` names stand unless you have naming conflicts.
- Keep consumer traits small and focused (one capability area).
- Use direct `delegate_components!` on context types (v0.6.1 style).

## Quick naming map template

For:

```rust
#[cgp_component(FooProvider)]
pub trait CanDoFoo {
    fn do_foo(&self);
}
```

Expect:

- consumer trait: `CanDoFoo`
- provider trait: `FooProvider`
- component key: `FooProviderComponent`

Use this map whenever names feel confusing.
