---
layout: post
title: rayon join  in Rust
date: 2020-10-07 11:00
categories: Rust
tag: Rust
excerpt: Rc and Arc in Rust
---

# 前言

Rust提供了std::thread::spawn，可以通过Fork-Join模式，完成并发。这个基本概念对于熟悉C/C++编程的程序员，并没有什么太多的挑战。介绍Rust并发的资料中，很多资料都不约而同地提到了Rayon，这个library非常有趣，也非常强大，他是Niko Matakis这位大神完成的。对这个大神感兴趣的，可以读他的[blog](http://smallcultfollowing.com/babysteps/)。

```Rust
extern creat rayon ;
use rayon::prelude::* ;

let (v1, v2) = rayon::join(fn1, fn2); 

giant_vector.par_iter().for_each(|value| {
    do_something_with_value(value) ; 
});
```



rayon这个库，提供了非常方便的接口，程序员很容易将串行的接口改造成成并行的：

```Rust
extern crate rayon ;
use rayon::prelude::*;

//sequential
let total_price = stores.iter()
                        .map(|store| store.compute_price(&list))
                        .sum();
//parallel                       
let total_price = stores.par_iter()
                        .map(|store| store.compute_price(&list))
                        .sum();                      
```

我们看到，将迭代器iter()变成了par_iter()就完成了并行处理，对串行代码的改造非常方便，其中par_iter是parallel iterator的缩写。

```Rust
Rayon’s goal is to make it easy to add parallelism to your sequential code
```



# Rayon的核心原语 join

join是Rayon的核心原语，前面提到的par_iter是构建在join之上的。因此，理解rayon，需要先理解join。

join的使用非常简单：

```rust
join(|| do_something(), || do_something_else());
```

其函数原型如下：

```rust
pub fn join<A, B, RA, RB>(oper_a: A, oper_b: B) -> (RA, RB) 
where
    A: FnOnce() -> RA + Send,
    B: FnOnce() -> RB + Send,
    RA: Send,
    RB: Send, 
```

在rayon实现中，是否会有并发线程一起处理两个closure，取决于是否有空闲的CPU core，即join的两个闭包是one by one地串行执行还是并发之行，取决于实际情况。

rayon采用了一种叫做work-stealing的技术，简单地说， join(a,b)，我们有两个任务要处理，a和b，而且这两个任务是并发安全的。我们并不知道，threadpool中是否有idle的thread可以处理， 处理方法如下：

* 把b放入到pending work queue
* 执行a
* 如果存在一个thread 空闲，扫瞄pending queue，如果找到任务，则执行它
* 执行a任务的线程执行a完毕后，检查b的情况：
  * 是否有其他线程执行了b，如果没有，该线程负责执行b
  * 如果存在其他线程执行了b，等待期间，可以偷其他任务完成。

其伪代码大概如下：

```Rust
fn join<A,B>(oper_a: A, oper_b: B)
    where A: FnOnce() + Send,
          B: FnOnce() + Send,
{
    // Advertise `oper_b` to other threads as something
    // they might steal:
    let job = push_onto_local_queue(oper_b);
    
    // Execute `oper_a` ourselves:
    oper_a();
    
    // Check whether anybody stole `oper_b`:
    if pop_from_local_queue(oper_b) {
        // Not stolen, do it ourselves.
        oper_b();
    } else {
        // Stolen, wait for them to finish. In the
        // meantime, try to steal from others:
        while not_yet_complete(job) {
            steal_from_others();
        }
        result_b = job.result();
    }
}
```



# rayon join 示例

```rust
let mut v = vec![5, 1, 8, 22, 0, 44];
quick_sort(&mut v);
assert_eq!(v, vec![0, 1, 5, 8, 22, 44]);

fn quick_sort<T:PartialOrd+Send>(v: &mut [T]) {
   if v.len() > 1 {
       let mid = partition(v);
       let (lo, hi) = v.split_at_mut(mid);
       rayon::join(|| quick_sort(lo),
                   || quick_sort(hi));
   }
}

fn partition<T:PartialOrd+Send>(v: &mut [T]) -> usize {
    let pivot = v.len() - 1;
    let mut i = 0;
    for j in 0..pivot {
        if v[j] <= v[pivot] {
            v.swap(i, j);
            i += 1;
        }
    }
    v.swap(i, pivot);
    i
}
```

上面给出了一个rayon join的示例，该示例中，join充分利用了多核，即多个CPU一起发挥作用参与排序。在一个4-Core的Macbook Pro上，我们可以看到，随着数组长度的增大，排序效率比单线程的quicksort快很多，因为是4-Core，因此最多也是快4倍，无法更快了。

| Array Length | Speedup |
| ------------ | ------- |
| 1K           | 0.95x   |
| 32K          | 2.19x   |
| 64K          | 3.09x   |
| 128K         | 3.52x   |
| 512K         | 3.84x   |
| 1024K        | 4.01x   |

这个结果是原作者对代码做了一些优化，即如数组长度低于5K，就使用串行的排序：

```rust
fn quick_sort<J:Joiner, T:PartialOrd+Send>(v: &mut [T]) {
    if v.len() <= 1 {
        return;
    }

    if J::is_parallel() && v.len() <= 5*1024 {
        return quick_sort::<Sequential, T>(v);
    }

    let mid = partition(v);
    let (lo, hi) = v.split_at_mut(mid);
    J::join(|| quick_sort::<J,T>(lo),
            || quick_sort::<J,T>(hi));
}
```

