---
layout: post
title: Rust错误处理
date: 2020-05-04 23:00
categories: Rust
tag: Rust
excerpt:  Error Handle in Rust
---
# 前言

本文介绍Rust的错误处理。Rust的错误处理有其独到之处，非常值得学习。Rust将错误分成两类：

* 不可恢复的错误
* 可恢复的错误

不可恢复的错误一般是程序编写出现疏漏，不应该发生的事情发生了

* 比如除以0 ，引发SIGFPE
* 访问数组元素越界
* thread::spawn 无法创建线程 （比如内存耗尽 process bomb等原因）

这些错误是不该发生的，一般伴随着程序员代码中的某种错误，这种情况下，一般是不可处理的，程序员也不知道该如何处理这种错误，程序不宜继续运行下去。对于这种错误，Rust一般采用panic的做法，即让程序崩溃，立刻停止执行。

可恢复的错误，是指一些特殊的情况，但是这种情况发生了，我们完全可以处理。

比如说我们要创建一个配置文件，我们会尝试open这个文件，如果open失败，报不存在这个文件，这尽管是个报错，但是没关系，我们可以处理，我们可以创建此文件。对于这种类型的错误，Rust引入了Result，来处理。

下面我们分别针对panic和Result两种处理方式，分开阐述。

# 不可恢复的错误与panic!

一般来讲，不可恢复的错误是不该发生的，如下面场景：

* 访问数组元素越界
* 除以0
* 资源耗尽，无法创建进程

这种事情一旦发生，一般伴随着程序中存在bug。这种情况下，任何的修复尝试都是徒劳的，程序尽快终止才是最合理的做法。Rust提供了panic!宏。

panic!宏会接收println!-style的参数，将出错信息打印出来。

```Rust
fn my_div(n: i32, m: i32) -> i32 {
    if m == 0 {
        panic!("Invalid number: {}, div 0", m);
    }
    n/m
}

fn main() {
    println!("result is {}", my_div(9, 2));
    println!("result is {}", my_div(4, 0));
}
```

​	输出如下：

```Rust
result is 4
thread 'main' panicked at 'Invalid number: 0, div 0', src/main.rs:3:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

出了上面列出的通用场景可能会引发panic，从编程角度讲，有哪些情况可能会产生panic呢？

* Option::None 遭遇 unwrap()
* Option::None 遭遇 expect()
* Result::Err 遭遇unwrap()
* Result::Err 遭遇 expect()
* 直接调用panic!宏

发生panic或者调用panic!宏之后，Rust的行为有两种可选

* unwind （展开）
* abort    （终止）

## unwind

unwind是默认的行为，采用这种行为的条件下，下面事情会发生：

* 打印报错信息
* 程序会利用栈回退（stack unwind）机制，确保栈内构造的局部变量和指针的析构函数被一一调用掉，完成清理动作。
* 触发panic的线程会exit。

注意第三点，panic是per thread的。对于一个多线程程序而言，如果一个线程panic了，那么其他线程可以继续正常运行。这是多线程章节里面的内容，此处不展开。

## abort

遇到panic!另一种处理方法是abort，即立即退出，不做清理动作。我们可以在Cargo.toml中的[profile]中添加：

```
panic = 'abort'
```

或者采用如下方法：

```Rust
rustc -C panic=abort src/main.rs
```

选择发生发生panic时，直接abort。

```shell
result is 4
thread 'main' panicked at 'Invalid number: 0, div 0', src/main.rs:3:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
[1]    22576 abort (core dumped)  ./main
```

## catch_unwind

我们很多人都学习过C语言，语言崩溃的时候，也可以打印出stack，但是对于这种意想不到的事情发生的时候，尽可能丰富的打印当时的情况下信息，是后续调试 debug和改进的关键。

Rust在std::panic::catch_unwind 来捕捉panic。我们可以看下刚才的程序。

```Rust
use std::panic ;

