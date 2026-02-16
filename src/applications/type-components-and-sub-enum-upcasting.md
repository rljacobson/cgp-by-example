# Type Components and Sub-Enum Upcasting

This chapter combines two CGP ideas that unlock reusable transformation providers:

- choose output types through a type component (`#[cgp_type]` + `UseType`),
- construct with a smaller local enum, then upcast to the full enum.

If you remember one sentence, use this:

- **providers stay generic, contexts choose concrete types.**

## Vocabulary first

- **Type component**: a CGP component whose job is to provide an associated type.
- **Type provider**: the concrete type chosen in wiring (for example `UseType<LispExpr>`).
- **Sub-enum**: a small enum that contains only variants a provider needs to build.
- **Upcast**: safe promotion from that sub-enum into a larger target enum.

## Why this pattern exists

Suppose you write providers that convert math expressions to Lisp expressions.

Without type components, providers would hard-code `LispExpr` everywhere.
With type components, providers can say:

- "I need *some* `LispExpr` type from context."

Then each context chooses the concrete enum.

## Complete example

```rust
use cgp::core::field::impls::CanUpcast;
use cgp::extra::dispatch::MatchWithValueHandlersRef;
use cgp::extra::handler::{CanComputeRef, ComputerRef, ComputerRefComponent, UseInputDelegate};
use cgp::prelude::*;

pub struct ToLisp;

#[derive(Debug, PartialEq, Eq, Clone)]
pub struct Number(pub u64);

#[derive(Debug, PartialEq, Eq, Clone)]
pub struct Plus<Expr> {
    pub left: Box<Expr>,
    pub right: Box<Expr>,
}

#[derive(Debug, PartialEq, Eq, Clone, CgpData)]
pub enum MathExpr {
    Number(Number),
    Plus(Plus<MathExpr>),
}

#[derive(Debug, PartialEq, Eq, Clone)]
pub struct Ident(pub String);

#[derive(Debug, PartialEq, Eq, Clone)]
pub struct List<Expr>(pub Vec<Box<Expr>>);

#[derive(Debug, PartialEq, Eq, Clone, CgpData)]
pub enum LispExpr {
    List(List<LispExpr>),
    Number(Number),
    Ident(Ident),
}

// Local subset enum: only variants these providers need to construct
#[derive(Debug, PartialEq, Eq, Clone, CgpData)]
enum LispSubExpr<Expr> {
    List(List<Expr>),
    Number(Number),
    Ident(Ident),
}

#[cgp_type]
pub trait HasLispExprType {
    type LispExpr;
}

#[cgp_impl(new ToLispNumber)]
impl<Context, Code, LispExprT> ComputerRef<Code, Number> for Context
where
    Context: HasLispExprType<LispExpr = LispExprT>,
    LispSubExpr<LispExprT>: CanUpcast<LispExprT>,
{
    type Output = LispExprT;

    fn compute_ref(_context: &Context, _code: PhantomData<Code>, number: &Number) -> Self::Output {
        LispSubExpr::Number(number.clone()).upcast(PhantomData)
    }
}

#[cgp_impl(new ToLispPlus)]
impl<Context, Code, Expr, LispExprT> ComputerRef<Code, Plus<Expr>> for Context
where
    Context: HasLispExprType<LispExpr = LispExprT> + CanComputeRef<Code, Expr, Output = LispExprT>,
    LispSubExpr<LispExprT>: CanUpcast<LispExprT>,
{
    type Output = LispExprT;

    fn compute_ref(context: &Context, code: PhantomData<Code>, Plus { left, right }: &Plus<Expr>) -> Self::Output {
        let left_expr = context.compute_ref(code, left);
        let right_expr = context.compute_ref(code, right);
        let plus_ident = LispSubExpr::Ident(Ident("+".to_owned())).upcast(PhantomData);

        LispSubExpr::List(List(vec![
            plus_ident.into(),
            left_expr.into(),
            right_expr.into(),
        ]))
        .upcast(PhantomData)
    }
}

pub struct Interpreter;

delegate_components! {
    Interpreter {
        LispExprTypeProviderComponent: UseType<LispExpr>,
        ComputerRefComponent:
            UseInputDelegate<
                new ToLispComponents {
                    MathExpr: DispatchToLisp,
                    Number: ToLispNumber,
                    Plus<MathExpr>: ToLispPlus,
                }
            >,
    }
}

#[cgp_impl(new DispatchToLisp)]
impl<Code> ComputerRef<Code, MathExpr> for Interpreter {
    type Output = LispExpr;

    fn compute_ref(context: &Interpreter, code: PhantomData<Code>, expr: &MathExpr) -> Self::Output {
        <MatchWithValueHandlersRef as ComputerRef<Interpreter, Code, MathExpr>>::compute_ref(
            context, code, expr,
        )
    }
}

fn main() {
    let interpreter = Interpreter;
    let code = PhantomData::<ToLisp>;

    let expr = MathExpr::Plus(Plus {
        left: MathExpr::Number(Number(2)).into(),
        right: MathExpr::Number(Number(3)).into(),
    });

    assert_eq!(
        interpreter.compute_ref(code, &expr),
        LispExpr::List(List(vec![
            LispExpr::Ident(Ident("+".to_owned())).into(),
            LispExpr::Number(Number(2)).into(),
            LispExpr::Number(Number(3)).into(),
        ]))
    );
}
```

