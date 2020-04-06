---
layout: post
title: Random in Rust
date: 2020-04-01 10:29
categories: Rust
tag: Rust
excerpt: Random in Rust
---
# 前言 

本文介绍Rust中的随机数产生方法。Rust提供了rand crate来产生随机数。

# 产生标准类型的随机数

常见的类型：

* i8 
* i32
* i64
* u8
* u32
* u64
* bool
* float

如何产生这些类型的随机数：

## random方法

rand crate提供了一个快捷的方式产生随机数

```Rust
use rand::random;
```

这个函数可以产生出各种类型的随机数：

```Rust
extern crate rand ;
use rand::{random};

fn main() {

    let r_u8  :u8   = random();
    let r_u32 :u32  = random();
    let r_u64 :u64  = random();
    let r_f64 :f64  = random();

    println!("r_u8 {} r_u32 {} r_u64 {} r_f64 {}", r_u8, r_u32, r_u64, r_f64);

    let r_i8  = random::<i8>();
    let r_i32 = random::<i32>();
    let r_i64 = random::<i64>();
    let r_f32 = random::<f32>();

    println!("r_i8 {} r_i32 {} r_i64 {} r_f32 {}", r_i8, r_i32, r_i64, r_f32);

    if random()
    {
        println!("This print sometimes appears ,sometimes cannot be seen");
    }
}
```

输出结果如下：

```shell
manu-Inspiron-5748 CookBook/random ‹master*› » cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.01s
     Running `target/debug/random`
r_u8 108 r_u32 2741032365 r_u64 1387371133752336062 r_f64 0.46364384148921434
r_i8 -27 r_i32 -100204817 r_i64 -5883903312845635387 r_f32 0.24733853

manu-Inspiron-5748 CookBook/random ‹master*› » cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.01s
     Running `target/debug/random`
r_u8 13 r_u32 2994006999 r_u64 10822813997525460261 r_f64 0.8190726733233058
r_i8 99 r_i32 -674518030 r_i64 8391467083991742077 r_f32 0.6915216
This print sometimes appears ,sometimes cannot be seen
```

我们可以看到：

* 每次执行，结果各不相同，即达到随机的效果。
* random()函数可以产生出各种类型的随机值，包括bool型
* random<f32>和random<f64>()产生出[0,1)范围内的浮点数

##  使用thread_rng函数

rand这个crate实现了Rng这个trait，这个trait的Rng的意思为 Random Number Generator，即随机数字生成器。该trait有如下重大的API：

* gen
* gen_range
* sample
* same_iter

我们可以利用gen函数来产生出我们需要的随机数。

```Rust
extern crate rand ;
use rand::{Rng};

fn main() {

    let mut rng = rand::thread_rng();
    let r_u8  :u8   = rng.gen();
    let r_u32 :u32  = rng.gen();
    let r_u64 :u64  = rng.gen();
    let r_f64 :f64  = rng.gen();

    println!("r_u8 {} r_u32 {} r_u64 {} r_f64 {}", r_u8, r_u32, r_u64, r_f64);

    let r_i8  = rng.gen::<i8>();
    let r_i32 = rng.gen::<i32>();
    let r_i64 = rng.gen::<i64>();
    let r_f32 = rng.gen::<f32>();

    println!("r_i8 {} r_i32 {} r_i64 {} r_f32 {}", r_i8, r_i32, r_i64, r_f32);

    if rng.gen()
    {
        println!("This print sometimes appears ,sometimes cannot be seen");
    }
}
```

我们可以看到，和之前的random函数一样，通过Rng的gen函数，一样可以获取各种类型的随机数：

```SHELL
输出如下：
r_u8 80 r_u32 1687585909 r_u64 8352424927451894494 r_f64 0.8696927377548344
r_i8 -57 r_i32 -1884445722 r_i64 -4327157282855795895 r_f32 0.074106276

再跑一轮
r_u8 98 r_u32 2598551176 r_u64 16337455607562742807 r_f64 0.997604506929101
r_i8 114 r_i32 1535487133 r_i64 39920825847438157 r_f32 0.9148346
This print sometimes appears ,sometimes cannot be seen
```

Rng的gen函数不仅仅是可以产生出标准类型，还可以产生出tuple

```
let tuple: (u8, i32, char) = rng.gen(); 
```

如果在上面的代码最后加入如下两行：

```Rust
    let tuple :(u8, i32, f64, bool) = rng.gen();
    println!("{:?}", tuple);
```

会多输入如下内容：

```SHELL
(242, -1061332895, 0.3170911846965502, false)
```

不仅可以产生tuple，数组也是一样：

```Rust
    let array1 :[f32; 7] = rng.gen();
    println!("{:?}", array1)
```

上面代码产生7个f32类型的数组，数组中的数字是随机f32:

```SHELL
[0.07171655, 0.5189788, 0.68672246, 0.7315745, 0.1870473, 0.36728072, 0.66637975]
```

# 产生某范围的数字

上面章节是产生某类型的随机数字，有些时候需要产生某个范围内的某类型的随机数字。Rng提供了gen_range函数，负责此事。

## 使用gen_range

Rng的gen_range函数，可以获取某个范围内的随机数字，该函数接受两个参数，（low，high），其中low包含，high不包含, including low , excluding high。

