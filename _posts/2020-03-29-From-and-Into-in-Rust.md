
---
layout: post
title: From and Into trait in Rust
date: 2020-03-29 10:29
categories: Rust
tag: Rust
excerpt: From and Into Trait in Rust
---
# 前言

From 和 Into是Rust内很重要的两个Trait，这两个trait的作用转化：

```English
consume a value of one type , and return a value of another
```

和AsRef和Borrow都有不同，AsRef和Borrow，是借用reference，从一个类型转成另一个类型，两种类型要想能实现AsRef和Borrow，那么这两种类型要有某种一目了然或者天然的联系。比如String 和 str， String 和Path。

但是From 和Into不同，要比AsRef和Borrow来的要重一些

* take the ownership of the argument 
* transform it
* return ownership of the result 

这两个trait的定义如下：

```
trait  Into<T>:Sized {
	fn into(self)->T ;
}

trait From<T>: Sized{
	fn from(T) -> Self;
}
```

定义很对称。

# From

From trait的意图是，允许一个类型定义"怎么根据另一个类型生成自己"。它提供了一种类型转换的机制，最典型的就是str和String这对冤家。

```Rust
let my_str = "Hello World!" ;
let my_string  = String::from(my_str);
```

String类型实现了From<&str> 。

Programming Rust 给出了另一个例子，IPV4的地址，一般是4个[0,255]范围内的数字，换言之，我们可以用[u8;4] 来表示一个IPV4的地址。下面我们看下IPV4相关的数据结构：

```Rust
pub enum IpAddr {
    V4(Ipv4Addr),
    V6(Ipv6Addr),
}
```

Ipv4Addr 结构体定义在 std::net::Ipv4Addr :

```Rust
use std::net::Ipv4Addr;

let localhost = Ipv4Addr::new(127, 0, 0, 1);
```

因为Ipv4Addr很明显可用 [u8;4] 或者干脆一个u32表示，因此，Ipv4Addr实现了如下接口

```Rust
impl From<[u8; 4]> for Ipv4Addr
impl From<u32> for Ipv4Addr
```

因此我们可以写出如下代码：

```Rust
use std::net::Ipv4Addr;

let addr1 = Ipv4Addr::from([66,146,219,98]);
let addr2 = Ipv4Addr::from(0xd0766b94_u32);
```

# Into

Into 和From的方向正好相反，对称。它表明，当前变量要转换到类型。大多数时候编译器并不能推断出，你要转换的类型，所以一般需要显式地指出，目标类型为何。

回到上面的Ipv4Addr的例子，我们可以从一个u32 转换成Ipv4Addr：

Programming Rust 给出了如下的例子：

```
use std::net::Ipv4Addr ;
fn ping<A> (address: A)-> std::io::Result<bool>
	where A: Into(Ipv4Addr)
{
	let ipv4_addr = address.into();
	...
}
```

注意，只要address 的类型，实现了Into(Ipv4Addr)，就可以，不一定非要接受Ipv4Addr类型的。比如[u8;4] 比如 u32，都可以作为ping的参数。

注意，对于两个类型A和B而言，如果A通过B得到，即from(B)，那么必然B可以 into A。当我们定义我们自己的类型的时候，如果我们实现了From trait， 那么Into，我们就自动实现了。

```Rust
use std::converter::From;

#[derive(Debug)]
struct Number {
  value :i32,
}

impl From<i32> for Number {
  fn from(item: i32) -> Self {
    Number {value: item}
  }
}
fn main() {
  let int = 5;
  let num: Number = int.into();
  println!("My number is {:?}", num);
}
```

# 讨论

我们开始讨论之前，我们先看一个例子：

```Rust
let text = "Beautiful Soup".to_string();
let bytes: Vec<u8> = text.into();
```

String类型实现了Into<Vec<u8>>，String的空间是在heap上的，这些空间完全可以改头换面，变成bytes所有，即不需要重新分配空间，也不需要copy String中的内容到目的类型，

但是并不是所有的转换都是如果美好的，From也好，Into也好，都没有承诺轻量级。AsRef 和AsMut承诺了。

举例子讲，String类型实现了From<&str>，但是为了做到From<&str>，程序不得不分配空间，拷贝string 切片到分配好的heap空间中，这些操作并不轻量。

我们考虑其他数据结构之间的转换，来进一步强调这个观点。二项堆BinarayHeap<T>是一个特殊的用于排序的数据结构，完全可以从Vec<T> 转换而成。如果要想完成From<Vec<T>> 

* 分配空间
* 比较Vec中的各个元素
* 重新组织元素的位置

等这些昂贵的操作就必不可少。因此，切莫觉得From和Into一定是轻量级的操作。

另一个需要注意的点是，From和Into这种类型转换，用在永不失败的场景。因为方法的返回值没有失败了的情况下的相关信息，因此，From和Into，基本是用于不会失败的。如果确实可能失败，最好定义一个方法，返回类型为Result类型。