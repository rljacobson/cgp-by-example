# Type Components and Context Bounds

This chapter introduces `#[cgp_type]` in the simplest useful way.

Goal:

- define a type capability with `#[cgp_type]`,
- choose the concrete type in context wiring,
- use trait bounds on `Context` inside a provider.

If you remember one sentence, use this:

- **a type component lets the context choose a type the provider depends on.**

## Vocabulary first

- **Type component**: a CGP capability that provides an associated type.
- **Context bound**: a `where` constraint on `Context`/`Self` in provider impls.
- **Type provider**: the concrete type selection in `delegate_components!` (usually `UseType<...>`).

## Complete example

```rust
use cgp::prelude::*;

#[cgp_type]
pub trait HasGreetingType {
    type Greeting;
}

#[cgp_component(Greeter)]
pub trait CanGreet: HasGreetingType {
    fn greet(&self, name: &str) -> Self::Greeting;
}

#[cgp_impl(new BuildGreeting)]
impl Greeter
where
    Self: HasGreetingType,
    String: Into<Self::Greeting>,
{
    fn greet(&self, name: &str) -> Self::Greeting {
        format!("Hello, {}!", name).into()
    }
}

pub struct AppText;

delegate_components! {
    AppText {
        GreetingTypeProviderComponent: UseType<String>,
        GreeterComponent: BuildGreeting,
    }
}

pub struct AppBoxed;

delegate_components! {
    AppBoxed {
        GreetingTypeProviderComponent: UseType<Box<str>>,
        GreeterComponent: BuildGreeting,
    }
}

fn main() {
    let text = AppText.greet("Ada");
    let boxed = AppBoxed.greet("Grace");

    assert_eq!(text, "Hello, Ada!".to_owned());
    assert_eq!(boxed, Box::<str>::from("Hello, Grace!"));
}
```

## What each part means

- `#[cgp_type]` on `HasGreetingType` defines a type capability.
- `CanGreet` uses that capability by returning `Self::Greeting`.
- `BuildGreeting` is one provider implementation of `Greeter`.
- The `where` clause in `BuildGreeting` is the key context bound:
  - `Self: HasGreetingType` means this provider needs a context that supplies `Greeting`.
  - `String: Into<Self::Greeting>` means the provider can build a `String` and convert it to the context-chosen greeting type.

## Where `GreetingTypeProviderComponent` comes from

`#[cgp_type]` generates the wiring key used by `delegate_components!`.

So these names correspond to two different things:

- `HasGreetingType` = trait you wrote.
- `GreetingTypeProviderComponent` = generated component key used in wiring (generated from the associated type name `Greeting` as `Greeting` + `TypeProviderComponent`).

## Why the same provider works in two contexts

`BuildGreeting` is generic over `Self`.

- `AppText` chooses `Greeting = String`.
- `AppBoxed` chooses `Greeting = Box<str>`.

Since both satisfy `String: Into<Self::Greeting>`, both can reuse the same provider.

This is the practical value of context bounds:

- provider says what it needs,
- context wiring chooses values that satisfy those needs.

## Naming guide

- Arbitrary names: `HasGreetingType`, `BuildGreeting`, `AppText`, `AppBoxed`.
- Must match references:
  - `Self::Greeting` in `CanGreet` and provider bounds,
  - `GreetingTypeProviderComponent` in `delegate_components!`,
  - `GreeterComponent: BuildGreeting` wiring.

## Practical checklist

1. Define a type trait with `#[cgp_type]`.
2. Use that associated type in a consumer trait.
3. Add provider bounds describing required capabilities/conversions.
4. Choose concrete type with `UseType<...>` per context.
5. Wire the main behavior component to your provider.

Practical rule: use `#[cgp_type]` when provider logic should be reusable but output/input type choice should stay in context wiring.
