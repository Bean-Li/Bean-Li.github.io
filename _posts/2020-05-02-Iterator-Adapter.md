---
layout: post
title: 迭代器适配器(Iterator Adapter)
date: 2020-05-02 10:29
categories: Rust
tag: Rust
excerpt:  Iterator Adapter
---
# 前言

本文介绍Iterator Adapters，即迭代器适配器。Iterator Adapter消耗一个迭代元素转换成另一个迭代元素。

迭代器就好像是现实世界的水管，现实世界中，水管是标准化的接口，打开水管就可以出水。厨房用水可能需要加热，洗澡间用水可能需要冷热混合，淋浴花洒可能需要将水分割成若干个小水流。干这些事情的，我们称之为适配器。

# map and filter

map很简单，将方法映射到迭代器产生的每一个元素，从而输出新的迭代元素。

```Rust
fn main() {
    let text = " Panda \n    giraffes\niguanas \nsquid".to_string();
    let v:Vec<&str> = text.lines()
        .map(str::trim)
        .collect() ;

    assert_eq!(v, ["Panda", "giraffes", "iguanas", "squid"]);
}
```

 String和&str提供了lines方法，该方法也会产生一个迭代器，这是上一篇产生迭代器方法中没有提到的。类似的方法有很多：

- s.bytes()
- s.chars()
- s.split_whitespace()
- s.lines()
- s.split('/')
- s.matches(char::is_numeric)

所以text.lines 方法返回的是一个迭代器，即以回车字符为界，每次迭代产生出一行的内容。

map将每一行的内容，执行str::trim方法，即将每一行头部和尾部的空白字符删除。

collect方法将迭代器转换成Vec，从而完成将一段文本，按行生成一个Vec，同时去除头部和尾部的无用空白字符。

map很容易理解，即对每一个元素都执行对应的方法，得到新的迭代元素，从而得到新的迭代器。filter，顾名思义，即过滤，即对每个迭代元素执行对应方法，得到bool型的结果，只有bool值为true的迭代元素才会保留下来，bool值为负的元素被滤出掉，从而形成新的迭代器。

```Rust
fn main() {
    let text = " Panda \n    giraffes\niguanas \nsquid".to_string();
    let v:Vec<&str> = text.lines()
        .map(str::trim)
        .filter(|s| *s != "iguanas")
        .collect() ;

    assert_eq!(v, ["Panda", "giraffes", "squid"]);
}
```

如上示范代码，滤除等于"iguanas"的迭代元素。

这两个适配器的方法签名如下：

```Rust
fn map<B, F>(self, f: F)->some Iterator<Item=B>
	 where Self: Sized, F: FnMut(Self::Item) -> B
	 
fn filter<P> (self, predicate: P) -> some Iterator<Item=Self::Item>
	where Self: Sized, P: Fnmut(&Self::Item)-> bool
```

我们细细品map和filter，可以得到如下结论：

* map 是值传递，而filter是共享引用传递。
* map之后新的迭代器的元素和原始迭代器产生的元素，未必是同一种类型
* filter之后，新的迭代器的元素和原始迭代器的元素是同一种类型

filter是共享引用，这也是为什么我们示例代码中，为 *s != "iguanas"的原因。

最后需要指出的是，迭代器适配器是惰性的，要有真正的消费行为，才会产生迭代，否则就只是个迭代器而已，无人调用。回到我们的范例，最后的collect方法调用，才真正产生了消费行为，才会不断调用迭代器的next方法。

我们给个反面的示范：

```Rust
["earth", "water", "air", "fire"].iter()
	.map(|elt| println!("{}", elt));
```

上面方法看起来很好，但是打印每一个元素的值，但是实际上，map适配器是产生出新的迭代器，但是没有任何方法要求迭代器产生出新的元素，因此，Rust会发出警告。

# enumerate 

enumerate 适配器是一个看起来相当无聊，但在实际编程中，非常常用的适配器。我相信写过python代码的人，读到这一句，都会会心一笑，赞同我所言非虚。

比如一个迭代器产生出 A ， B ， C这种元素，那么经过enumerate适配器之后，新的迭代器产生出(0,A),(1,B), (2,C) 这种迭代元素。

```Rust
fn main() {
    let a = ['a', 'b', 'c'] ;
    let iter = a.iter().enumerate();

    for (idx, elem) in iter {
        println!("({}, {})", idx, elem)
    }
}
```

输出结果如下：

```Rust
(0, a)
(1, b)
(2, c)
```

# zip

zip适配器是将两个迭代器的元素，捏合成一个迭代器。比如第一个迭代器的迭代元素是"a", "b", "c"，第二个迭代器的迭代元素'A', 'B', 'C'，那么iter1.zip(iter2)，产生的新的迭代器的元素为('a', 'A'), ('b', 'B'),('c', 'C')。

从上面的讨论可以看出，enumerate适配器，本质是zip适配器的一种特例。

```Rust
(0..).zip(iter) 
iter.enumerate()
```

#  chain

zip是将两个迭代器配对成一个新的迭代器，新迭代器中每个元素是两个迭代器同一位置的元素组成的元组。

chain 适配器是将两个迭代器连接在一起，即append的意思。

