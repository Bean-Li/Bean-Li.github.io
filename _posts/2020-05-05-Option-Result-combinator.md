---
layout: post
title: Option和Result相关的组合算子
date: 2020-05-05 20:00
categories: Rust
tag: Rust
excerpt:  Option and Result
---
# 前言

上一篇文章，从宏观的角度介绍了Rust错误处理的方法。提到了Option，作为弱化版的Result。

```Rust
enum Option<T>{
	None,
	Some(T),
}
```

Option作为Rust的的系统类型，用来表示值不存在的可能。在编程中，这是一个非常好的做法，因为它强制Rust检测和处理值不存在的情况。

Option和Result，Rust都提供了一些组合算子来，来简化代码的书写，提供更强大的表达力。

# Option相关的组合算子

## ok_or和ok_or_else

这两个是一组，作用都是从Option转成Result。

```Rust
pub fn ok_or<E>(self, err: E) -> Result<T, E>

pub fn ok_or_else<E, F>(self, err: F) -> Result<T, E>
where
    F: FnOnce() -> E, 
```

其作用细细来讲，如下：

* Some(v)  --> Ok(v)
* None  --> Err(err)

其行为大概如下所示：

```Rust
fn ok_or<T, E>(option: Option<T>, err: E) -> Result<T, E> {
    match option {
        Some(val) => Ok(val),
        None => Err(err),
    }
}
```

下面以一个例子来展示这个组合算子的作用

```Rust
use std::env;

fn double_arg(mut argv: env::Args) -> Result<i32, String> {
    argv.nth(1)
        .ok_or("Please give at least one argument".to_owned())
        .and_then(|arg| arg.parse::<i32>().map_err(|err| err.to_string()))
}

fn main() {
    match double_arg(env::args()) {
        Ok(n) => println!("{}", n),
        Err(err) => println!("Error: {}", err),
    }
}
```

如果执行，不给任何参数：

```
Error: Please give at least one argument
```

给一个参数为5，输出如下：

```
5
```

nth(1) 因为没有第一个参数（0-index），所以Option的值为None，ok_or将Option转成了Result，最终返回的Err(err)。

## map 

这个组合算子的作用是将Option<T> 转换成Option<U>。

* Some(T) ->Some(U)
* None -> None

```Rust
fn main() {
    let maybe_some_string = Some(String::from("Hello, World!"));
    let maybe_some_len = maybe_some_string.map(|s| s.len());

    assert_eq!(maybe_some_len, Some(13));
}
```

注意map是值传递，会消耗掉原始的值。

## and_then

and_then 又被称为flatmap，它的作用是从一个Option转成另一Option：

* None -> None
* Some(x) -> Some(f(x))

对于Some而言，and_then把包裹的值作为参数

```Rust
fn sq(x: u32) -> Option<u32> { Some(x * x) }
fn nope(_: u32) -> Option<u32> { None }

assert_eq!(Some(2).and_then(sq).and_then(sq), Some(16));
assert_eq!(Some(2).and_then(sq).and_then(nope), None);
assert_eq!(Some(2).and_then(nope).and_then(sq), None);
assert_eq!(None.and_then(sq).and_then(sq), None);
```

注意sq函数的入参，是u32型，并不是Some()。

# Result相关的组合算子

## is_ok and is_err

result.is_ok和result.is_err返回bool型的结果。

对于is_ok而言：

* Ok(T) 返回true
* Err(E) 返回false

对于is_err而言：

* Ok(T)返回false
* Err(E)返回true

```Rust
let x: Result<i32, &str> = Err("Some error message");
assert_eq!(x.is_err(), true);

let x: Result<i32, &str> = Ok(-3);
assert_eq!(x.is_ok(), true);
```

## ok and err

```Rust
pub fn ok(self) -> Option<T>
pub fn err(self) -> Option<E>
```

注意，这两个算子都是将Result转成Option类型，一个关注成功的情况，一个关注出错的情况。

result.ok() 如果Result返回Ok(T)，那么返回值是Option<T>，即Some(success_value)。如果Result失败，那么该函数返回None，把Err的值丢弃掉。

```Rust
let x: Result<u32, &str> = Ok(2);
assert_eq!(x.ok(), Some(2));

let x: Result<u32, &str> = Err("Nothing here");
assert_eq!(x.ok(), None);
```

result.err()如果Result失败，返回Err(E)，那么返回返回Option<E>。

```Rust
let x: Result<u32, &str> = Ok(2);
assert_eq!(x.err(), None);

let x: Result<u32, &str> = Err("Nothing here");
assert_eq!(x.err(), Some("Nothing here"));
```

## and_then