fn my_div(n: i32, m: i32) -> i32 {
    if m == 0 {
        panic!("Invalid number: {}, div 0", m);
    }
    n/m
}

fn main() {
    let result = panic::catch_unwind(|| {
        println!("result is {}", my_div(9, 2));
        println!("result is {}", my_div(4, 0));
    }) ;

    match result {
        Ok(()) => println!("every thing goes OK"),
        Err(_) => println!("catch panic")
    }
}
```

我们调用了panic::catch_unwind ，将我们觉得有危险的部分可能panic的部分，包了起来。如果发生了panic，会被捕捉到。

运行结果如下：

```Rust
result is 4
thread 'main' panicked at 'Invalid number: 0, div 0', src/main.rs:5:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
catch panic
```

注意，panic的默认行为还是unwind，所以你看到了错误信息打印之类的，但是进程并没有退出，后面的catch panic得到了执行。

如果不想打印出来panicked at 之类的信息，应该怎么办？

std::panic提供了set_hook和take_hook两个方法：

```Rust
use std::panic ;

fn my_div(n: i32, m: i32) -> i32 {
    if m == 0 {
        panic!("Invalid number: {}, div 0", m);
    }
    n/m
}

fn main() {
    /*注册新的hook*/
    panic::set_hook(Box::new(|_info| {
        //do_nothing
    }));

    let result = panic::catch_unwind(|| {
        println!("result is {}", my_div(9, 2));
        println!("result is {}", my_div(4, 0));
    }) ;

    match result {
        Ok(()) => println!("every thing goes OK"),
        Err(_) => println!("catch panic")
    }

    let _ = panic::take_hook();

    let result = panic::catch_unwind(|| {
        println!("result is {}", my_div(9, 2));
        println!("result is {}", my_div(4, 0));
    }) ;

    match result {
        Ok(()) => println!("every thing goes OK"),
        Err(_) => println!("catch panic")
    }

}
```

因为首先用set_hook注册了新的hook，而新的hook什么也不做，所以，前半部的输出如下：

```Rust
result is 4
catch panic
```

后半部分，take_hook的作用是 unregister当前的hook，并且返回当前hook值。所以后半部分，有回归到了默认的行为，尽管被捕捉，但是还是会执行默认的unwind。

catch_unwind这个方法，很容易被熟悉python语言或者Java语言的人当作 try-catch来用，用于捕捉异常。但是下面的话需要务必重视：

```
极其不推荐使用catch_unwind处理常规错误。
```

**不要**用catch_unwind来做正常的流程控制，它的主要用途在：

1. 在FFI场景下的时候，如果说C语言调用了Rust的函数，在Rust内部出现了panic，如果这个panic在Rust内部没有处理好，直接扔到C代码中去，会导致C语言产生“未定义行为（undefined behavior）”
2. 某些高级抽象机制，需要阻止栈展开，比如线程池，如果一个线程中出现了panic，我们希望只把这个线程关闭，而不至于将整个线程池一起被拖下水

# Result

Rust其实提供了一个分层的错误处理方案：

* if something might reasonably be absent, Option is used
* if something goes wrong and can reasonably be handled, Result is used
* if something goes wrong and cannot reasonably be handled, the thread panics
* if something catastrophic happens, the program aborts

这个分层方案中，第一层就是Option，第二层是Result，其实严格意义上讲，Option，是Result的特例：

```Rust
enum Option<T>{
	None,
	Some(T),
}

