# Provider-Side Dependencies

This chapter covers a practical cleanup pattern for provider `where` clauses.

Important framing:

- the cleanup technique is plain Rust (not CGP-specific),
- it is especially useful in CGP because provider bounds can grow quickly.

If you remember one sentence, use this:

- **factor out detailed trait bounds into a helper trait with a blanket impl, then depend on that helper in providers.**

## Step 1: direct provider bounds (works, but can get noisy)

Start with a straightforward provider that depends directly on getter traits.

```rust
use cgp::prelude::*;

#[cgp_auto_getter]
pub trait HasFirstName {
    fn first_name(&self) -> &str;
}

#[cgp_auto_getter]
pub trait HasLastName {
    fn last_name(&self) -> &str;
}

#[cgp_auto_getter]
pub trait HasTitle {
    fn title(&self) -> &str;
}

#[cgp_component(Introducer)]
pub trait CanIntroduce {
    fn introduce(&self) -> String;
}

#[cgp_impl(new FormalIntroduce)]
impl Introducer
where
    Self: HasFirstName + HasLastName + HasTitle,
{
    fn introduce(&self) -> String {
        format!("Hello, I am {} {} {}.", self.title(), self.first_name(), self.last_name())
    }
}

#[derive(HasField)]
pub struct Person {
    pub title: String,
    pub first_name: String,
    pub last_name: String,
}

delegate_components! {
    Person {
        IntroducerComponent: FormalIntroduce,
    }
}

fn main() {
    let person = Person {
        title: "Dr.".into(),
        first_name: "Ada".into(),
        last_name: "Lovelace".into(),
    };

    assert_eq!(person.introduce(), "Hello, I am Dr. Ada Lovelace.");
}
```

This is valid and often fine for small cases.

The issue appears when many providers repeat the same long bound list.

## Step 2: factor out dependencies with plain Rust

Now we use a standard Rust move:

1. create a helper trait,
2. add a blanket impl with detailed bounds,
3. depend on the helper trait in the provider.

```rust
use cgp::prelude::*;

#[cgp_auto_getter]
pub trait HasFirstName {
    fn first_name(&self) -> &str;
}

#[cgp_auto_getter]
pub trait HasLastName {
    fn last_name(&self) -> &str;
}

#[cgp_auto_getter]
pub trait HasTitle {
    fn title(&self) -> &str;
}

// Plain Rust helper trait
pub trait CanFormatDisplayName {
    fn display_name(&self) -> String;
}

// Plain Rust blanket impl with detailed dependencies
impl<Context> CanFormatDisplayName for Context
where
    Context: HasFirstName + HasLastName + HasTitle,
{
    fn display_name(&self) -> String {
        format!("{} {} {}", self.title(), self.first_name(), self.last_name())
    }
}

#[cgp_component(Introducer)]
pub trait CanIntroduce {
    fn introduce(&self) -> String;
}

#[cgp_impl(new FormalIntroduce)]
impl Introducer
where
    Self: CanFormatDisplayName,
{
    fn introduce(&self) -> String {
        format!("Hello, I am {}.", self.display_name())
    }
}

#[derive(HasField)]
pub struct Person {
    pub title: String,
    pub first_name: String,
    pub last_name: String,
}

delegate_components! {
    Person {
        IntroducerComponent: FormalIntroduce,
    }
}

fn main() {
    let person = Person {
        title: "Dr.".into(),
        first_name: "Ada".into(),
        last_name: "Lovelace".into(),
    };

    assert_eq!(person.introduce(), "Hello, I am Dr. Ada Lovelace.");
}
```

## What changed (same behavior, cleaner provider)

- Before: provider directly listed `HasFirstName + HasLastName + HasTitle`.
- After: provider lists `Self: CanFormatDisplayName`.
- The detailed bounds still exist, now localized in one blanket impl.

Equivalent ways to say this:

- we factor out trait dependencies,
- we hide low-level bounds behind a helper capability,
- we compose constraints once, then reuse them everywhere.

## Why `CanFormatDisplayName` is not a `#[cgp_component(...)]`

- `CanIntroduce` is consumer-facing and needs component wiring (`IntroducerComponent`).
- `CanFormatDisplayName` is an internal helper dependency trait.

So we intentionally use:

- `#[cgp_component(...)]` for capabilities that need provider selection in `delegate_components!`,
- plain traits + blanket impls for helper dependency factoring.

If you later want swappable formatting strategies, you can promote formatting to its own component.

## Plain helper trait vs component trait

Use this quick contrast when deciding:

| Plain Rust helper trait (`CanFormatDisplayName`) | CGP component + consumer trait |
|---|---|
| Reuse internal helper logic across providers | Expose a capability as part of context API |
| No independent provider wiring needed | Needs provider selection via `delegate_components!` |
| Behavior is derived from existing capabilities/bounds | Behavior should be swappable/configurable per context |
| Acts as implementation composition | Acts as a capability slot |
| Answers: "what helper logic do I need?" | Answers: "which provider should this context use?" |
| Usually plain trait + blanket impl | Usually `#[cgp_component(...)]` + provider impl(s) |

Memory shortcut:

- plain trait = compose helpers,
- component trait = choose providers.

## Where provider-side dependencies live

Look in two places:

- helper blanket impl: detailed trait requirements,
- provider impl: compact high-level dependency.

This is the provider-side dependency pattern in one line:

- **detailed bounds in helper impl, short bounds in provider impl.**

## Practical checklist

1. Start with a working provider, even if bounds are long.
2. Spot repeated or noisy bound groups.
3. Extract a helper trait for that group.
4. Add a blanket impl with detailed bounds.
5. Replace long provider bounds with the helper trait.

Practical rule: when provider bounds repeat or become hard to scan, factor them out with a plain Rust helper trait + blanket impl.
