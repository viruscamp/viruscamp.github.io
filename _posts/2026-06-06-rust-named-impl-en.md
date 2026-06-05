---
title: 'Rust named impl Draft'
date: 2026-06-06
permalink: /posts/2026/06/rust-named-impl-en/
tags:
  - rust
---

# Rust named impl Draft

The purpose is clear: to bypass Rust's orphan rule.  
The proposed syntax should be more fluent to write, more natural to read, and more complete than the referenced proposals.  

1. Introduce named trait implementations (named-impl)
```rust
pub impl<T> Trait1 for Struct1<T> use ST1 {
    fn trait1_fn(&self) {
        println!("named impl ST1");
    }
}
```
2. Introduce a new type definition syntax, the Implementation Selection Variant (Impl Variant for short)
```rust
type Struct1Default = Struct1<i32> + _ use default;
type Struct1WithST1 = Struct1<i32> + Trait1 use ST1;
```


## Overview
```rust
// Default impl for struct and trait, same as Rust 2024 edition
pub struct Struct1<T>(T);
pub trait Trait1 {
    fn trait1_fn(&self);
}
impl<T> Trait1 for Struct1<T> {
    fn trait1_fn(&self) {
        println!("default impl");
    }
}

// New named-impl definition, can be defined in any crate
// ST1 below lives in the Type Namespace
pub impl<T> Trait1 for Struct1<T> use ST1 {
    fn trait1_fn(&self) {
        println!("named impl ST1");
    }
}

// Forbidden: using `default` as a named-impl name
pub impl Clone for i32 use default {} // ❌

// Import with `use` just like structs and traits; the full path mod1::ST1 can also be used
use mod1::{Struct1, Trait1, ST1};

// Type definitions
type Struct1Generic = Struct1<i32>;
// A generic type, which by default resolves to the following line: Struct1<i32> + _ use default

type Struct1Default = Struct1<i32> + _ use default;
// A concrete type, behaving exactly like Struct1 in Rust 2024; no named-impl is used.
// `default` is a weak keyword here.

type Struct1WithST1 = Struct1<i32> + Trait1 use ST1;
// A concrete type, same layout as Struct1, but uses ST1's implementation for Trait1.
// The trailing `_ use default` is implicitly omitted.

// Different impl => different types
assert!(TypeId::of::<Struct1Default>() != TypeId::of::<Struct1WithST1>());

// Complex definition 1
type ComplexType1 = &(i32 + Trait1 use ST1) + Trait2 use ST2;

// Complex definition 2: overlapping implementations and priority rules
type ComplexType2 = Struct1<i64>
    + From<i32> use mod2::Struct1Fromi32
    + From<_: Add> use default
    + From<_> use StructFrom
    + Trait1 use ST1;
// Multiple named-impl generic parameters; overlapping of traits is allowed.
// When acting as From<i32>, Struct1Fromi32 is used; as From<&str>, StructFrom is used.
// The first matching trait (before `use`) from left to right wins and its implementation is used.


// Type conversions
let s = Struct1(1); // defaults to Struct1<i32> + _ use default
let mut s_ref_named: &(Struct1<i32> + Trait1 use ST1) = &s; // ✅ type conversion
let s_ref_default = s_ref_named as &(Struct1<i32> + _ use default); // ✅ mutual conversion
s_ref_named = s_ref_default; // ❌ implicit conversion without explicit type annotation is forbidden

let s4 = Struct1(4); // defaults to Struct1<i32> + _ use default
let s5: Struct1<i32> + Trait1 use ST1 = s4; // move plus type conversion

let vs: Vec<Struct1<i32>> = vec![]; // defaults to Vec<Struct1<i32> + _ use default>
let vs1: &Vec<Struct1<i32> + Trait1 use ST1> = &vs; // Struct1 with impl variant can appear anywhere in the type
let vs2: &Vec<Struct1<i32> + _ use default> = vs1;


// End usage
// Any T: Trait1 can be instantiated with both Struct1<i32> + _ use default and Struct1<i32> + Trait1 use ST1

// 1. Direct call
s5.trait1_fn();

// 2. UFCS
<Struct1<i32> as Trait1 use ST1>::trait1_fn(&s5);

// 3. Generic function
fn use_trait1<T: Trait1>(a: &T) {
    a.trait1_fn();
}

use_trait1(s_ref_default); // default impl
use_trait1(s_ref_named); // named impl ST1

// 4. Generic type
pub struct Struct2<'a, T: Trait1>(&'a T);
impl Struct2<'a, T: Trait1> {
    pub fn use_inner(&self) {
        self.0.trait1_fn();
    }
}

let s21 = Struct2(s_ref_default);
s21.use_inner(); //  default impl

let s22 = Struct2(s_ref_named);
s22.use_inner(); // named impl ST1


// trait object
let mut dyn_trait1: &dyn Trait1;

dyn_trait1 = s_ref_named;
dyn_trait1.use_inner(); // named impl ST1

dyn_trait1 = s_ref_default;
dyn_trait1.use_inner(); //  default impl


// impl trait
// 1. `impl trait` in argument position is merely syntactic sugar for generics
fn use_trait1(a: &impl Trait1) {
    a.trait1_fn();
}
use_trait1(s_ref_default); // default impl
use_trait1(s_ref_named); // named impl ST1

// 2. `impl trait` in return position
fn return_trait1() -> impl Trait1 {
    let s4 = Struct1(4); // defaults to Struct1<i32> + _ use default
    return s4 as (Struct1<i32> + Trait1 use ST1); // move plus type conversion
}

```

