---
title: rust学习笔记
tags:
  - rust
status: doing
createDate: 2023-12-28
---

## 小技巧

### 整理溢出

要显式处理可能的溢出，可以使用标准库针对原始数字类型提供的这些方法：

- 使用 `wrapping_*` 方法在所有模式下都按照补码循环溢出规则处理，例如 `wrapping_add`
- 如果使用 `checked_*` 方法时发生溢出，则返回 `None` 值
- 使用 `overflowing_*` 方法返回该值和一个指示是否存在溢出的布尔值
- 使用 `saturating_*` 方法使值达到最小值或最大值

```rust
fn main() {
    let a : u8 = 255;
    println!("{:?}", a.wrapping_add(20));     // 19
    println!("{:?}", a.checked_add(20));      // None
    println!("{:?}", a.overflowing_add(20));  // (19, true)
    println!("{:?}", a.saturating_add(20));   // 255
}
```


## 引用和借用

**引用的生命周期从借用处开始，一直持续到最后一次使用的地方**。

## 认识生命周期

- **标记的生命周期只是为了取悦编译器，让编译器不要难为我们**
- **在通过函数签名指定生命周期参数时，我们并没有改变传入引用或者返回引用的真实生命周期，而是告诉编译器当不满足此约束条件时，就拒绝编译通过**。
- **函数的返回值如果是一个引用类型，那么它的生命周期只会来源于**：
    - 函数参数的生命周期
    - 函数体中某个新建引用的生命周期
- **三条消除规则**
    - 1. **每一个引用参数都会获得独自的生命周期**
        例如一个引用参数的函数就有一个生命周期标注: `fn foo<'a>(x: &'a i32)`，两个引用参数的有两个生命周期标注:`fn foo<'a, 'b>(x: &'a i32, y: &'b i32)`, 依此类推。
    2. **若只有一个输入生命周期(函数参数中只有一个引用类型)，那么该生命周期会被赋给所有的输出生命周期**，也就是所有返回值的生命周期都等于该输入生命周期
        例如函数 `fn foo(x: &i32) -> &i32`，`x` 参数的生命周期会被自动赋给返回值 `&i32`，因此该函数等同于 `fn foo<'a>(x: &'a i32) -> &'a i32`
    3. **若存在多个输入生命周期，且其中一个是 `&self` 或 `&mut self`，则 `&self` 的生命周期被赋给所有的输出生命周期**
        拥有 `&self` 形式的参数，说明该函数是一个 `方法`，该规则让方法的使用便利度大幅提升。