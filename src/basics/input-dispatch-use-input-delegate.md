# Input-Based Dispatch with UseInputDelegate

This chapter shows how one compute component can route to different providers based on input type.

Goal:

- define small handlers per expression node type,
- wire them with `UseInputDelegate`,
- dispatch enum variants with `MatchWithValueHandlersRef`.

If you remember one sentence, use this:

- **`UseInputDelegate` chooses a provider by input type.**

## Vocabulary first

- **Input type**: the type being computed (for example `Number`, `Plus<MathExpr>`, or `MathExpr`).
- **Input dispatch table**: the `new ... { Type: Provider }` block inside `UseInputDelegate`.
- **Dispatcher provider**: a provider that routes enum variants (here: `DispatchEval`).

You can say the same idea in different ways:

- input type in -> matching provider out,
- one component, many input-specific handlers,
- the context picks provider by input shape.

## Complete example

```rust
use cgp::extra::dispatch::MatchWithValueHandlersRef;
use cgp::extra::handler::{CanComputeRef, ComputerRef, ComputerRefComponent, UseInputDelegate};
use cgp::prelude::*;

pub struct Eval;

#[derive(Debug, PartialEq)]
pub struct Number(pub u64);

#[derive(Debug, PartialEq)]
pub struct Plus<Expr> {
    pub left: Box<Expr>,
    pub right: Box<Expr>,
}

#[derive(Debug, PartialEq, CgpData)]
pub enum MathExpr {
    Number(Number),
    Plus(Plus<MathExpr>),
}

#[cgp_impl(new EvalNumber)]
impl<Context, Code> ComputerRef<Code, Number> for Context {
    type Output = u64;

    fn compute_ref(_context: &Context, _code: PhantomData<Code>, Number(value): &Number) -> Self::Output {
        *value
    }
}

#[cgp_impl(new EvalPlus)]
impl<Context, Code, Expr> ComputerRef<Code, Plus<Expr>> for Context
where
    Context: CanComputeRef<Code, Expr, Output = u64>,
{
    type Output = u64;

    fn compute_ref(context: &Context, code: PhantomData<Code>, Plus { left, right }: &Plus<Expr>) -> Self::Output {
        context.compute_ref(code, left) + context.compute_ref(code, right)
    }
}

pub struct Interpreter;

delegate_components! {
    Interpreter {
        ComputerRefComponent:
            UseInputDelegate<
                new EvalComponents {
                    MathExpr: DispatchEval,
                    Number: EvalNumber,
                    Plus<MathExpr>: EvalPlus,
                }
            >,
    }
}

#[cgp_impl(new DispatchEval)]
impl<Code> ComputerRef<Code, MathExpr> for Interpreter {
    type Output = u64;

    fn compute_ref(context: &Interpreter, code: PhantomData<Code>, expr: &MathExpr) -> Self::Output {
        <MatchWithValueHandlersRef as ComputerRef<Interpreter, Code, MathExpr>>::compute_ref(
            context, code, expr,
        )
    }
}

fn main() {
    let interpreter = Interpreter;
    let code = PhantomData::<Eval>;

    let expr = MathExpr::Plus(Plus {
        left: MathExpr::Number(Number(2)).into(),
        right: MathExpr::Number(Number(3)).into(),
    });

    assert_eq!(interpreter.compute_ref(code, &expr), 5);
}
```

## What each part means

- `ComputerRef<Code, Input>` is the compute capability for borrowed input.
- `EvalNumber` handles `Number` input.
- `EvalPlus` handles `Plus<MathExpr>` input and recursively computes children.
- `DispatchEval` handles top-level `MathExpr` by delegating variant routing.
- `UseInputDelegate<new EvalComponents { ... }>` is the input dispatch table on `Interpreter`.

## Why `DispatchEval` is on `Interpreter`, but others use generic `Context`

You might notice:

- `EvalNumber` and `EvalPlus` are implemented for generic `Context`.
- `DispatchEval` is implemented for concrete `Interpreter`.

That split is intentional.

- `EvalNumber` and `EvalPlus` are reusable node handlers. They only need capability bounds like `Context: CanComputeRef<...>`, so they can work in many contexts.
- `DispatchEval` is the top-level entry-point router for this specific interpreter wiring. It relies on the context's configured input-dispatch table for `MathExpr`, so keeping it on `Interpreter` makes the assembly point explicit.

Practical pattern:

- make leaf/sub-expression providers generic and reusable,
- make the top-level enum dispatcher context-specific.

## How the routing actually works

For `interpreter.compute_ref(code, &expr)` where `expr: MathExpr`:

1. Input is `MathExpr`, so `UseInputDelegate` picks `DispatchEval`.
2. `DispatchEval` calls `MatchWithValueHandlersRef`, which matches the enum variant.
3. If variant is `Number`, it routes to `EvalNumber`.
4. If variant is `Plus`, it routes to `EvalPlus`.
5. `EvalPlus` recursively calls `compute_ref` on sub-expressions.

So there are two levels:

- level 1: input-type dispatch (`UseInputDelegate`),
- level 2: enum-variant dispatch (`MatchWithValueHandlersRef`).

## When to use `MatchWithValueHandlersRef`

Use `MatchWithValueHandlersRef` at the enum-dispatch boundary.

- It belongs in the provider that handles the enum input itself (here: `DispatchEval` for `MathExpr`).
- It does not belong in variant-specific providers (`EvalNumber`, `EvalPlus`), because those already handle concrete input types.

Rule of thumb:

- top-level enum provider -> `MatchWithValueHandlersRef`,
- variant handlers -> regular provider logic.

And by ownership style:

- use `MatchWithValueHandlers` with by-value `Computer`,
- use `MatchWithValueHandlersRef` with by-reference `ComputerRef`.

## `new EvalComponents` explained

In this wiring:

```rust
UseInputDelegate<
    new EvalComponents {
        MathExpr: DispatchEval,
        Number: EvalNumber,
        Plus<MathExpr>: EvalPlus,
    }
>
```

- `EvalComponents` is an arbitrary name for the generated input-dispatch table type.
- It is not a reserved CGP keyword.
- You can rename it (for example `ExprHandlers`, `InputTable`, `EvalByInput`).

What must match:

- each key type (`MathExpr`, `Number`, `Plus<MathExpr>`) must match the input type of the provider on the right.

## Why the `DispatchEval` impl uses explicit trait syntax

This line uses fully-qualified syntax on purpose:

```rust
<MatchWithValueHandlersRef as ComputerRef<Interpreter, Code, MathExpr>>::compute_ref(...)
```

Reason: both `ComputerRef` and `CanComputeRef` define `compute_ref`, so explicit syntax avoids method-name ambiguity in current `cgp` usage.

## What is doing the routing?

- `UseInputDelegate` selects a provider by input type.
- `DispatchEval` handles the top-level enum `MathExpr`.
- `MatchWithValueHandlersRef` picks the matching variant handler (`EvalNumber` or `EvalPlus`).

You can read it as: **input type in -> matching provider out**.

## Why this pattern is useful

- each handler stays tiny and focused,
- adding a new variant means adding one new handler + one wiring line,
- no giant manual `match` function is needed in one place.

Practical rule: when behavior depends on input shape, start with `UseInputDelegate` and keep handlers small.