## Prohibition Rules
1. Named-impl must be forbidden for `Copy`, `Drop`, `Send`, `Sync`, `Unpin`, etc.
```rust
#[disable_named_impl(all)]
pub trait Copy {}
```
`#[disable_named_impl(all)]` can be used by external code and placed on traits, structs, enums, etc.

2. Named-impl must be forbidden for `Hash`, `Ord`, `PartialOrd`, `Eq`, `PartialEq`, etc. at least when a default implementation exists.
```rust
#[disable_named_impl(exist_default)]
pub trait Hash {}
```
`#[disable_named_impl(exist_default)]` can be used by external code and placed on traits, structs, enums, etc.

## Extended Usages
1. Implementing external traits for external types, bypassing the orphan rule.

2. Compile-time proxy, or function decorator  
   (akin to `java.lang.reflect.Proxy` or Python decorators)
```rust
impl fmt::Display for i32 use ProxyDisplay {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "before ")?;
        (self as &(i32 + Display use default)).fmt(f)?;
        write!(f, " after")?;
        Ok(())
    }
}

// Generic proxy, but might be impossible to write due to recursion
impl<T: Display> fmt::Display for T use ProxyDisplayDefault {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "before ")?;
        (self as &(T + Display use default)).fmt(f)?; // ❓
        write!(f, " after")?;
        Ok(())
    }
}
```

3. Specialization, partial specialization
```rust
// Overlapping implementations and priority rules
type ComplexType2 = Struct1<i64>
    + From<i32> use mod2::Struct1Fromi32
    + From<_: Add> use default
    + From<_> use StructFrom
    + Trait1 use ST1;
// Multiple named-impl generic parameters; overlapping of traits is allowed.
// When acting as From<i32>, Struct1Fromi32 is used; as From<&str>, StructFrom is used.
// The first matching trait (before `use`) from left to right wins.
```

## Implicit Type Generics, Optional
1. Generic bounds can be structs or enums
```rust
#[derive(Default)]
struct Struct1;
impl Trait1 for Struct1 {};
impl Trait1 for Struct1 use TS1 {};
fn use_struct1<T: Struct1>() {
    T::default().trait1_fn();
}

use_struct1<Struct1 + _ use default>();
use_struct1<Struct1 + Trait1 use TS1>();
```

2. Implicit type generics: function parameters and return types become generic
- Advantage: old-style functions can automatically work with named-impl types.
- Disadvantage: unsound, breaks the function's original intent.
```rust
fn use_struct1(s: &Struct1) {
    s.trait1_fn();
}
// Automatically translates to
fn use_struct1<T: Struct1>(s: &T) {
    s.trait1_fn();
}
```
- Problem 1: Multiple `Struct1`s would be forced to the same type.
- Problem 2: If a parameter is `i32` and becomes generic, it destroys logic inside the function that relies on that `i32` (e.g., `Add` or `PartialEq`).

## Simplified Syntax
1. Named-impl selection parameter simplification
```rust
type Struct1Default = Struct1<i32> + use default;
type Struct1WithST1 = Struct1<i32> + use ST1;
type ComplexType = Struct1<i64> + use Struct1Fromi32<i32> + use StructFrom<_> + use ST1;
// The trait type can be inferred; generic parameters follow the impl name.
```

