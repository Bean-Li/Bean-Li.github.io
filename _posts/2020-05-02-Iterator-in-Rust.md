layout: post
title: Iterator in Rust (1)
date: 2020-05-01 10:29
categories: Rust
tag: Rust
excerpt:  Iterator in Rust (1)

# 前言

Iterators是迭代器，迭代器用来产生一串值。Rust提供了迭代器来遍历vectors，string，hashtables等，除此外：

* 产生文本中的每一行的内容

* 到达网络服务器的每个网络连接 (connection)

* 通过communication channel获得的来自其他线程的值

我们也可以创建自己的迭代器。Rust的for 循环可以使用迭代器，除此外，迭代器本身也提供很丰富的方法来使用迭代器。

迭代器是弹性灵活的，表达力很强，而且效率很高。我们以下面的函数为例：

```Rust
fn triangle(n: i32)-> i32{
    let mut sum = 0;
    for i in 1..n+1 {
        sum += i
    }
    sum
}
```

给定入参n ，计算， 1+2+ ...+n 的值。函数写的啰嗦。如果此处使用迭代器，因为迭代器有fold方法，可以用如下语句简洁地完成同样的功能：

```Rust
fn triangle_v2(n: i32) -> i32 {
    (1..n+1).fold(0, |sum, item| sum+item)
}
```

# Iterator and IntoIterator Traits

实现了std::iter::Iterator的任意数据结构，都可以称之为迭代器：

```Rust
trait Iterator {
	type Item ;
	fn next(&mut self) -> Option<self::Item> ;
}
```

Item是迭代器产生的数据类型。该Trait要实现一个next method来产生出Some(v):

* 要么产生出该迭代器的next value
* 要么返回None，表示迭代器已经迭代到尽头，无法产生新的value

如果某种数据结构，存在一种自然的迭代方法，这表明，该数据结构实现了std::iter::IntoIterator Trait，该trait中提供了into_iter方法处理此事，即：

* 输入是该数据结构本身
* 输出是该数据结构对应的迭代器

最典型最容易理解的是vector。

```Rust
fn main() {
    println!("There's:");
    let v = vec!["antimony", "arsenic", "aluminum", "tony"];

    for elem in &v {
        println!("{}", elem);
    }

    println!("===========================================");
    let mut iterator = (&v).into_iter();
    while let Some(element) = iterator.next(){
        println!("{}", element);
    }
    println!("Hello, world!");
}
```

输出如下：

```shell
There's:
antimony
arsenic
aluminum
tony
===========================================
antimony
arsenic
aluminum
tony
Hello, world!
```

for循环使用IntoIterator::into_inter 把 &v转成了一个迭代器，然后不断地执行 Iterator::next。当Iterator::next 返回Some(element)时，执行循环体的内容，如果Iterator::next 返回None，那么循环结束。

上面示范代码中，两种迭代的方式本质时一样的，上面方法for循环的本质就是下面的方法。

如果返回Iterator::next 返回None之后，继续调用next方法会发生什么？Iterator未定义这种行为，大部分迭代器只是会再次返回None，但也不是全部的迭代器都是如此。

# 创建迭代器

## iter 和 iter_mut 方法

大多数的collection类型提供了iter和iter_mut方法，可以返回迭代器。切片如&[T]或者&str 也有iter和iter_mut方法。这种方法比较通用和自然的方法产生迭代器。

```Rust
fn main() {
    let v = vec!["antimony", "arsenic", "aluminum", "tony"];
    let mut iterator = v.iter();
    assert_eq!(iterator.next(), Some(&"antimony"));
    assert_eq!(iterator.next(), Some(&"arsenic"));
    assert_eq!(iterator.next(), Some(&"aluminum"));
    assert_eq!(iterator.next(), Some(&"tony"));
}
```

std::path::Path 也实现了iter方法，每次产生一个路径部分。

```Rust
use std::ffi::OsStr ;
use std::path::Path ;

fn main() {
    let path = Path::new("/etc/ceph/ceph.conf");
    let mut iterator = path.iter();

    assert_eq!(iterator.next(), Some(OsStr::new("/")));
    assert_eq!(iterator.next(), Some(OsStr::new("etc")));
    assert_eq!(iterator.next(), Some(OsStr::new("ceph")));
    assert_eq!(iterator.next(), Some(OsStr::new("ceph.conf")));
}
```

## IntoIterator 实现

如果一个类型实现了IntoIterator，那么就可以通过调用 into_iter方法，转换成迭代器。

```Rust
use std::collections::BTreeSet ;

fn main() {
    let mut favorite = BTreeSet::new();
    favorite.insert("Rust Programming".to_string());
    favorite.insert("Hello World".to_string());
    favorite.insert("ZFS internal".to_string());

    let mut it = favorite.into_iter();
    assert_eq!(it.next(), Some("Hello World".to_string()));
    assert_eq!(it.next(), Some("Rust Programming".to_string()));
    assert_eq!(it.next(), Some("ZFS internal".to_string()));
    assert_eq!(it.next(), None);
}
```

上面的BTreeSet类型实现了IntoIterator trait，所以可以调用into_iter方法。BTreeSet中的元素，以升序迭代，因此，顺序是：

* Hello World
* Rust Programming
* ZFS internal

实际上，大部分的collections类型都实现了IntoIterator trait。

我们需要注意如下三种遍历方法：

```Rust
for element in &collection {..}
for element in &mut collection {..}
for element in collection {..}
```

第一种是共享引用，第二种是可变引用，第三种是会consume collection，即会获得元素的所有权。

但是我们也要小心，不是所有的collection都提供三种实现，比如HashSet和BTreeSet，不支持可变引用。

## drain 方法

很多collection提供了drain方法。这个方法对该类型使用可变应用 mutable reference，即执行完drain之后，原始的类型发生了变化。调用该方法后：

* 返回一个新的迭代器
* 原迭代器的内容发生了变化

```Rust
use std::iter::FromIterator;

fn main() {
    let mut outer = "Earth".to_string();
    let inner = String::from_iter(outer.drain(1..4));

    assert_eq!(outer, "Eh");
    assert_eq!(inner, "art");

    let mut v = vec![1,2,3];
    let u: Vec<_> = v.drain(1..).collect();
    assert_eq!(v , &[1]);
    assert_eq!(u, &[2,3]);

    v.drain(..);
    assert_eq!(v, &[]);
}
```

## 