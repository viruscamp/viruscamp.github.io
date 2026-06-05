---
title: 'Rust named impl 草案'
date: 2026-05-28
permalink: /posts/2026/05/rust-named-impl/
tags:
  - rust
---

# Rust named impl 草案

目的很明确，绕过 rust 的孤儿规则。  
应该比参考链接的方案写法更流利，读法更自然，且语法更完善。  

## 概述
```rust
// struct trait 默认 impl, 与当前版本2024相同
pub struct Struct1<T>(T);
pub trait Trait1 {
    fn trait1_fn(&self);
}
impl<T> Trait1 for Struct1<T> {
    fn trait1_fn(&self) {
        println!("default impl");
    }
}

// 新增 named-impl 定义, 可定义在任意 crate
// 下面的 ST1 在 Type Namespace 中
pub impl<T> Trait1 for Struct1<T> use ST1 {
    fn trait1_fn(&self) {
        println!("named impl ST1");
    }
}

// 禁止使用 default 命名 named-impl
pub impl Clone for i32 use default {} // ❌

// 像 struct 和 trait 一样用 use 导入, 也可写全名 mod1::ST1
use mod1::{Struct1, Trait1, ST1};

// 类型定义
type Struct1Generic = Struct1<i32>;
// 泛型类型, 默认推导为下一行的 Struct1<i32, _ use default>

type Struct1Default = Struct1<i32, _ use default>;
//具体类型, 与当前版本2024 Struct1 行为完全相同, 不使用任何 named-impl
//此处 default 为弱关键字 weak keyword

type Struct1WithST1 = Struct1<i32, Trait1 use ST1>;
//具体类型, 布局与 Struct1 相同, 作为 Trait1 时使用 ST1 的实现， 默认省略了最后的 _ use default

// 实现重叠与优先级规则
type ComplexType = Struct1<i64,
    From<i32> use mod2::Struct1Fromi32,
    From<_: Add> use default,
    From<_> use StructFrom, 
    Trait1 use ST1
>;
// 多个 named-impl 泛型参数, trait 之间可以有重叠overlap
// 作为 From<i32> 时使用 Struct1Fromi32 , 作为 From<&str> 时使用 StructFrom
// 从左到右找到第一个匹配的trait(use 前)后, 使用对应的实现

// 有不同 impl 就不是同一类型
assert!(TypeId::of::<Struct1Default>() != TypeId::of::<Struct1WithST1>());


// 类型转换
let s = Struct1(1); // 默认推导为 Struct1<i32, _ use default>
let mut s_ref_named: &Struct1<i32, Trait1 use ST1> = &s; // ✅ 类型转换
let s_ref_default = s_ref_named as &Struct1<i32, _ use default>; // ✅ 多向互转
s_ref_named = s_ref_default; // ❌ 禁止没有显式类型的自动转换

let s4 = Struct1(4); // 默认推导为 Struct1<i32, _ use default>
let s5: Struct1<i32, Trait1 use ST1> = s4; // move 加类型转换

let vs: Vec<Struct1<i32>> = vec![]; // 默认推导为 Vec<Struct1<i32, _ use default>>
let vs1: &Vec<Struct1<i32, Trait1 use ST1>> = &vs; // 类型定义中任意位置的 Struct1
let vs2: &Vec<Struct1<i32, _ use default>> = vs1;


// 最终使用
// 任意 T: Trait1 的 T 均可使用 Struct1<i32, _ use default> 和 Struct1<i32, Trait1 use ST1> 来类型实例化

// 1. 直接调用
s5.trait1_fn();

// 2. UFCS
<Struct1<i32> as Trait1 use ST1>::trait1_fn(&s5);

// 3. 泛型函数
fn use_trait1<T: Trait1>(a: &T) {
    a.trait1_fn();
}

use_trait1(s_ref_default); // default impl
use_trait1(s_ref_named); // named impl ST1

// 4. 泛型类型
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
// 1. 参数位置的 impl trait 仅为泛型参数的简写
fn use_trait1(a: &impl Trait1) {
    a.trait1_fn();
}
use_trait1(s_ref_default); // default impl
use_trait1(s_ref_named); // named impl ST1

// 2. 返回值位置的 impl trait
fn return_trait1() -> impl Trait1 {
    let s4 = Struct1(4); // 默认推导为 Struct1<i32, _ use default>
    return s4 as Struct1<i32, Trait1 use ST1>; // move 加类型转换
}

```

## 禁止规则
1. `Copy`, `Drop`, `Send`, `Sync`, `Unpin` 等必须禁止 named-impl .
```rust
#[disable_named_impl(all)]
pub trait Copy {}
```
`#[disable_named_impl(all)]` 可供外部代码使用, 可置于 trait struct enum 等

2. `Hash` `PartialOrd` 等至少在有默认实现时禁止 named-impl .
```rust
#[disable_named_impl(exist_default)]
pub trait Hash {}
```
`#[disable_named_impl(exist_default)]` 可供外部代码使用, 可置于 trait struct enum 等

## 扩展用法
1. 对外部类型实现外部trait , 绕过孤儿规则

