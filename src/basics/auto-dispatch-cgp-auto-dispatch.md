# Auto Dispatch with `#[cgp_auto_dispatch]`

This chapter shows how to remove enum dispatch boilerplate with `#[cgp_auto_dispatch]`.

Goal:

- define a normal trait once,
- implement it for each variant payload type,
- get automatic enum-level dispatch for free.

If you remember one sentence, use this:

- **`#[cgp_auto_dispatch]` generates the "match enum variant, then call trait on payload" glue for you.**

## When this is useful

You often have:

- an enum (for example `Shape`),
- per-variant structs (`Circle`, `Rectangle`),
- the same trait implemented on each payload type.

Without the macro, you would write manual `impl Trait for Enum` with a `match`.
With the macro, CGP generates that dispatch impl.

## Complete example

```rust
use core::f64::consts::PI;

use cgp::prelude::*;

#[derive(CgpData)]
pub enum Shape {
    Circle(Circle),
    Rectangle(Rectangle),
}

pub struct Circle {
    pub radius: f64,
}

pub struct Rectangle {
    pub width: f64,
    pub height: f64,
}

#[cgp_auto_dispatch]
pub trait HasArea {
    fn area(&self) -> f64;
}

impl HasArea for Circle {
    fn area(&self) -> f64 {
        PI * self.radius * self.radius
    }
}

impl HasArea for Rectangle {
    fn area(&self) -> f64 {
        self.width * self.height
    }
}

#[cgp_auto_dispatch]
pub trait CanScale {
    fn scale(&mut self, factor: f64);
}

impl CanScale for Circle {
    fn scale(&mut self, factor: f64) {
        self.radius *= factor;
    }
}

impl CanScale for Rectangle {
    fn scale(&mut self, factor: f64) {
        self.width *= factor;
        self.height *= factor;
    }
}

fn main() {
    let mut shape = Shape::Rectangle(Rectangle {
        width: 2.0,
        height: 2.0,
    });

    assert_eq!(shape.area(), 4.0);

    shape.scale(2.0);
    assert_eq!(shape.area(), 16.0);
}
```

## What each part does

- `#[derive(CgpData)]` marks enum data metadata needed for auto dispatch.
- `#[cgp_auto_dispatch]` on `HasArea` generates enum dispatch for `&self` style calls.
- `#[cgp_auto_dispatch]` on `CanScale` generates enum dispatch for `&mut self` style calls.
- Your manual impls stay focused on payload types (`Circle`, `Rectangle`).

So the enum behaves like it implements the trait, without hand-written `match` boilerplate.

## What this is (and is not)

- This is a dispatch convenience macro.
- It is not a component wiring macro.

In other words:

- `#[cgp_auto_dispatch]` helps with enum method forwarding.
- `#[cgp_component]` + `#[cgp_impl]` + `delegate_components!` helps with provider selection by context.

Both are useful. They solve different problems.

## Practical checklist

1. Derive `CgpData` on the enum.
2. Add `#[cgp_auto_dispatch]` on the trait you want forwarded.
3. Implement that trait on each payload type.
4. Call methods on enum values directly.

Practical rule: use `#[cgp_auto_dispatch]` when your enum dispatch is straightforward and you want to avoid manual forwarding code.