以 iter1.chain(iter2)为例，返回一个新的迭代器，新的迭代器的next方法，会先返回iter1迭代器中的元素，当iter1中元素耗尽，那么开始从iter2 迭代器中获取元素，知道iter2中的元素也耗尽。简单地说，就是把iter2 放到iter1的后面，连成一个新的迭代器。

```Rust
fn main() {
    let a1 = [1, 2, 3] ;
    let a2 = [4, 5, 6] ;
    let iter = a1.iter().chain(a2.iter());
    for elem in iter {
        println!("{}", elem)
    }
}
```

上面的做法是严格的做法，实际上，

```Rust
fn chain<U>(self, other: U) -> Chain<Self, <U as IntoIterator>::IntoIter>
where  U: IntoIterator<Item = Self::Item>,
```

chain 适配器使用IntoIterator，我们可以传给它任何可以转换成Iterator的变量，比如Vec本身，即上面的示范，也可以写成：

```Rust
let iter = a1.iter().chain(a2)
```

因为slice(&[T]) 实现了IntoIterator，所以可以直接传给chain作为参数。

# filter_map 

filter_map 适配器是个非常有用的适配器，它有点是filter 和map的结合体的意味。我们有时候遍历Result类型的迭代器的时候，有时候，可能会有失败的情况，我们就用filter_map来忽略失败的项。

```Rust
fn main() {
    let strings = vec!["tofu", "93", "18"];
    let numbers: Vec<_> = strings
        .into_iter()
        .map(|s| s.parse::<i32>())
        .filter_map(Result::ok)
        .collect();
    println!("Results: {:?}", numbers);
}
```

我们故意用了一个比较啰嗦的版本，我们来细细地分析：

1. map(|s| s.parse::<i32>())返回的是类型是Result类型
2. Result::ok()方法，如果result是成功的，那么返回Some(success_value)，如果执行失败，返回None
3. filter_map适配器会对迭代器的每一个元素调用闭包，如果闭包返回Some(element)，那么element元素返回，如果闭包返回None，那么它会忽略该元素，尝试对下一个迭代器元素调用闭包。

所以我们总结下filter_map的实质：

1. filter_map适配器，会对迭代器中的每一元素执行闭包，体现了map的方面
2. 执行闭包之后，正常的结果元素，保留，作为结果迭代器的元素，如果闭包的返回值是None，忽略，体现filter的方面。

上面的实例代码，我们可以进一步简化：

```Rust
fn main() {
    let strings = vec!["tofu", "93", "18"];
    let numbers: Vec<_> = strings
        .into_iter()
        .filter_map(|s| s.parse::<i32>().ok())
        .collect();
    println!("Results: {:?}", numbers);
}
```

# flat_map 

flat是平坦的意思，flatten是拍平。有些时候，可能需要将多个序列的结果级联起来，变成一个序列。

比如说我们有个数据结构称为主要城市：

* 日本Japan的主要城市有Tokyo， Kyoto
* 中国China的主要城市有BeiJing，ShangHai，NanJing
* 美国USA的主要城市有Portland，Washington

如果我们需要得到一个中国 日本 美国的城市列表，我们就需要打破各自序列的界限，变成一个列表。

```Rust
use std::collections::HashMap ;

fn main() {
    let mut major_cities = HashMap::new() ;

    major_cities.insert("China", vec!["Beijing", "Shanghai", "Nanjing"]);
    major_cities.insert("Japan", vec!["Tokyo", "Kyoto"]);
    major_cities.insert("USA", vec!["Portland", "New York"]);

    let counties = ["China", "Japan", "USA"];

    for &city in counties.iter().flat_map(|country| &major_cities[country]) {
        println!("{}", city);
    }
}
```

输出如下：

```
Beijing
Shanghai
Nanjing
Tokyo
Kyoto
Portland
New York
```

我们细细的解析flat_map

* 首先是map，map之后，每个元素都是一个迭代器
* 然后将map之后的多个迭代器的界限打破，平坦化，把多个迭代器变成一个迭代器。

另一个类似的例子如下：

```Rust
fn main() {
    let words = ["alpha", "beta", "gamma"];

    // chars() returns an iterator
    let merged: String = words.iter()
        .flat_map(|s| s.chars())
        .collect();
    assert_eq!(merged, "alphabetagamma");
}
```

# take和take_while

take适配器用来截取迭代器的前n个元素。

```Rust
fn main() {
    let a = [1, 2, 3];
    let mut iter = a.iter().take(2);

    assert_eq!(iter.next(), Some(&1));
    assert_eq!(iter.next(), Some(&2));
    assert_eq!(iter.next(), None);
}
```

这个take没啥特别的，理解了take，可以接下来学习take_while适配器了。

take_while适配器和take一样，截取原始迭代器的部分元素，但是结束条件不一样：

* take(n)：很明确，即选取前n个元素，余者不取
* take_while : 会有闭包函数，闭包函数返回bool型结果，如果bool型结果为false，即立刻停止。

```Rust
fn main() {
    let message = "To: jimb\r\n\
                   From: superego <editor@oreilly.com>\r\n\
                   \r\n\
                   Did you get the any writing done today \r\n\
                   when will you stop wasting time plotting fractals\r\n";
    for header in message.lines().take_while(|l| !l.is_empty()){
        println!("{}", header)
    }
}
```

message是一封邮件的内容，header和body之间，有一空行。解析程序通过判断空行位置，来打印header信息：

```
To: jimb
From: superego <editor@oreilly.com>
```

