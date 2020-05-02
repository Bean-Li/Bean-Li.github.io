---
layout: post
title: 迭代器消费器(Iterator Consumer)
date: 2020-05-02 10:29
categories: Rust
tag: Rust
excerpt:  Iterator Consumer
---
# 前言

前面提到，Rust中的迭代器都是惰性的，也就是说，无人触发的情况下，不会自己遍历。最直接消费迭代器数据的方法就是for循环。for循环会隐式地调用迭代器的next方法，从而达到循环的目的。

为了方便编程，Rust也提供了for循环之外的用于消费迭代器内数据的方法，他们被称为消费器 Consumer。

# any

any用来遍历迭代器，查找是否存在满足条件的元素。any只要找到了满足条件的元素，就停止遍历， 即不会继续Consume迭代的元素。

```Rust
fn main() {
    let id = "Iterator" ;
    assert_eq!(id.chars().any(char::is_uppercase), true);
    assert_eq!(id.chars().any(char::is_uppercase), false);
}
```

# all

all用来判定是否迭代器的所有元素都满足某个条件。当存在一个元素不满足条件，就不必继续下去了，因为一定会返回false。

空的迭代器总是返回true。

```Rust
fn main() {
    let a = [1, 2, 3];
    let mut iter = a.iter();
    assert!(!iter.all(|&x| x != 2));
    assert_eq!(iter.next(), Some(&3));
}
```

# fold

fold我喜欢翻译成折叠器，即遍历迭代器的所有元素，通过执行执行的运算，最终折叠成一个元素。比如，将迭代器所有元素累加在一起，累乘在一起。

```Rust
fn main() {
    let a = [1, 2, 3,4];

    println!("the count of element is {}", a.iter().fold(0, |n, _| n+1));
    println!("the sum of element is {}", a.iter().fold(0, |n, e| n+e));
    println!("the multiple of element is {}", a.iter().fold(1, |n, e| n*e));
}
```

运行结果为：

```
the count of element is 4
the sum of element is 10
the multiple of element is 24
```

依次对迭代器的所有元素执行某种运算，很明显，需要一个初始值。

* 对于累加而言，0是合理的初始值
* 对于累乘而言，1是合理的初始值

folder方法的签名如下：

```
fn fold<B, F>(self, init: B, f: F) -> B
where
    F: FnMut(B, Self::Item) -> B, 
```

# count，sum和product

fold是个通用的方法，对于上面提到的常用的fold方法，Rust有提供了专门的方法：

| 方法    | 作用                                                  |
| ------- | ----------------------------------------------------- |
| count   | 迭代器中元素的个数                                    |
| sum     | 迭代器中所有元素之和                                  |
| product | 迭代器中所有元素的乘积                                |
| max     | 迭代器中最大的元素（Iterator必须实现了std::cmp::Ord） |
| min     | 迭代器中最小的元素（Iterator必须实现了std::cmp::Ord） |

```Rust
fn main() {
    let a = [1, 2, 3,4];

    println!("the count of element is {}", a.iter().fold(0, |n, _| n+1));
    println!("the sum of element is {}", a.iter().fold(0, |n, e| n+e));
    println!("the multiple of element is {}", a.iter().fold(1, |n, e| n*e));

    println!("the count of element is {}", a.iter().count());
    println!("the sum of element is {}", a.iter().sum::<u32>());
    println!("the multiple of element is {}", a.iter().product::<u32>());
}
```

其中max和min，要必须是实现了std::cmp::Ord的类型才能调用。比如f32和f64类型是不行的，因为浮点数中有NaN。

# collect

collect消费器应该算是使用最广泛的一种了。collect有搜集的意思。它将迭代器通过next方法获得的元素，"搜集"起来，收集到指定的集合容器中。

比如我们很常见命令行获取入参会有如下的代码：

```Rust
let args: Vec<String> = std::env::args().collect() ;
```

std::env::args()是一个入参的迭代器，我们通过collect方法，将迭代器转成vector。

因为collect转成vector是最常见的一种用法，很容易让人产生误解，即collect是用来完成iterator到vector转换的。这种理解是不对的。只要我们愿意，我们可以将迭代器转成各种不同类型的collection。

```
let args: HashSet<String> = std::env::args().collect();
let args: BTreeSet<String> = std::env::args().collect();
let args: LinkedList<String> = std::env::args().collect();

let args: HashMap<String, usize> = std::env::args().zip(0..).collect();
```

看到这个地方，可能会很奇怪，为什么HashSet /BTreeSet/LinkedList等可以从迭代器转化而来？因为这些类型都实现了std::iter::FromIterator trait，它会调用该trait的from_iter方法