Option也有and_then 算子，Result和Option一样的，语义是一样的。而且有很多这样的算子。

```Rust
use std::env;

fn double_arg(mut argv: env::Args) -> Result<i32, String> {
    argv.nth(1)
        .ok_or("Please give at least one argument".to_owned())
        .and_then(|arg| arg.parse::<i32>().map_err(|err| err.to_string()))
}

fn main() {
    match double_arg(env::args()) {
        Ok(n) => println!("{}", n),
        Err(err) => println!("Error: {}", err),
    }
}
```

还是前面的例子， ok_or()之后，Option就转成了Result，如果Ok的情况下，类型是字符串，那么and_then 会尝试将字符串解析成i32。

```Rust
pub fn and_then<U, F>(self, op: F) -> Result<U, E>
where
    F: FnOnce(T) -> Result<U, E>, 
```

如果Result的值是Ok(T)，那么执行op(T)，否则直接返回Err。

```Rust
n sq(x: u32) -> Result<u32, u32> { Ok(x * x) }
fn err(x: u32) -> Result<u32, u32> { Err(x) }

assert_eq!(Ok(2).and_then(sq).and_then(sq), Ok(16));
assert_eq!(Ok(2).and_then(sq).and_then(err), Err(4));
assert_eq!(Ok(2).and_then(err).and_then(sq), Err(2));
assert_eq!(Err(3).and_then(sq).and_then(sq), Err(3));
```

## map and map_err

上面的例子中给出了map_err的使用。map和map_err是对Result中的Ok(T)或者Err(E)执行某个函数或者执行某种变换，得到新的Result：

```
pub fn map<U, F>(self, op: F) -> Result<U, E>
where
    F: FnOnce(T) -> U, 
    
pub fn map_err<F, O>(self, op: O) -> Result<T, F>
where
    O: FnOnce(E) -> F, 
```

对于map函数而言：

* 如果Result值是Ok(T)，对T执行函数
* 如果Result的值是Err(E)，保持不变，新的结果Option的值也是Err(E)

如下实例代码：

```Rust
let line = "1\n2\n3\n4\n";

for num in line.lines() {
    match num.parse::<i32>().map(|i| i * 2) {
        Ok(n) => println!("{}", n),
        Err(..) => {}
    }
}
```

对于map_err而言：

* 如果Result的值是Err(E)，对E执行函数
* 如果Result的值是Ok(T)，保持不变。结果Result和输入Result一致。

```Rust
fn stringify(x: u32) -> String { format!("error code: {}", x) }

let x: Result<u32, u32> = Ok(2);
assert_eq!(x.map_err(stringify), Ok(2));

let x: Result<u32, u32> = Err(13);
assert_eq!(x.map_err(stringify), Err("error code: 13".to_string()));
```

map_err可以用来对错误进行转换。

# 代码赏析

最后的最后，赏析一段Error Handle的代码

```Rust
use std::fs::File;
use std::io::Read;
use std::path::Path;

fn file_double<P: AsRef<Path>>(file_path: P) -> Result<i32, String> {
    File::open(file_path)
        .map_err(|err| err.to_string())
        .and_then(|mut file| {
            let mut contents = String::new();
            file.read_to_string(&mut contents)
                .map_err(|err| err.to_string())
                .map(|_| contents)
        })
    .and_then(|contents| {
        contents.trim().parse::<i32>()
            .map_err(|err| err.to_string())
    })
    .map(|n| 2 * n)
}

fn main() {
    match file_double("foobar") {
        Ok(n) => println!("{}", n),
        Err(err) => println!("Error: {}", err),
    }
}
```

上面这段代码中，有很多种不同的错误类型：

* io::Error
* num::ParseIntError

通过map_err都转成了Result<i32,String>类型。

上述写法看起来技巧性很强，但是我不喜欢，属于炫技派做法，心智负担比较重。

```Rust
use std::fs::File;
use std::io::Read;
use std::path::Path;

fn file_double<P: AsRef<Path>>(file_path: P) -> Result<i32, String> {
    let mut file = match File::open(file_path) {
        Ok(file) => file,
        Err(err) => return Err(err.to_string()),
    };
    let mut contents = String::new();
    if let Err(err) = file.read_to_string(&mut contents) {
        return Err(err.to_string());
    }
    let n: i32 = match contents.trim().parse() {
        Ok(n) => n,
        Err(err) => return Err(err.to_string()),
    };
    Ok(2 * n)
}

fn main() {
    match file_double("foobar") {
        Ok(n) => println!("{}", n),
        Err(err) => println!("Error: {}", err),
    }
}
```

这段代码，看起来更舒服，更易懂。