enum Result<T, E>{
	Ok(T),
	Err(E),
}
```

使用Option的时候，一般表示不太关心出错原因，出错的时候，直接返回None，而Result的表达力要比Option更强，将错误的不同原因也包括进来。因此从这个角度讲，Option不过是Result的特例：

```
Option<T>  ~ Result<T, ()>
```

比如从HashMap中取值或者对Vector进行pop操作，前者出错的原因只有键值即key不存在，后者出错的原因只有Vector已经是空的了，这种情况下，Option比Result更合适。

比如从File中读取某个信息，出错的可能性很多，比如文件不存在，没有读取权限，数据不合法等，这种情况下，Option就力不逮己了，需要用Result<T,E>。

## Option

什么时候使用Option？

* 如果一个函数的某个参数的可选的
* 一个函数有返回值，但是其返回值可能是空的
* 一个数据类型，其值可能会为空

对于第一种情况：

```rust
fn get_full_name(fname: &str, lname: &str, mname: Option<&str>) -> String { // middle name can be empty
  match mname {
    Some(n) => format!("{} {} {}", fname, n, lname),
    None => format!("{} {}", fname, lname),
  }
}

fn main() {
  println!("{}", get_full_name("Galileo", "Galilei", None));
  println!("{}", get_full_name("Leonardo", "Vinci", Some("Da")));
}
```

middle name可能会空，所以用Option类型传递，如果为空，传None，如果不为空，传递Some(T)。

输出如下：

```shell
Galileo Galilei
Leonardo Da Vinci
```

对于第二种情况，比较典型的是从HashMap查找某键值对应的value，可能会不存在，或者从Vector中pop，可能会Vector已经为空，那么函数的返回值，最好是Option，因为Option类型会强制函数调用方必须要检查是None的情形，防止程序员编程时忽略。

我们以我们的除法为例，除以0会触发panic，但是我们可以封装一个除法，检查除数是否是0，如果除数是0，那么返回None，否则，返回Some(T)。

```Rust
fn checked_div(n: i32, m: i32) -> Option<i32> {
    if m == 0 {
        None
    }
    else {
        Some(n/m)
    }
}

fn try_div(dividened: i32, divisor: i32){
    match checked_div(dividened, divisor){
        None => println!("{} / {} failed", dividened, divisor),
        Some(result) => {
            println!("{} / {} = {}", dividened, divisor, result)
        },
    }
}

fn main() {
    try_div(9,2);
    try_div(9,0);
}
```

返回值是Option<i32>，所以try_div函数，必须要判断结果等于None时候，防止默认结果总是存在的。输出如下：

```
9 / 2 = 4
9 / 0 failed
```

另一个比较典型的例子是dirs::home_dir，在Linux下很多用户可能存在HOME 目录，但是也可能不存在，因此返回值是Option类型：

```Rust
pub fn home_dir() -> Option<PathBuf>
```

示例代码如下：

```Rust
extern crate dirs;

fn main() {
    let home_path = dirs::home_dir();
    match home_path {
        Some(p) => println!("{:?}", p), 
        None => println!("Can not find the home directory!"),
    }
}
```

输出如下：

```
"/home/manu"
```

## Result

Option 介绍完毕之后，就可以安心介绍最重要的Error Handle方法了，就是Result：

```
enum Result<T, E>{
	Ok(T),
	Err(E),
}
```

Result本质是一种enum，Ok表示执行成功的情况，Err表示执行失败的情况，其中T和E分别是Ok或者Err时，对应返回值的类型。如果你的函数或者方法，可能失败可能出错，建议返回类型选择Result。

枚举类型实例表示定义中的成员中的任意一项，反映到我们Result类型，就是要么时Ok，要么是Err，但是不可能既是Ok，又是Err，因此，如果函数返回Result类型，那么结果处于薛定谔的猫，即处于生和死的叠加，即Result处于Ok和Err的叠加态，函数不返回，无从得知到底是否成功。

那用这种数据类型有什么好处呢？好处在于，无论程序是否出错，函数的返回值的类型是相同的，函数调用者可以用相同的方式处理Result。从这个角度说，无论成功与否，函数调用者都要处理Result，错误处理称为业务逻辑的一部分，而不能随意的忽略。

（过去几年间，我处理过非常多的现场问题，我也修复过非常多的bug，我看过太多bug，仅仅是忽视Error Handle，程序编写者对于可能出现的错误并不在意，轻率或者轻浮地认为，很多错误并不会发生，或者发生了也就随它去吧，并不会认真地思考，这种错误发生的时候，应该怎么处理才比较妥当。）

### match处理 Result类型结果

因为Result类型的值是enum，所以典型的处理方法是模式匹配match。

Rust By Example中的的例子比较典型：

```Rust
mod checked {
    // Mathematical "errors" we want to catch
    #[derive(Debug)]
    pub enum MathError {
        DivisionByZero,
        NonPositiveLogarithm,
        NegativeSquareRoot,
    }

