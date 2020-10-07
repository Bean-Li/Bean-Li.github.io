---
layout: post
title: Arc in Rust
date: 2020-10-07 11:00
categories: Rust
tag: Rust
excerpt: Rc and Arc in Rust
---

# 前言

在Rust语言中，所有权大多数情况下是明确的，对于一个给定的值，你可以准确地判断出，哪个变量拥有它。但是也存在一些情况，单个值可能同时被多个所有者持有。

请看下面的例子：

 ```Rust
`  fn process_files_in_parallel(filenames: Vec<string>,  glossary: &GigabyteMap) 
     -> io::Result<()> 
 {
      ...
      for worklist in worklists
      {
          thread_handlers.push(
              spawn(move || process_files(worklist, glossary)
          );
      }
      ....
 }
`
 ```

上面的例子中，拟采用多个进程，并发地处理文件。但是处理文件需要一个消耗巨大内存的数据结构， glossary，可能数据结构大小超过GB，这种情况下，拷贝多个副本给不同线程使用无疑是无法承受的。但是上面的代码，存在一个问题，即glossary的生命周期。

因为glossary并不能保证在所有的子进程完成任务之前不被销毁，如果被销毁，会导致子进程无法正常完成任务。

# Rc and Arc

Rust提供了一个名为Rc<T>的类型来支持多重所有权，它名称中的Rc是Reference Counting 引用计数的缩写。Rc<T>类型会在内部维护一个记录值引用计数的计数器，从而确定这个值是否仍在被使用。如果对一个值的引用计数变成了0，意味着这个值可以被安全地清理掉。

那Arc是干啥的呢？Arc具备Rc的一切特征，那是Rc不是线程安全的，所以Rust又提供了一个Arc<T>的类型，来确保多个线程之前操作某个变量的安全性。这个Arc中的A指的是Atomic，即Atomic Reference Counting。

那看起来是Arc是更强大的Rc，能够保证安全地用在并发的场景，那为什么不干脆消灭掉Rc，统统只使用Arc呢？

没有免费的午餐，Arc更强大，但是这种强大，需要付出一些性能开销才能做到，如果是单线程条件下，我们其实不需要付出这种开销。因此

* 单线程条件下，可以使用Rc
* 多线程并发条件下，建议使用Arc

```Rust
use std::thread;
use std::time::Duration;
use std::sync::Arc;


fn main() {
    let foo = Arc::new(vec![0]);
    for _ in 0..10 {
        let bar = Arc::clone(&foo);
        thread::spawn(move || {
            thread::sleep(Duration::from_millis(200));
            println!("{:?}", &bar);
        });
    }
    println!("{:?}", foo);
}
```

上面的用法中， Arc::clone 并不是复制了一份vec，而是仅仅增减了一个引用计数，这是多个线程之间共享数据的一个方法。

