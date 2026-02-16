# Enum Upcasting and Downcasting

This chapter shows the safe enum casting pattern in CGP.

Goal:

- move from a smaller enum to a bigger enum (**upcast**),
- move from a bigger enum to a smaller enum (**downcast**),
- safely handle what does not fit.

## What to derive

For these casts, derive `CgpData` on your enums.

`CgpData` gives the enum traits needed by the cast helpers (`upcast`, `downcast`, `downcast_fields`).

## Complete example

```rust
use core::marker::PhantomData;

use cgp::core::field::impls::{CanDowncast, CanDowncastFields, CanUpcast};
use cgp::prelude::*;

#[derive(Debug, PartialEq, CgpData)]
pub enum Shape {
    Circle(Circle),
    Rectangle(Rectangle),
}

#[derive(Debug, PartialEq, CgpData)]
pub enum ShapePlus {
    Triangle(Triangle),
    Rectangle(Rectangle),
    Circle(Circle),
}

#[derive(Debug, PartialEq, CgpData)]
pub enum TriangleOnly {
    Triangle(Triangle),
}

#[derive(Debug, PartialEq, CgpData)]
pub enum CircleOnly {
    Circle(Circle),
}

#[derive(Debug, PartialEq)]
pub struct Circle {
    pub radius: f64,
}

#[derive(Debug, PartialEq)]
pub struct Rectangle {
    pub width: f64,
    pub height: f64,
}

#[derive(Debug, PartialEq)]
pub struct Triangle {
    pub base: f64,
    pub height: f64,
}

fn main() {
    // Upcast: smaller enum -> bigger enum
    let shape = Shape::Circle(Circle { radius: 5.0 });
    let shape_plus = shape.upcast(PhantomData::<ShapePlus>);
    assert_eq!(shape_plus, ShapePlus::Circle(Circle { radius: 5.0 }));

    // Downcast success: bigger enum -> smaller enum
    let ok = ShapePlus::Rectangle(Rectangle {
        width: 3.0,
        height: 4.0,
    })
    .downcast(PhantomData::<Shape>)
    .ok();
    assert_eq!(ok, Some(Shape::Rectangle(Rectangle { width: 3.0, height: 4.0 })));

    // Downcast failure: variant not in Shape
    let remainder = ShapePlus::Triangle(Triangle {
        base: 3.0,
        height: 4.0,
    })
    .downcast(PhantomData::<Shape>)
    .unwrap_err();

    // Handle the remainder by downcasting fields into another enum
    let triangle_only_result: Result<TriangleOnly, _> =
        remainder.downcast_fields(PhantomData::<TriangleOnly>);

    let triangle_only = triangle_only_result.ok();
    assert_eq!(
        triangle_only,
        Some(TriangleOnly::Triangle(Triangle {
            base: 3.0,
            height: 4.0,
        }))
    );

    // downcast_fields can also fail, depending on what the remainder still contains.
    let remainder2 = ShapePlus::Rectangle(Rectangle {
        width: 8.0,
        height: 5.0,
    })
    .downcast(PhantomData::<CircleOnly>)
    .unwrap_err();

    let not_triangle: Result<TriangleOnly, _> =
        remainder2.downcast_fields(PhantomData::<TriangleOnly>);

    assert!(not_triangle.is_err());
}
```

## What each cast means

- `upcast(PhantomData::<Target>)` says: "convert this enum into a compatible larger target enum."
- `downcast(PhantomData::<Target>)` says: "try converting into this smaller target enum."
- `downcast` returns `Result<Target, Remainder>`:
  - `Ok(target)` when the variant exists in target,
  - `Err(remainder)` when it does not.
- `downcast_fields(PhantomData::<Target>)` tries another safe conversion step on an extracted remainder source.

## What is `Remainder`?

`Remainder` is **not usually the original source enum type**.

It is an internal "what is left" extraction type produced by CGP when a downcast fails.
You can think of it as a strongly typed carrier for the still-unhandled variants.

That is why this works:

- first `downcast` gives `Err(remainder)`,
- then `downcast_fields` continues from that remainder without losing type safety.

Practical takeaway: treat `Remainder` as "unhandled cases so far," not as "the original enum again."

## How `downcast_fields` works

`downcast_fields` serves the same purpose as `downcast`, but for the remainder type.

- Use `downcast` on a full enum value.
- If that returns `Err(remainder)`, use `downcast_fields` on that remainder.

Its return shape is also the same pattern: `Result<Target, NextRemainder>`.

- `Ok(target)` means the remainder fits the new target enum.
- `Err(next_remainder)` means it still does not fit, and you can continue routing.

## Why this is useful

- **Layered APIs**: convert from domain-specific enums into broader app enums with `upcast`.
- **Selective handling**: try `downcast` into the subset you care about, then pass remainder onward.
- **Exhaustive workflows**: repeatedly downcast remainder into other subsets until every case is handled.

## Mental model

- Upcast = "embed into a bigger enum."
- Downcast = "attempt to recover a smaller enum."
- Remainder = "the part that did not fit yet."

This gives you safe, composable enum routing without `unsafe` and without ad-hoc conversion glue.