    pub type MathResult = Result<f64, MathError>;

    pub fn div(x: f64, y: f64) -> MathResult {
        if y == 0.0 {
            // This operation would `fail`, instead let's return the reason of
            // the failure wrapped in `Err`
            Err(MathError::DivisionByZero)
        } else {
            // This operation is valid, return the result wrapped in `Ok`
            Ok(x / y)
        }
    }

    pub fn sqrt(x: f64) -> MathResult {
        if x < 0.0 {
            Err(MathError::NegativeSquareRoot)
        } else {
            Ok(x.sqrt())
        }
    }

    pub fn ln(x: f64) -> MathResult {
        if x <= 0.0 {
            Err(MathError::NonPositiveLogarithm)
        } else {
            Ok(x.ln())
        }
    }
}

// `op(x, y)` === `sqrt(ln(x / y))`
fn op(x: f64, y: f64) -> f64 {
    // This is a three level match pyramid!
    match checked::div(x, y) {
        Err(why) => panic!("{:?}", why),
        Ok(ratio) => match checked::ln(ratio) {
            Err(why) => panic!("{:?}", why),
            Ok(ln) => match checked::sqrt(ln) {
                Err(why) => panic!("{:?}", why),
                Ok(sqrt) => sqrt,
            },
        },
    }
}

fn main() {
    // Will this fail?
    println!("{}", op(1.0, 10.0));
}

```

在上述例子中:
$$
f(x,y) = sqrt(ln(x/y))
$$
这个计算中有很多的失败可能：

* y = 0 ,导致 x/y panic   即 DivisionByZero
* x/y的结果 < 0 , 导致 ln运算异常 即NonPositiveLogarithm
* ln(x/y)的结果 < 0 ，导致sqrt异常 （不考虑无理数）即NegativeSquareRoot

因为有异常的可能，因此，每一步运算的结果，都是Result类型。每一层运算，都要用match来模式匹配，判断是否有异常发生。

```Rust
thread 'main' panicked at 'NegativeSquareRoot', src/main.rs:48:29
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

因为用match来处理错误是最常用最推崇的方法，我们下面再给出一个例子，来介绍用match来处理Result。

```Rust
use std::fs::File ;

fn main()
{
    let f = File::open("hello.txt");

    let _f = match f {
        Ok(file) => file ,
        Err(error) => {
            panic!("There was a problem opening the file {:?}", error)
        },
    };
}
```

结果用match匹配，如果出错，处理方法是panic!。

```Shell
thread 'main' panicked at 'There was a problem opening the file Os { code: 2, kind: NotFound, message: "No such file or directory" }', src/main.rs:10:13
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

上面两个例子中，出现异常都是粗暴的panic，但是以open为例，如果hello.txt不存在的话，可能我们可以接受，比如我们创建一个空文件出来。

```Rust
use std::fs::File ;
use std::io::ErrorKind ;