## What each term maps to in this code

- **Context**: `Interpreter`.
- **Type component capability**: `HasLispExprType`.
- **Type provider choice**: `UseType<LispExpr>`.
- **Sub-enum**: `LispSubExpr<Expr>`.
- **Upcast boundary**: `LispSubExpr<...>: CanUpcast<...>` and `.upcast(PhantomData)`.

## Where `LispExprTypeProviderComponent` comes from

`#[cgp_type]` on `HasLispExprType` generates CGP wiring artifacts, including the component key type used in delegation.

That is why this wiring key exists:

- `LispExprTypeProviderComponent`

So in practice:

- `HasLispExprType` is the trait you write,
- `LispExprTypeProviderComponent` is the generated key used in `delegate_components!`.

## Why `LispSubExpr<Expr>` is generic

`List` contains nested expression values, so the sub-enum needs a type parameter for "what expression type goes inside the list."

In providers, we instantiate it as `LispSubExpr<LispExprT>`:

- build local pieces as `LispSubExpr<LispExprT>`,
- upcast to `LispExprT`.

This keeps providers generic over the final output type.

## Step-by-step flow of one call

For `interpreter.compute_ref(PhantomData::<ToLisp>, &math_expr)`:

1. Input type is `MathExpr`, so input dispatch picks `DispatchToLisp`.
2. `DispatchToLisp` uses `MatchWithValueHandlersRef` to route by enum variant.
3. `Number` variant goes to `ToLispNumber`; `Plus` variant goes to `ToLispPlus`.
4. `ToLispPlus` recursively computes left/right subexpressions.
5. Provider constructs `LispSubExpr` values (`Ident`, `List`, etc.).
6. Each local value is upcast into the full context-chosen output type.

## Why this is better than hard-coding `LispExpr`

- providers can be reused in contexts that choose a different output enum,
- providers only construct variants they care about,
- upcast keeps construction safe and type-checked.

## Naming guide (what is arbitrary vs fixed)

- Arbitrary: `HasLispExprType`, `LispSubExpr`, `ToLispComponents`, `ToLispPlus`, `ToLispNumber`.
- Must stay consistent:
  - trait associated type name used in bounds (`LispExpr`),
  - wiring key generated from the type trait (`LispExprTypeProviderComponent`),
  - type/provider pairs in `UseInputDelegate`.

## Practical checklist

1. Define a type component with `#[cgp_type]`.
2. Choose concrete type in context with `UseType<...>`.
3. Build a local sub-enum for only needed variants.
4. Add `CanUpcast` bounds from sub-enum to final output type.
5. Upcast local pieces as you build outputs.

Practical rule: when providers only need part of a large output enum, build with a sub-enum and upcast into the context-chosen final type.