```Rust
extern crate rand ;
use rand::{Rng};

fn main() {
    let mut rng = rand::thread_rng();
    let num = rng.gen_range(0,10);
    let f_num = rng.gen_range(0.0,10.0);

    println!("num {} f_num {}", num, f_num);
}
```

输出结果如下：(多轮次结果)

```shell

num 7 f_num 2.52105525700977

num 9 f_num 6.242366880456077

num 9 f_num 7.333549347293831

num 7 f_num 5.24803166200549
```

如果需要反复地获取某个范围内的随机数字，处于效率的考量，我们最好采用下面方法：

## 使用Uniform

struct Uniform可以获取均匀分布的数值，效率要比反复调用gen_range高一些：

```Rust
extern crate rand ;
use rand::distributions::{Distribution, Uniform};

fn main() {
    let mut rng = rand::thread_rng();
    let uniform = Uniform::from(1..7);

    loop {
        let throw = uniform.sample(&mut rng);
        println!("Roll the die : {}", throw);

        if throw == 6
        {
            break;
        }
    }
}
```

输出结果如下：

```shell
manu-Inspiron-5748 CookBook/random ‹master*› » cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.01s
     Running `target/debug/random`
Roll the die : 1
Roll the die : 3
Roll the die : 6
manu-Inspiron-5748 CookBook/random ‹master*› » cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.01s
     Running `target/debug/random`
Roll the die : 4
Roll the die : 5
Roll the die : 1
Roll the die : 3
Roll the die : 6
```

Uniform表示均匀分布，我们使用了from函数，除此外，Uniform还提供了new和new_inclusive两个方法：

```Rust
pub fn new<B1, B2>(low: B1, high: B2) -> Uniform<X>
pub fn new_inclusive<B1, B2>(low: B1, high: B2) -> Uniform<X>
```

接受两个参数，low和high，区别在于是否包含high。

* new ：[low, high)
* new_inclusive ： [low, high]

我们可以用这两个方法，改造下我们的代码：

```Rust
extern crate rand ;
use rand::distributions::{Distribution, Uniform};

fn main() {
    let mut rng = rand::thread_rng();
    let uniform = Uniform::new_inclusive(1,6);

    loop {
        let throw = uniform.sample(&mut rng);
        println!("Roll the die : {}", throw);

        if throw == 6
        {
            break;
        }
    }
}
```

输入结果如下：

```
manu-Inspiron-5748 CookBook/random ‹master*› » cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.01s
     Running `target/debug/random`
Roll the die : 4
Roll the die : 2
Roll the die : 3
Roll the die : 5
Roll the die : 2
Roll the die : 4
Roll the die : 3
Roll the die : 3
Roll the die : 2
Roll the die : 5
Roll the die : 5
Roll the die : 2
Roll the die : 6
```

# 关于 distribution

既然介绍到了随机，就不得不讲概率分布。概率分布是指，采样空间内的每一个值出现的概率。这个世界上并不是只有均匀分布（Uniform）一种，比如常见的正态分布。	

## 标准分布：distributions::Standard

这个分布比较重要，实际上，Rng::gen就使用这个分布，这是产生很多类型随机数的默认分布：

* integers ： 均匀分布
* char： 在整个Unicode内均匀分布
* bool： 产生true or false，概率均为0.5
* float：均匀分布在[0,1)范围内

Alphanumeric是一种特殊的Standard，它是在[a-zA-Z0-9]这个字符集上均匀分布。

```Rust
extern crate rand ;
use rand::{Rng, thread_rng};
use rand::distributions::{Alphanumeric};

fn main() {
    let rand_string :String = thread_rng()
        .sample_iter(&Alphanumeric)
        .take(12)
        .collect();
    println!("rand_string : {}", rand_string)
}
```

产生出长度是12的随机字符：

```
rand_string : IZFJGQoCJFWc
rand_string : WYTW38x7lJ64
```

上面代码片段中，给出了sample_iter 。这是一个迭代器，如果我们想产生指定长度或者指定个数的随机数，我们可以使用这种方式：

## 均匀分布：Uniform

这种分布前面介绍很多了。我们需要记住，Standard分布和Uniform在很多场景下是一样的，而Standard分布并不是正态分布，切莫混淆即可。

##  伯努利分布 ：Bernoulli

我们通过扔硬币，正反面出现的概率是相等的，即true or false 都是一半的概率。某些情况下，返回true和false的概率并不相等，比如我们希望1/3的概率返回True，2/3的概率返回true，rand提供了gen_bool函数：

```Rust
extern crate rand ;
use rand::{Rng, thread_rng};

fn main() {
    let mut rng = thread_rng();

    for _ in 0..10{
        println!("{}", rng.gen_bool(1.0/4.0));
    }
}
```

输出结果如下：

```Shell
manu-Inspiron-5748 CookBook/random ‹master*› » cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.01s
     Running `target/debug/random`
true
false
false
false
false
false
true
false
false
true
```

gen_bool函数，接受一个f64类型的参数：

```Rust
fn gen_bool(&mut self, p: f64) -> bool
```

p的含义是，返回true的概率。该值的取值范围为[0,1]。

* 当p = 0，表示永远返回false，
* 当p = 1， 表示永远返回true
* （0，1）， p表示返回true的概率