fn main()
{
    let f = File::open("hello.txt");

    let _f = match f {
        Ok(file) => file ,
        Err(error) => match error.kind() {
            ErrorKind::NotFound => match File::create("hell.txt") {
                Ok(fc) => fc ,
                Err(e) => panic!("Try to create file but encounter a problem{:?}", e)
            },
            other_error => panic!("There was a problem opening the file {:?}", other_error),
        },
    };
}
```

open失败的话，如果是NotFound错误，那么我们就尝试创建该文件。

```
创建file的接口为 std::fs::File::create ,这一次没有错误的写成creat。C接口中，创建文件的系统调用为creat，被戏称成最大的遗憾。go语言没有此遗憾，Rust一样没有此遗憾。
```

### unwrap 和 expect

处理Result类型，一般是用match，对于Err也要必须处理。但是这个世界上懒人是很多的，如果程序员确实不愿意处理Err，或者说处理也无益，直接崩溃是比较合理的做法，Rust对于这种情况提供了快捷方式，即unwrap和expect。

```Rust
use std::fs::File ;

fn main()
{
    let _f = File::open("hello.txt").unwrap();
}
```

输出如下：

```
thread 'main' panicked at 'called `Result::unwrap()` on an `Err` value: Os { code: 2, kind: NotFound, message: "No such file or directory" }', src/main.rs:5:14
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

如果使用expect函数：

```Rust
use std::fs::File ;

fn main()
{
    let _f = File::open("hello.txt").expect("failed to open hello.txt");
}
```

输出如下：

```
thread 'main' panicked at 'failed to open hello.txt: Os { code: 2, kind: NotFound, message: "No such file or directory" }', src/main.rs:5:14
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

在我们上面的代码中，File::open返回io::Result, Rust标准库中，有很多Result类型，其中io::Result就是其中之一。io::Result实现了unwrap和expect方法。如果函数执行成功，那么unwrap 和 expect方法就将正确的值取出来，如果出错，不做任何处理，直接panic。expect函数和unwrap函数功能点很接近，unwrap会打印出标准库内置的错误信息，而expect函数允许用户定义一个字符串，在程序挂掉的时候显示。

尽管我们介绍了unwrap和expect，但是这并不是推荐的错误处理方式，事实上，在严肃的商用程序上，不希望看到这两个函数出现。我们一般讲这两个函数用于原型设计。

### 传播错误

Rust允许程序像其他语言处理异常一样，讲错误扔给更上一层调用者，这被称为传播错误， Propagating Error。

```Rust
use std::io ;
use std::io::Read;
use std::fs::File ;


fn read_username_from_file() -> Result<String, io::Error>{
    let f = File::open("hello.txt");

    let mut f = match f {
        Ok(file) => file,
        Err(e) => return Err(e),
    };

    let mut s = String::new() ;
    match f.read_to_string(&mut s){
        Ok(_) => Ok(s),
        Err(e) => Err(e),
    }

}
fn main()
{
    let _s = read_username_from_file().unwrap();
}
```

我们看下上面的代码，从hello.txt中读取内容到String中，这个过程中可能会遇到很多错误：

* open 失败
* 读取失败

写程序的时候，会写很多的match，总体来讲，这些错误都是同一种类型的错误即 io::Error，对于read_username_from_file而言，对于这些io::Error也并没有做什么特殊的处理，遇到任何io::Error，提前返回错误，否则，将String返回。

Rust提供了传播错误的快捷方式，这个堪称比较好用的语法糖了。

```Rust
use std::io ;
use std::io::Read;
use std::fs::File ;

fn read_username_from_file() -> Result<String, io::Error>{
    let mut f = File::open("hello.txt")?;
    let mut s = String::new() ;
    f.read_to_string(&mut s)? ;
    Ok(s)

}
fn main()
{
    let _s = read_username_from_file().unwrap();
}
```

第五行 open后面的? ，作用如下：

* open成功，将Result中Ok包裹的值，赋值给变量f
* open如果失败，？会让整个函数退出执行，并且返回Err(error)给调用者。

有了这个神器，代码就可以写的比较简洁了，不需要一坨代码，match来match去。

事实上上面的代码还可以写的更加简洁：

```Rust
fn read_username_from_file() -> Result<String, io::Error>{
    let mut s = String::new() ;
    File::open("hello.txt")?.read_to_string(&mut s)?;
    Ok(s)
}
```



