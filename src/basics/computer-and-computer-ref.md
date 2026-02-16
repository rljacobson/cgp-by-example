# Compute Components: Computer and ComputerRef

This chapter explains how CGP models computation as a component.

Goal:

- use `Computer` for by-value compute,
- use `ComputerRef` for by-reference compute,
- wire both on one context.

## One sentence mental model

- `Computer` means: consume input and compute output.
- `ComputerRef` means: borrow input and compute output.

Same idea, different ownership style.

## Vocabulary first: `Eval` and `Code`

You will see this shape a lot:

- `compute(code, input)`
- `compute_ref(code, input_ref)`

The `code` argument is a type-level selector (a marker type), usually a zero-sized struct.

Example marker:

- `pub struct Eval;`

What that means:

- `Eval` is not runtime data.
- It is a type tag used to name "which computation mode" you are invoking.
- In simple examples, we pass it but do not branch on it yet.

So `Code` in `Computer<Code, Input>` is the generic type parameter for that tag.

## Example 1: one computation mode, generic provider

Start with one mode (`Eval`) and one generic provider impl.

```rust
use cgp::extra::handler::{CanCompute, Computer};
use cgp::prelude::*;

pub struct Eval;

#[derive(Clone, Copy)]
pub struct Pair {
    pub left: u64,
    pub right: u64,
}

#[cgp_impl(new SumPair)]
impl<Context, Code> Computer<Code, Pair> for Context {
    type Output = u64;

    fn compute(_context: &Context, _code: PhantomData<Code>, input: Pair) -> Self::Output {
        input.left + input.right
    }
}

pub struct App;

delegate_components! {
    App {
        ComputerComponent: SumPair,
    }
}

fn main() {
    let app = App;
    let eval = PhantomData::<Eval>;

    let out = app.compute(eval, Pair { left: 2, right: 3 });
    assert_eq!(out, 5);
}
```

What this shows:

- `SumPair` is generic over `Code`, so one impl can serve many mode tags.
- We only use one mode (`Eval`) at call-site.
- This is a good default starting point.

## Example 2: two computation modes, separate impl behavior

Now we keep the same input type, but give different behavior per mode.

```rust
use cgp::extra::handler::{CanCompute, Computer};
use cgp::prelude::*;

pub struct Eval;
pub struct Explain;

#[derive(Clone, Copy)]
pub struct Pair {
    pub left: u64,
    pub right: u64,
}

#[cgp_impl(new SumValue)]
impl<Context> Computer<Eval, Pair> for Context {
    type Output = u64;

    fn compute(_context: &Context, _code: PhantomData<Eval>, input: Pair) -> Self::Output {
        input.left + input.right
    }
}

#[cgp_impl(new SumExplain)]
impl<Context> Computer<Explain, Pair> for Context {
    type Output = String;

    fn compute(_context: &Context, _code: PhantomData<Explain>, input: Pair) -> Self::Output {
        format!("{} + {}", input.left, input.right)
    }
}

pub struct App;

delegate_components! {
    App {
        ComputerComponent: UseDelegate<
            new ModeHandlers {
                Eval: SumValue,
                Explain: SumExplain,
            }
        >,
    }
}

fn main() {
    let app = App;

    let value = app.compute(PhantomData::<Eval>, Pair { left: 2, right: 3 });
    let text = app.compute(PhantomData::<Explain>, Pair { left: 2, right: 3 });

    assert_eq!(value, 5);
    assert_eq!(text, "2 + 3");
}
```

What changed from Example 1:

- provider impls are now mode-specific (`Eval` vs `Explain`),
- wiring uses `UseDelegate` to dispatch by `Code`,
- output type can differ by mode (`u64` vs `String`).

This is the key role of `Code`: same input type, different compute behavior.

## `UseDelegate` and `new ModeHandlers` explained

In Example 2, this wiring is the important line:

```rust
ComputerComponent: UseDelegate<
    new ModeHandlers {
        Eval: SumValue,
        Explain: SumExplain,
    }
>,
```

Read it as a compile-time lookup table:

- key = `Code` type (`Eval`, `Explain`),
- value = provider (`SumValue`, `SumExplain`).

`ModeHandlers` is just a name for that table type.

- It is **not** a reserved CGP keyword.
- You can rename it (for example `ComputeByMode`, `PairHandlers`, `CodeTable`).

What must match:

- `Eval` and `Explain` in this table must match the code tags you call with at runtime (`PhantomData::<Eval>`, `PhantomData::<Explain>`).
- `SumValue` and `SumExplain` must implement the corresponding `Computer<Code, Input>` cases.

So the full phrase `UseDelegate<new ModeHandlers { ... }>` means:

- "for this component, choose provider by code tag using this table."

## Where `ComputerRef` fits

Everything above used `Computer` (by-value).

`ComputerRef` is the borrowed-input counterpart:

- `Computer<Code, Input>` takes `input: Input`
- `ComputerRef<Code, Input>` takes `input: &Input`

Use `ComputerRef` when you want to avoid moving/cloning large values, or when recursive data should stay borrowed.

## Naming map for this chapter

- `Eval`, `Explain` = mode tags (`Code` values at type level).
- `Pair` = input type.
- `ComputerComponent` = compute capability slot on the context.
- `SumPair` / `SumValue` / `SumExplain` = providers.
- `App` = context choosing providers.

## Practical rules

- Start with one mode tag (`Eval`) and a generic `Computer<Code, Input>` impl.
- Split into mode-specific impls only when behavior diverges.
- Use `UseDelegate<...>` when dispatch should happen by `Code`.
- Prefer `ComputerRef` for borrow-first recursive workflows.
