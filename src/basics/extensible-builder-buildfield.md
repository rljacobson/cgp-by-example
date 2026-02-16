# Extensible Builder with BuildField

This chapter shows a simple CGP builder pattern using `BuildField`.

Goal:

- build a bigger struct from smaller structs,
- keep each piece independent,
- finish with one strongly typed final value.

## What `BuildField` gives you

For a target struct, deriving `BuildField` gives you a typed builder flow:

- `Target::builder()` starts an empty partial builder,
- `.build_field(...)` sets one field,
- `.build_from(...)` merges fields from another struct,
- `.finalize_build()` produces the final target when all required fields are present.

Think of it as: **start empty -> add pieces -> finalize**.

## `HasField` vs `HasFields` (important)

The names are very similar, and that is easy to mix up:

- `HasField` (singular) is about one field at a time by tag (for example `name` or `employee_id`).
- `HasFields` (plural) is about the whole set of fields of a type.

In this chapter, `build_from(...)` takes a whole source struct and merges its field set into the target builder.
So source types used with `build_from(...)` need `HasFields` (plural), even if they only have one field like `EmployeeId`.

By contrast, `build_field(...)` does not take a source struct. It takes a field tag (`Symbol!("...")`) and a field value, then sets that one target field.

(In many CGP codebases you also derive `HasField` for individual field access, but this specific `build_from` + `build_field` example does not require it on `Person` or `EmployeeId`.)

Quick memory trick:

- **Field** = one slot.
- **Fields** = a bundle of slots.

## Complete example

```rust
use core::marker::PhantomData;

use cgp::core::field::impls::CanBuildFrom;
use cgp::prelude::*;

#[derive(Debug, Eq, PartialEq, HasFields, BuildField)]
pub struct Employee {
    pub employee_id: u64,
    pub first_name: String,
    pub last_name: String,
}

#[derive(Debug, Eq, PartialEq, HasFields, BuildField)]
pub struct Person {
    pub first_name: String,
    pub last_name: String,
}

#[derive(Debug, Eq, PartialEq, HasFields, BuildField)]
pub struct EmployeeId {
    pub employee_id: u64,
}

fn main() {
    let person = Person {
        first_name: "Ada".to_owned(),
        last_name: "Lovelace".to_owned(),
    };

    let employee_id = EmployeeId { employee_id: 7 };

    let employee: Employee = Employee::builder()
        .build_from(person)
        .build_from(employee_id)
        .finalize_build();

    assert_eq!(
        employee,
        Employee {
            employee_id: 7,
            first_name: "Ada".to_owned(),
            last_name: "Lovelace".to_owned(),
        }
    );

    let employee2: Employee = Employee::builder()
        .build_from(Person {
            first_name: "Grace".to_owned(),
            last_name: "Hopper".to_owned(),
        })
        .build_field(PhantomData::<Symbol!("employee_id")>, 99)
        .finalize_build();

    assert_eq!(employee2.employee_id, 99);
}
```

## What each step means

- `Employee::builder()` starts a partial `Employee` builder with no fields filled.
- `.build_from(person)` copies matching fields from `Person` (`first_name`, `last_name`).
- `.build_from(employee_id)` copies `employee_id` from `EmployeeId`.
- `.build_field(PhantomData::<Symbol!("employee_id")>, 99)` sets one field directly by field tag.
- `.finalize_build()` returns `Employee`.

This is why it is called extensible: different modules can contribute different field bundles.

## Where extensibility comes from

- `Person` does not need to know about `Employee`.
- `EmployeeId` does not need to know about `Employee`.
- The composition happens at the build site by chaining `build_from` calls.

So you can keep field producers small and reusable, then assemble bigger targets as needed.

## Name and type checklist

- The `Symbol!("employee_id")` tag must match a real target field name.
- Types must match (`employee_id` is `u64`, so value must be `u64`).
- `finalize_build()` works only after all required fields are present.

Practical rule:

- use `build_from` for multi-field pieces,
- use `build_field` for one-off values.
