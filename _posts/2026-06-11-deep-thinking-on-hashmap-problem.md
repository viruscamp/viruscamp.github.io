---
title: 'Deep thinking on the hashmap problem of alternative trait implementation'
date: 2026-06-11
permalink: /posts/2026/06/thinking-on-hashmap-problem/
tags:
  - rust
---

# Deep thinking on the hashmap problem of alternative trait implementation

The earliest description is from [nikomatsakis](https://internals.rust-lang.org/t/named-and-scoped-trait-implementations-as-a-solution-to-orphan-rules-ok-it-wont-work/4711/10)
>We called it the hashtable problem because – imagine you had a hashtable with keys to type K that is built up using one impl of Hash, but then you pass that hashtable to another module, where a distinct Hash impl is in scope. It’s going to be pandemonium. What that e-mail proposed was to solve this by making the impl “part of the type”, in a sense. At the end of the day it seemed like a lot of complexity and we opted against it – but there are real costs. I’m not sure of the best fix here.

The problem affects:
- struct `HashMap` with trait `Hash`
- struct `BTreeMap` with trait `Ord`

1. example in typical scoped impl
```rust
let mut map = HashMap::<i32, &'static str>::new();
map.insert(42, "the answer to the universe"); // default hash of i32
{
     use AnotherHashForI32 as impl Hash for i32;
     let found = map.get(&42); // a different hash of i32, can we find the answer?
}
```

2. example in typical named impl
```rust
let mut map = HashMap::<i32, &'static str>::new();
map.insert(42, "the answer to the universe"); // default hash of i32

let altmap = &map as &HashMap<i32 + Hash use AnotherHashForI32, &'static str>;
let found = altmap.get(&(42 as (i32 + Hash use AnotherHashForI32))); // a different hash of i32, can we find the answer?
```

## Where is the problem from?
```rust
let key = 42;
let mut map = HashMap::<i32, &'static str>::new();
map.insert(key, "the answer to the universe"); // default hash of i32
let found = map.get(&key); // ok

let altkey = key as (i32 + Hash use AnotherHashForI32);
let mut altmap = HashMap::<i32 + Hash use AnotherHashForI32, &'static str>::new();
altmap.insert(altkey, "the answer to the universe"); // a different hash of i32
let found = altmap.get(&altkey);  // ok
```
The code above works fine.

The problem is from the type casting of the container:
```rust
let altmap = &map as &HashMap<i32 + Hash use AnotherHashForI32, &'static str>;
```

With scoped impl, there are also implicitly type casting: 
```rust
let mut map = HashMap::<i32, &'static str>::new();
map.insert(42, "the answer to the universe");
{
     use AnotherHashForI32 as impl Hash for i32;
     let found = map.get(&42); // `map` is casted, and `&42` is casted
}
```

## Solutions
1. the most strict way  
  Only allow casting of `T <-> T + Impl`, `&T <-> &(T + Impl)`, and `&mut T <-> &mut(T + Impl)`  
  This is just a better newtype pattern.  

2. the most detailed way  
  Disable implementation casting from `HashMap<K + Hash use HashImpl1, _>` to `HashMap<K + Hash use HashImpl2, _>`  
    ```rust
    pub struct HashMap<#[disable_impl_casting(Hash)]K: Hash, V> {}
    ```
  But there are still unresolved problems:  
  - nested types `HashMap<(i32 + Hash use HashImpl1, i32), _>`
  - indirect implements
    ```rust
    struct Key;
    impl Hash for Key {
        fn hash<H: Hasher>(&self, state: &mut H) {
            format!("{self}").hash(state); // will impl of Display changed?
        }
    }
    impl Display for Key {
        fn fmt(&self, f: &mut Formatter<'_>) -> Result<(), Error> {
            write!(f, "Key")
        }
    }
    impl Display for Key use AlterDisplay {
        fn fmt(&self, f: &mut Formatter<'_>) -> Result<(), Error> {
            write!(f, "Key use AlterDisplay")
        }
    }
  
    let map = HashMap::<Key, i32>::new();
    let altmap = &map as &HashMap<Key + Display use AlterDisplay, i32>;
    ```

3. disable implementation casting on `HashMap`
    ```rust
    #[disable_impl_casting]
    pub struct HashMap<K, V> {}
    ```

4. disable implementation casting on `HashMap` with `K`
    ```rust
    pub struct HashMap<#[disable_impl_casting(all)]K: Hash, V> {}
    ```

5. disable alternative impl of `Hash`, indirect implements is also the problem  
    In [My Named impl draft](https://internals.rust-lang.org/t/named-impl-with-implementation-selection-variant/24374), this is selected:  
    ```rust
    #[disable_named_impl(exist_default)]
    pub trait Hash {}
    ```
