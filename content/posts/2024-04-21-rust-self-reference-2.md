---
title: "Rust的自引用（2）：Pin"
date: 2024-04-21
categories: ["编程"]
tags: ["Rust", "编程"]
summary: 'Rust中的自引用结构的解决方案：Pin'
---

自引用结构体在移动时出现问题的原因在于，数据已经移动到了其它内存里，但引用指向的还是原来的位置。既然在移动时才会出现不安全问题，那限制它的移动就好了。这就是`Pin`的思路。

## `Unpin`和`!Unpin`

Rust中有一个名为[`Unpin`](https://doc.rust-lang.org/std/marker/trait.Unpin.html)的特征。它是一个标记特征，不提供任何方法，只表示移动该类型是**安全的**。

（如果你不理解何为「移动是安全的」，最好阅读我的上一篇文章《*Rust的自引用（1）：不安全的移动*》）

编译器会为大部分类型自动实现`Unpin`特征，。如果你定义的结构体所包含的类型都是`Unpin`的，那编译器也会为它实现`Unpin`。因此，我们才能安全地移动大部分类型。

`!Unpin`则是`Unpin`的负向实现（`!`名为[`negative_impls`](https://doc.rust-lang.org/nightly/unstable-book/language-features/negative-impls.html)，表示某类型永远不会实现该特征）。你可以把`!Unpin`当作另一个特征，它表示移动该类型是**不安全的**。`Unpin`和`!Unpin`是互斥的，对同一个类型，无法同时实现它们（编程中没有薛定谔的猫:)）。

如果你定义的类型，在移动后可能带来安全问题，那就应该为它实现`!Unpin`（比如自引用结构体）。有两种办法能实现`!Unpin`：

- 使用`PhantomPinned`。带有这个类型的结构体，编译器不会为其实现`Unpin`。

```rust
struct Foo {
    _marker: PhantomPinned,
}
```

- 使用`impl`实现。

```rust
// 手动实现需要
// 1）使用nightly Rust
// 3）在模块层级加上：#![feature(negative_impls)]
struct Foo;
impl !Unpin for Foo {}
```

`!Unpin`也是标记特征——它不带来任何限制，你依然可以移动它。它只是在告诉编译器：移动这种类型是不安全的。但编译器听归听，也不对它进行任何处理。

`!Unpin`真正发挥作用的地方，在于它和`Pin`结合使用。

## `Pin`

[`Pin`](https://doc.rust-lang.org/std/pin/struct.Pin.html)是结构体。它是一个包装类型，用来包装指针。

```rust
pub struct Pin<Ptr> {
    pub pointer: Ptr,
}
```

`Pin`的效果是，当`Ptr`所指向的类型是`!Unpin`时，就会阻止它在内存中移动。但如果`Ptr`所指向的类型是`Unpin`，那就不会有任何行为限制。

这么说有点绕。可以从另一个角度理解——`Pin`用来**限制不安全的移动**。

- 如果类型是`Unpin`，那它的移动本来就是安全的，没必要限制。
- 如果类型是`!Unpin`，那它的移动不安全，于是要限制。

Pin这个单词本身的意思就是「固定」。它要「固定」那些移动时可能出现安全问题的类型，使其不能移动。通过限制不安全的移动，来使得剩下的移动都是安全的。这跟Rust的哲学一脉相承。

## 如何限制移动

前面是概念上的解释。这里从代码上，解释`Pin`如何进行限制。

我的上一篇文章《*Rust的自引用（1）：不安全的移动*》提到，你可以通过`T`或`&mut T`来移动值。现在来看，`Pin`如何限制两点。

### `T`

`Pin`类似一种指针。移动它，不会转移内部值的所有权，不发生移动。

```rust
struct Foo {
    _marker: PhantomPinned,
}
let foo = Foo {
    _marker: PhantomPinned,
};
let pin_var = Box::pin(foo); // 类型为：Pin<Box<Foo>>
println!("{:?}", pin_var.as_ref().get_ref() as *const _);
let pin_var2 = pin_var; // 安全，不会移动数据。
println!("{:?}", pin_var2.as_ref().get_ref() as *const _);
```

你会发现，两次打印的地址都一样，这正是因为值的内存位置被固定了。如果你发现打印出`0x1`，也不要惊讶，它也是地址。这只是因为`Foo`内部没有真正的值，编译器做了优化。

### `&mut T`

另一种移动值的方式，是通过`&mut T`。许多能移动值的方法，如[`std::mem::swap(&mut T, &mut T)`](https://doc.rust-lang.org/std/mem/fn.swap.html)和[`std::mem::take(&mut T)`](https://doc.rust-lang.org/std/mem/fn.take.html)等，接收的参数都是`&mut T`。

`Pin`对此的处理是（假设是`Pin<Ptr<T>>`）：

- 如果`T`实现的是`Unpin`，能拿到`&mut T`（没有限制）。
- 如果`T`实现的是`!Unpin`，不能拿到`&mut T`。

对于`Pin<Ptr<T>`，可以通过[`as_mut(&mut self)`](https://doc.rust-lang.org/std/pin/struct.Pin.html#method.as_mut)方法，以解引用的方式转换成`Pin<&mut T>`。然后再通过`Pin<&mut T>`的方法[`get_mut(self)`](https://doc.rust-lang.org/std/pin/struct.Pin.html#method.get_mut)，来得到`&mut T`。

```rust
let mut pin_var = Box::pin(1); // Pin<Box<i32>>
let pin_var = pin_var.as_mut(); // Pin<&mut i32>
let pin_var = pin_var.get_mut(); // &mut i32
```

但[`get_mut(self)`](https://doc.rust-lang.org/std/pin/struct.Pin.html#method.get_mut)存在限定

```
pub fn get_mut(self) -> &'a mut T
where
    T: Unpin
```

因此，如果`T`是`!Unpin`的，就不能调用`get_mut`方法。这样就无法通过`Pin<Ptr<T>>`来得到`&mut T`，更不可能调用那些接收`&mut T`的方法来移动值。我们安全了。

## 第三种引用

看到这里，好像`Pin`只是通过暴露方法，来限制用户拿到内部值的`&mut T`。看起来也没有什么特别的，我们完全能实现一个类似的类型。

而且这种设计似乎也没有什么用。拿不到`&mut T`，那意味着不能使用值的`&mut self`方法。难道要求我们只使用它的`&self`方法？如果需要修改数据，就在方法里用unsafe代码或内部可变性（如`RefCell<T>`）来修改内部的值？

当然不是这样。接下来要介绍`Pin`神奇的地方了——它能作为方法的`self`。

```rust
impl Foo {
    fn do_something(self: Pin<&mut Self>) {
        // 通过这个方法你能得到&mut Self，然后能修改它的值
        let this = unsafe { self.get_unchecked_mut() };
        println!("do something");
    }
}
```

`do_something`方法要通过`Pin`类型才能访问：

```rust
let mut foo = Foo {
    _marker: PhantomPinned,
};
// pin_var.do_something();  // 无法这么调用
let mut pin_var = Box::pin(foo); // Pin<Box<Foo>>
pin_var.as_mut().do_something();  // as_mut先将Pin<Box<Foo>>转换成Pin<&mut Foo>
```

那现在的路线就就很明确了。我们可以遵循以下方式设计一个`!Unpin`类型：

1. 只提供返回`Pin<Ptr<T>>`的构造方法。既然只在操作`Pin<Ptr<T>>`时才能保证行为安全，那限制用户只能获得这个类型就好了。我们不允许用户创建带有所有权的`T`，以阻止它进一步得到`&mut T`。
2. 提供`Pin<&mut Self>`方法。所有可变操作，都在这类方法上完成。
3. 可以提供`&self`方法。它们是安全的，且能通过`Pin<Ptr<T>>`来访问。
4. 不提供`&mut self`和`self`方法。它们是不安全的，而且也无法访问到。

这样，就能确保它的行为是安全的，且有足够的表达力。

我们可以将`Pin<&mut T>`看作是一种特殊的引用类型。它的效果是：

- 能像`&mut T`一样修改自身
- 但不能在内存中移动数据——不能拿到`&mut T`并传给其他函数。

这种能力跟`&T`和`&mut T`都不同，有点介于两者之间。因此，有人说`Pin<&mut T>`是Rust中的第三种引用，我表示认同。

## 重新实现自引用结构体

我的上一篇文章《*Rust的自引用（1）：不安全的移动*》中的自引用结构体实现失败了。让我们用`Pin`重新实现一下。

```rust
struct SelfRef {
    a: String,
    b: *const String, // 裸指针
    _marker: PhantomPinned,  // 实现!Unpin
}

impl SelfRef {
    fn new(data: &str) -> Pin<Box<Self>> {
        let v = Self {
            a: String::from(data),
            b: std::ptr::null(), // 先初始化为空指针
            _marker: PhantomPinned,
        };
        // 下面的语句执行后，所创建的SelfRef值在内存中的位置就是固定的
        let mut v = Box::pin(v); // 获得Pin<Box<Self>>
        let mv = unsafe { v.as_mut().get_unchecked_mut() }; // 获得&mut Self
        mv.b = &mv.a;  // 指向自身
        v
    }

    fn b(self: Pin<&Self>) -> &String {
        unsafe { &*(self.b) }
    }

    fn set_a(self: Pin<&mut Self>, data: &str) {
        let this = unsafe { self.get_unchecked_mut() }; // 拿到&mut T
        this.a = String::from(data);
    }
}
```

使用它

```rust
let v = SelfRef::new("hello"); // 返回Pin<Box<SelfRef>>
println!("v.a: {}", v.as_ref().b()); // 打印hello

let mut vv = v;
vv.as_mut().set_a("foo"); // 重新赋值vv.a
println!("{:?}", vv.as_ref().b()); // 打印foo。不会出问题，依然指向自身

// 无法被接收&mut T的方法调用，能保证这个自引用变量不发生移动
// std::mem::take(&mut vv.as_mut())
```

我们无法创建`SelfRef`类型的`T`、`&T`和`&mut T`，所有的操作都是`Pin<Ptr<T>>`上完成的，不会发生不安全的移动。这个实现还非常优雅，去掉了丑陋的`init`方法。Rust很开心，我们也很开心。

PS：尝试在不同地方执行这个语句，打印数据的内存地址：

```rust
let ptr: &SelfRef = vv.as_ref().get_ref();
println!("{:?}", ptr as *const _);
```

你会发现，地址都是一样的，因为不管怎么操作，数据都被固定在了内存的某片区域中，不能移动了。

## 总结

如果一个类型实现了`!Unpin`，那表示它在内存中移动时，可能出现不安全的行为。

Rust对这种类型的处理办法是，使用`Pin<Ptr<T>>`来**限制它在内存中的移动**。无法移动，体现在`Pin`不暴露`&mut T`，使得一系列接收`&mut T`的移动数据的方法，都无法在该类型上使用。

`Pin`的另一个能力是支持编写`self`为`Pin<&mut Self>`的方法。这样只需要暴露`Pin<Ptr<T>>`给用户即可，并不会削弱这种类型的表达能力。

目前，似乎只有自引用结构必须是`!Unpin`（如果发现了其他结构也需要，请告知我！）。这种类型一般使用了指向自身的裸指针。在内存中移动它，可能出现空指针或指向非法内存，因此要配合`Pin`使用。

在实际开发中，我们很少用到自引用结构体。即使需要，也往往可以用其他设计来代替，比如使用`Rc`和`RefCell`，让用户使用前做自行检查等等。

自引用结构体只是一种数据结构。它没有好坏之分，只是在某些场景下更适用。Rust编译器实现的`Future`是自引用结构体，内部有指向自身的引用，用来记录当前状态下内部数据的复杂关系。但你完全可以其他方式来实现`Future`，比如用某种数据结构来记这种关系。但这种需求无疑更适合自引用结构，性能也会更高。


## 参考

- [Unpin in std::marker - Rust](https://doc.rust-lang.org/nightly/std/marker/trait.Unpin.html)
- [Pinning - Asynchronous Programming in Rust](https://rust-lang.github.io/async-book/04_pinning/01_chapter.html)
- [定海神针 Pin 和 Unpin - Rust语言圣经(Rust Course)](https://course.rs/advance/async/pin-unpin.html)