## Alternative Syntax
1. Use `as` instead of `use`
```rust
pub impl<T> Trait1 for Struct1<T> as ST1 {
    fn trait1_fn(&self) {
        println!("named impl ST1");
    }
}
type Struct1Default = Struct1<i32> + _ as default;
type Struct1WithST1 = Struct1<i32> + Trait1 as ST1;

<Struct1<i32> as Trait1 as ST1>::trait1_fn(&s5);
```

2. Named-impl selection parameters inside `<>` after generic parameters
```rust
type i32AlterDisplay = i32<i32, Display use AlterDisplay>;
type Struct1Default = Struct1<i32, _ use default>;
type Struct1WithST1 = Struct1<i32, Trait1 use ST1>;
type ComplexType = Struct1<i64, From<i32> use Struct1Fromi32, From<_> use StructFrom, Trait1 use ST1>;

//❓ How to express this with the alternative syntax?
type ComplexType1 = &(i32 + Trait1 as ST1) + Trait2 as ST2;
```

## Trait Inheritance Issues
1. Type bloat.
2. Whether a subtrait's named-impl can restrict which implementation of the supertrait is used.
3. More complex upcasting.
```rust
trait T1 {}
trait T2 : T1 {}

struct S;
impl T1 for S {}
impl T2 for S {}

impl T1 for S use T1S3 {}
impl T2 for S use T2S3 where S: T1 use T1S3 {} //❓ Optional, forces T2S3 to only be used with T1S3

// Bad news: type bloat
// Good news: the four types are not automatically generated; they don't exist unless written (compile-time and runtime)
let s1: S + _ use default = S;
let s2: S + T1 use T1S3 = S;
let s3: S + T2 use T2S3 = S; // If restricted, it is automatically inferred as the next
let s4: S + T1 use T1S3 + T2 use T2S3 = S;

type Invalid = S + T1 use default + T2 use T2S3; //❌ If the restrictive syntax above is present
``` 

## Compatibility Breakage
1. FFI boundaries.
2. dylib exported functions.
3. All code that assumes trait object data pointers are the same to judge identity will break.
```rust
fn is_same(a: &dyn T1, b: &dyn T1) -> bool {
	let (ptr_a, vt_a) = unsafe { transmute::<_, (usize, usize)>(a) }
	let (ptr_b, vt_b) = unsafe { transmute::<_, (usize, usize)>(b) }
	return ptr_a == ptr_b;
}
```

## Negative Implementations (negative-impls)
1. [negative-impls](https://doc.rust-lang.org/unstable-book/language-features/negative-impls.html) cannot be used for named-impl
```rust
pub impl !Trait1 for i32 use NegTrait1 {} // ❌
```
2. Overlap checking between named-impl and negative-impls is deferred to type monomorphization time
```rust
#![feature(negative_impls)]
trait DerefMut { }
impl<T: ?Sized> !DerefMut for &T { }

// Overlap with other implementations (including !Trait) is NOT checked at the definition of named-impl
impl DerefMut for &i32 use DerefMutForI32 {} // ✅

// The overlap check between named-impl and negative-impls happens at type instantiation
type X = &i32 + DerefMut use DerefMutForI32; // ❌
```

## GAT
Orthogonality with GAT
```rust
type A1 = S + Iterator use SI64;  // ✅
type A2 = S + Iterator<Item=i64> use SI64;  // ❌
```
```rust
struct S;

impl Iterator for S {
    type Item = i32;
    fn next(&mut self) -> Option<Self::Item> { todo!() }
}

impl Iterator for S use SI64{
    type Item = i64;
    fn next(&mut self) -> Option<Self::Item> { todo!() }
}

fn use_iter<T: Iterator>() -> T::Item { todo!() }
fn use_iter_i32<T: Iterator<Item=i32>>() -> T::Item { todo!() }
fn use_iter_i64<T: Iterator<Item=i64>>() -> T::Item { todo!() }

use_iter::<S>();      // ✅
use_iter_i32::<S>();  // ✅
use_iter_i64::<S>();  // ❌

use_iter::<S + Iterator use SI64>();     // ✅
use_iter_i32::<S + Iterator use SI64>(); // ❌
use_iter_i64::<S + Iterator use SI64>(); // ✅
```


## References
- https://internals.rust-lang.org/t/pre-rfc-selectable-trait-implementations/23829
- https://internals.rust-lang.org/t/pre-rfc-scoped-impl-trait-for-type/19923
- https://internals.rust-lang.org/t/looking-for-rfc-coauthors-on-named-impls/6275
- https://internals.rust-lang.org/t/named-impls-and-impl-generics/22158
