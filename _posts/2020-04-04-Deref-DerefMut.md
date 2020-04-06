---
layout: post
title: Deref and DerefMut trait in Rust
date: 2020-03-29 10:29
categories: Rust
tag: Rust
excerpt:  Deref and DerefMut trait in Rust
---
# Deref的基本概念

本文学习Rust中的Deref和DerefMut，通过实现Deref和DerefMut Trait，可以重载解引用运算符（*）。我非常喜欢上面描述中的重载二字，因为Rust中的Deref和DerefMut绝不是简简单单的解引用，而是允许我们强制隐式转换。

Deref和DerefMut trait的定义如下：

```Rust
trait Deref{
	type Target: ?Sized;
  fn deref(&self)-> &Self::Target;
}

trait DerefMut: Deref{
  fn deref_mut(&mut self) -> &mut Self::Target;
}
```

注意上面的返回值，和Clone Trait不同，返回的类型的Target类型，这种关联类型，而不是Self。这就意味着通过Deref或者DerefMut，可以得到一个Self::Target 类型的变量，而这个Self::Target类型的变量，是和原类型有关联的，属于原类型的一个部分。	

下面看一个例子：

```Rust
struct Selector <T> {
	elements: Vec<T> ,
  current: usize 
}

use std::ops::{Deref, DerefMut} ;

impl<T> Deref for Selector<T>{
  type Target = T ;
  fn deref(&self)->&T {
    &self.elements[self.current]
  }
}

impl<T> DerefMut for Selector<T>{
  type Target=T ;
  fn deref_mut(&self)->&mut T{
    &mut self.elements[self.current]
  }
}
```

注意，Selector是一个结构体，但是解引用可以直接获得该结构体内的elements数组中的当前元素：

```Rust
let mut s = Selector {elements: vec!['x', 'y','z'] ,               current:2};

assert_eq!(*s, 'z');
assert!(s.is_alphabetic());
*s = 'w';
assert_eq!(s.elements, ['x', 'y', 'w']);
```

如果我们不能理解通过Deref实现的类型转换，我们就不能理解，为什么s明明是Selector类型，为什么就和 'z'字符串相等了呢？

# 日常编程中的例子一：String 和 &str

很多内置的类型都实现了Deref，比如String类型。

```Rust
fn main() {
    let a = "hello".to_string();
    let b = " world".to_string();

    let c = a + &b ;
    println!("{:?}", c);
}
```

注意，a 和 b都是String类型，当使用加号操作将字符串连接起来时，我们使用了&b，此时它应该是个&String类型，但是String类型实现的add方法的右值参数必须是&str类型。比如我们故意将&b其那面的&拿掉，报错信息如下：

```shell
   Compiling string v0.1.0 (/home/manu/CODE/Rust/CookBook/string)
error[E0308]: mismatched types
 --> src/main.rs:5:17
  |
5 |     let c = a + b ;
  |                 ^
  |                 |
  |                 expected &str, found struct `std::string::String`
  |                 help: consider borrowing here: `&b`
  |
  = note: expected type `&str`
             found type `std::string::String`

error: aborting due to previous error

For more information about this error, try `rustc --explain E0308`.
```

从上述报错信息中，可以看出，右值期待一个&str类型。可是我们的b是String类型。我们提供了&String类型，为什么也没有报错呢。

原因是String类型实现了Deref<Target=str>：

```Rust
impl ops::Deref for String{
	type Target = str ;
	fn deref(&self)->&str{
		unsafe{str::from_utf8_unchecked(&self.vec)
	}
}
```

正式因为String类型提供了这个，所以&String类型自动被隐式转化成了&str类型，所以，上面字符串的加法才得以正常的执行。

在此处，多啰嗦几句，来厘清String类型和str类型的区别。String的本质是Vec<u8>,	是分配在堆（heap）上的字节流，但是它保证它一定是一个有效的UTF-8序列。

&str本质是一个切片（slice），总是指向有效UTF-8序列的切片。注意，和其他切片一样，它是UnSized的，即不是固定大小的。和Trait object一样，需要胖指针。

String和&str的关系，就和Vec<T> 和 &[T]的关系是一样的。

# 日常编程中的例子二： Vec<T> 和 &[T]

Vec<T> 和 切片&[T]的关系也是类似的。我们在此处多介绍下切片的背景知识。

[T] 不指定长度，称之为切片，它是数组或Vector的一段，因为Slice最大的问题在于它长度不一定，不是（Sized），因此，不能直接存储在变量里面，或者作为参数传给函数。因此它总是通过引用传递：

```
since a slice can by any length , slices cannot be stored derectly in variables or passed as function parameter. Slices are always passed by reference
```

因为Vec实现了Deref，所有用在Slices上的函数，都可以用在Vec上：

```Rust
fn foo(s: &[i32]){
	println!("{:?}", s[0]);
}

fn main(){
	let v = vec![1,2,3];
	foo(&v);
}
```

因为Vec<T>实现了Deref<Target=[T]>，所以 &Vec<T>，会被自动转成&[T]，所以foo函数才不会报错。自动解引用避免了开发者手动转换，简化编程。