2. 编译期代理，或函数装饰器
java.lang.reflect.Proxy , Python Decorator
```rust
impl fmt::Display for i32 use ProxyDisplay {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "before ")?;
        (self as &i32<Display use default>).fmt(f)?;
        write!(f, " after")?;
        Ok(())
    }
}

// 通用代理, 但可能因为递归写不出
impl<T: Display> fmt::Display for T use ProxyDisplayDefault {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "before ")?;
        (self as &T<Display use default>).fmt(f)?;
        write!(f, " after")?;
        Ok(())
    }
}
```

3. 特化, 偏特化
```rust
// 实现重叠与优先级规则
type ComplexType = Struct1<i64,
    From<i32> use Struct1Fromi32,
    From<_: Add> use default,
    From<_> use StructFrom, 
    Trait1 use ST1
>;
// 多个 named-impl 泛型参数, trait 之间可以有重叠overlap
// 作为 From<i32> 时使用 Struct1Fromi32 , 作为 From<&str> 时使用 StructFrom
// 从左到右找到第一个匹配的trait(use 前)后, 使用对应的实现
```

## 类型隐式泛型, 可选
1. 泛型约束可为 struct enum
```rust
#[derive(Default)]
struct Struct1;
impl Trait1 for Struct1 {};
impl Trait1 for Struct1 use TS1 {};
fn use_struct1<T: Struct1>() {
    T::default().trait1_fn();
}

use_struct1<Struct1<_ use default>>();
use_struct1<Struct1<Trait1 use TS1>>();
```

2. 类型隐式泛型, 函数参数和返回值都变成泛型
- 优点: 旧式函数可自动用于带 named-impl 的类型
- 缺点: unsound , 破坏函数原意
```rust
fn use_struct1(s: &Struct1) {
    s.trait1_fn();
}
// 自动翻译为
fn use_struct1<T: Struct1>(s: &T) {
    s.trait1_fn();
}
```
- 问题代码1 多 Struct1 只能为同一类型
- 问题代码2 参数 i32 变泛型后, 破坏函数内部与参数无关的 i32, 如有 Add 或 PartialEq 等将完全破坏逻辑

## 简化语法
1. named-impl 选择参数简化
```rust
type Struct1Default = Struct1<i32, use default>;
type Struct1WithST1 = Struct1<i32, use ST1>;
type ComplexType = Struct1<i64, use Struct1Fromi32<i32>, use StructFrom<_>, use ST1>;
// trait的类型可以推导得到, 泛型参数在 impl name 之后
```

## 替代语法
1. 用 `as` 而不是 `use`
```rust
pub impl<T> Trait1 for Struct1<T> as ST1 {
    fn trait1_fn(&self) {
        println!("named impl ST1");
    }
}
type Struct1Default = Struct1<i32, _ as default>;
type Struct1WithST1 = Struct1<i32, Trait1 as ST1>;

<Struct1<i32> as Trait1 as ST1>::trait1_fn(&s5);
```

2. named-impl 选择参数用 `+` 追加在类型之后
```rust
type Struct1Default = Struct1<i32> + _ use default;
type Struct1WithST1 = Struct1<i32> + Trait1 use ST1;
type ComplexType = Struct1<i64> + From<i32> use Struct1Fromi32 + From<_> use StructFrom + Trait1 use ST1;
```


## trait继承问题
1. 类型膨胀
2. subtrait 的 named-impl 是否是否可以选定其 super trait 的实现
3. 更复杂的 upcasting
```rust
trait T1 {}
trait T2 : T1 {}

struct S;
impl T1 for S {}
impl T2 for S {}

impl T1 for S as T1S3 {}
impl T2 for S as T2S3 {} // 是否可以在此处限制必须使用 T1S3 ?

// 坏消息: 类型膨胀
// 好消息：不是自动生成的4种类型，不写就不会存在（不管是编译时还是运行时）
let s1: S<_ use default> = S;
let s2: S<T1 use T1S3> = S;
let s3: S<T2 use T2S3> = S; // 如果有限制, 则自动推导为下一个
let s4: S<T1 use T1S3, T2 use T2S3> = S;
``` 

## 兼容性破坏
1. FFI 边界
2. dylib 导出函数
3. 所有假设 trait object 数据指针相同则判断为同一对象的代码出错
```rust
fn is_same(a: &dyn T1, b: &dyn T1) -> bool {
	let (ptr_a, vt_a) = unsafe { transmute::<_, (usize, usize)>(a) }
	let (ptr_b, vt_b) = unsafe { transmute::<_, (usize, usize)>(b) }
	return ptr_a == ptr_b;
}
```

## 否定性实现 negative-impls
1. [negative-impls](https://doc.rust-lang.org/unstable-book/language-features/negative-impls.html) 不能用于 named-impl
```rust
pub impl !Trait1 for i32 use NegTrait1 {} // ❌
```
2. 当存在 negative-impls 时, 禁止有重叠overlap的 named-impl
```rust
#![feature(negative_impls)]
trait DerefMut { }
impl<T: ?Sized> !DerefMut for &T { }

impl DerefMut for &i32 use DerefMutForI32 {} // ❌
```


## 参考
- https://internals.rust-lang.org/t/pre-rfc-selectable-trait-implementations/23829 最接近, 但没有名字
- https://internals.rust-lang.org/t/looking-for-rfc-coauthors-on-named-impls/6275
- https://internals.rust-lang.org/t/named-impls-and-impl-generics/22158
- https://internals.rust-lang.org/t/pre-rfc-scoped-impl-trait-for-type/19923 理论最完善