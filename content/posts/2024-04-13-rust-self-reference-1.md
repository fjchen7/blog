---
title: "Rust的自引用（1）：不安全的移动"
date: 2024-04-13
categories: ["编程"]
tags: ["Rust", "编程"]
summary: 'Rust中的自引用结构个很神奇的结构'
---

自引用结构体，是指一种结构体，它内部有某个引用，指向自身的一部分。

这种结构体在某些场景下很有用。Rust编译器实现的[`Future`](https://doc.rust-lang.org/std/future/trait.Future.html)就是自引用结构体。它是一个状态机，所有的`sync`代码块或函数都会被编译器转化成这种结构。

`Future`可能被随时调度和执行，因此需要保存它当前的运行状态，以能下次继续执行。保存的状态，包括了它内部变量之间的关系。

```rust
async {
    let mut s = String::from("foo");
    let s_ref = &s;
    println!("{:?}", s_ref);
}
```

比如这段异步代码，当`Future`执行到`let s_ref = &s`后停下来，你可以认为，它内部要存两个值：`s`和`s_ref`，且`s_ref`要指向`s`。这就是一个自引用的关系。

在这篇文章里，让我们来思考一个问题：如果你是一个库的开发者，要实现一个自引用结构体，那应该怎么做？

## 目标

```rust
struct SelfRef {
	a: Data,
	b: Ptr,  // 我们希望b指向a
}
```

我们的设计目标：实现安全的自引用结构体——如果以safe方式操作这个结构体，那永远不会出现未定义行为（[Undefined Behavior，UB](https://doc.rust-lang.org/reference/behavior-considered-undefined.html)）。

那应该怎么做呢？基于自引用结构体的定义，我们需要保证引用`b`永远都指向自身的字段`a`，也就是总是保持自引用关系。

这个定义的实现就是它的安全保证。可以这么理解：如果结构体总是「自引用」的，那它的方法必定都是按照这个特点实现的。想象一下，如果在safe代码下，用户能以某种方式让引用`b`指向其它数据，那这些方法的安全前提就被破坏了。执行它们可能出现无法预期的行为。比如，用户让引用`b`变成空指针，或者指向类型完全不同的值，那访问就会出现问题。

当然，我们可以要求用户在使用前，通过检查指针来保证安全。但这是通过使用来保证安全，而不是通过设计来保证。否则，我们就退回到C++编程上了。

接下来，开始我们的尝试吧。

## 第一次尝试：`&T`

先考虑完全用safe代码来实现它。我们定义结构体：

```rust
struct SelfRef<'a> {
	a: String,
	b: &'a str,  // 我们希望b指向a
}
```

接着先用笨方法来初始化它：

```rust
let s = String::from("");
let mut v = SelfRef {
    a: String::from("hello"),
    b: &s,
};
v.b = &v.a;
```

由于`b`不能是空指针，于是先给它一个占位符，等结构体初始化完成后再重新赋值。看着有点丑，但好歹我们完成了，现在`v.b`指向的是`v.a`。

这个过程肯定不能暴露给用户，否则无法保证它永远是自引用的。我们进行封装：

```rust
impl<'a> SelfRef<'a> {
    fn new(data: &str) -> Self {
        let s = String::from("");
        let mut v = Self {
            a: String::from(data),
            b: &s,
        };
        v.b = &v.a;
        v
    }
}
```

毫无疑问，编译器报错了：

```rust
cannot move out of `v` because it is borrowed
   returning this value requires that `v.a` is borrowed for `'a`
```

我们在`new`方法里创建了`v`，且`v.b`指向的是当前`v`的某个内存区域。但当`new`方法返回时，`v`中的数据移动到了其他的内存位置，原本`v.b`指向的内存就失效了。这违反了Rust的借用规则。

我们没法解决这个问题。似乎用`&T`无法实现自引用结构体。

## 第二次尝试：`*const T`

### 初始化

编译器报错的原因，在于这个结构体的生命周期标注。那我们引入unsafe代码，用裸指针代替引用，去掉生命周期标注，是否可行？让我们试试：

```rust
struct SelfRef {
    a: String,
    b: *const String,  // 裸指针
}
```

继续实现初始化方法：

```rust
impl SelfRef {
    fn new(data: &str) -> Self {
        Self {
            a: String::from(data),
            b: std::ptr::null(),  // 先初始化为空指针
        }
    }

    fn init(&mut self) {
        let self_ptr: *const String = &self.a;
        self.b = self_ptr;  // 指向自身
    }
}
```

用户可以用以下方式初始化：

```rust
let mut v = SelfRef::new("hello");
v.init();
```

`new`方法返回时，发生了移动，所以我们无法获得`a`的地址。因此我们分成了两个方法来初始化。现在，当裸指针`b`非空时，那就可以确定它是指向自身的。

唯一的缺陷是，用户可能忘记调用`init`来初始化，使得`b`是空指针。但我们在代码里可以做对应的检查，比如：

```rust
impl SelfRef {
    fn do_something(&mut self) {
        if self.b.is_null() {
            self.init();
        }
        // do something
    }
}
```

看来初始化的问题解决了。

### 添加方法

接着来尝试给它增加一些方法：

```rust
impl SelfRef {
    fn set_a(&mut self, data: &str) {
        self.a = String::from(data);
    }

    fn b(&mut self) -> &String {
        assert!(!self.b.is_null(), "called without init");
        unsafe { &*(self.b) }
    }
}
```

使用

```rust
let mut v = SelfRef::new("hello");
v.init();
println!("{:?}", v.b()); // hello
v.set_a("world");
println!("{:?}", v.b()); // world
```

很不错！即使是修改了`v.b`的值，`v.a`依然指向它。看起来，只要不暴露任何能修改指针的方式，那它永远是自引用的，是合法的。

## 不安全的移动

似乎问题解决了，我们用裸指针实现了一个安全的自引用结构体。

但真的安全吗？很不幸，虽然我们小心谨慎，但它依然会在移动时出现问题。

### 问题

看这段代码：

```rust
let mut vv = v;
vv.set_a("foo");
println!("{:?}", vv.b());
```

预期是打印`foo`。但我执行了两次，结果都不一样（你的可能不同）：

```
// 第一次打印了乱码
"@@u\u{10}*"

// 第二次运行，直接崩溃
thread 'main' panicked at library/core/src/fmt/mod.rs:2463:26:
byte index 4 is not a char boundary; it is inside 'Ҍ' (bytes 3..5) of `@�Ҍ`
stack backtrace:
   0:        0x102921618 - std::backtrace_rs::backtrace::libunwind::trace::h8e34a2e8e90ca39c ...
```

我们移动了自引用结构体，接着就访问到了非法内存。这还是在safe代码中，是Rust完全不能容忍的。这背后发生了什么？

### 移动是什么

在解释原因前，我想再聊聊Rust中移动（move）的含义

```rust
let a = b;
```

这是一个值移动的操作。可以理解成它背后发生了这样的事：

1. 先为`a`变量开辟一个新内存
2. 将`b`的数据复制到`a`所指向的内存中
3. 清空`b`所指向的内存中原本的数据

这里的`a`和`b`都是在栈上，虽然涉及数据的复制，但非常快。而且Rust编译器会做优化（比如不一定清空`b`内存的数据），因此开销很小。

变量代表了某个内存地址，是固定的。而所有权针对的是值。如果`a`变量拥有值`x`的所有权，那就能理解为，`a`的地址存放了`x`的数据（或指向它的指针）。如果值`x`移动了，那可以说它的数据从某个内存区域转移到了另一个地方，所有权移动到了其他变量身上。

以下代码：

```rust
#[derive(Debug)]
struct Foo(i32);
let mut a = Foo(42);
println!("&a = {:?}", &a as *const _);
let b = Foo(30);
println!("&b = {:?}", &b as *const _);
a = b;
println!("&a = {:?}", &a as *const _);
```

我的打印结果（你的可能不同）

```
&a = 0x16b6357d0
&b = 0x16b635850
&a = 0x16b6357d0
```

两次打印的`a`地址都相同，变的是它内部存放的值。

这样，就能理解为什么移动自引用数据会出现问题：执行`let mut vv = v`后，`v`就释放了自身的内存。但`vv.b`的指向还是原来的`v.a`。此时访问`vv.b`就是非法的。

![](/images/rust-self-reference-1.png)

### 无处不在的移动

移动是Rust里的基本操作。从定义来说，它只是「将整块数据一同搬到某个内存」的行为。它不仅发生在操作`T`上，也能发生在操作`&mut T`时。后者的使用也很普遍，比如这几个方法：

- [`std::mem::swap(&mut T, &mut T)`](https://doc.rust-lang.org/std/mem/fn.swap.html)：交换两个变量的值。
- [`std::mem::take(&mut T)`](https://doc.rust-lang.org/std/mem/fn.take.html)：返回变量的值，并将其置为默认值。
- [`std::mem::replace(&mut T, T)`](https://doc.rust-lang.org/std/mem/fn.replace.html)：返回变量的值，并将其置为指定值。
- [`Option::take(&mut self)`](https://doc.rust-lang.org/std/option/enum.Option.html#method.take)：取出`Option`的包装值，并将`self`置为`None`。

这些方法，都是对`&mut T`所指向的值做替换，并把原来的值「移动」到了其他的内存区域。

这些方法不受限于任何特征或类型的实现，只需要`&mut T`。这样就很危险了——只要我们得到自引用结构体的`&mut T`，就能移动它的值，然后引发安全问题，比如让内部的引用指向非法的内存。

这非常危险，而且看起来没有解决的方式。

## 总结

我们尝试了两种方案实现自引用结构体：

- `&T`：普通的引用。无法限制结构体维持自引用的特点。
- `*const T`：裸指针。谨慎地暴露方法，可以让结构体保持自引用。但在数据移动时，可能出现非法的内存访问。

这两种方案都失败了。

自引用结构体的实现，并不是容易的事。在大名鼎鼎的Too Many Linked List中，[第一次对双向队列的实现](https://rust-unofficial.github.io/too-many-lists/fifth-layout.html)，就遇到了自引用问题。最后的方案是修改数据结构，使其不再是自引用，用这种方式绕过去（不过人家也是为了教学）。

那没有什么方案能实现自引用结构吗？当然有：

- 使用`Rc`和`RefCell`：不用引用，而是直接拥有所有权（严格来说，这就不是自引用结构体了）。但这种方式不仅增加代码理解的负担，维持`Rc`的引用计数也需要额外的开销。
- 使用`Pin`：既然自引用结构体在移动才会出现不安全问题，那限制它的移动就好了。这就是`Pin`的思路。

我们将在下节介绍`Pin`如何实现自引用结构体。