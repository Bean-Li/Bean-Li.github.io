---
layout: post
title: AsRef and Borrow trait in Rust
date: 2020-03-29 10:29
categories: Rust
tag: Rust
excerpt:  AsRef and Borrow trait in Rust
---
# AsRef

在Rust中，我们打开文件，经常使用如下接口：

```Rust
let dot_emacs = std::fs::File::open(""/home/manu/.emacs")
```

注意，在std::fs的实现中，Struct File实现了open method，该method的定义如下：

```Rust
pub fn open<P: AsRef<Path>>(path: P) -> Result<File>
```

open方法真正需要的参数是 &Path类型，可是我们知道，我们就是传个String类型或者str类型给它，类型也是匹配的，为何如此？

就在open声明中的 <P: AsRef<Path>> ，即任何实现了AsRef<Path>的类型，都OK。

* String
* str
* OsString
* OsStr

这些类型，都实现了AsRef<Path>，可以隐式地转成Path类型。当然了，Path和PathBuf更是可以作为参数传给open。

# Borrow

Rust除了AsRef以外，还提供了一个Borrow的Trait，这两个Trait的意图非常接近，以至于，有人问是否是重复的 [`Borrow` and `AsRef` are redundant #24140](https://github.com/rust-lang/rust/issues/24140)

```Rust
trait AsRef<T: ?Sized>{
	fn as_ref(&self)-> &T ;
}

trait Borrow<Borrowed: ?Sized> {
  fm borrow(&self)-> &Borrowed ;
}
```

两者从定义上看，几乎是一模一样的，是多余重复么？

并非如此！

我们以String为例，String类型同时实现了

* AsRef<&str>
* AsRef<[u8]>
* AsRef<Path>

当时这三种类型有不同的hash值，这三种中，只有&str类型保证hash值与String的hash值一致，因此String类型只实现了Borrow<str>。

从上面的讨论可以看出，Borrow通常用于HashMap和BTreeMap这种数据结构，两种类型的hash的值或者排序的值，必须是一样的，方能Borrow。所以要比AsRef要窄。

```Rust
There are a few really important differences:

Borrow is used by data structures like HashMap and BTreeMap which assume that the result of hashing or ordering based on owned and borrowed values is the same. But there are many AsRef conversions for which this isn't true. So in general, Borrow should be implemented for fewer types than AsRef, because it makes stronger guarantees.
Borrow provides a blanket implementation T: Borrow<T>, which is essential for making the above collections work well. AsRef provides a different blanket implementation, basically &T: AsRef<U> whenever T: AsRef<U>, which is important for APIs like fs::open that can use a simpler and more flexible signature as a result. You can't have both blanket implementations due to coherence, so each trait is making the choice that's appropriate for its use case.
This should be better documented, but there's definitely a reason that things are set up this way.
```

下面我们以HashMap为例，考虑将String映射成i32，哈希表的键值是String，我们如何写查找函数？

下面是直觉之下的第一次尝试：

```Rust
impl HashMap<K, V> where K: Eq + Hash 
{
    fn get(&self, key:K)-> Option<&V> {
    	...
    }
}
```

初看下来，还不错，但是细细一想，本次任务，我们需要String类型，用值传递，太奢侈浪费了，改成引用，得到第二版：

```Rust
impl HashMap<K, V> where K: Eq + Hash 
{
    fn get(&self, key: &K)-> Option<&V> {
    	...
    }
}
```

第二个版本，比第一个版本是好一些了，但是我们想象，如果我们的get函数实现成这样，如果我们要使用get函数，我们的代码可能不得不写成这样：

```Rust
hashtable.get(&"twenty-two".to_string())
```

我们不得不先分配String Buffer，然后将我们的常量字符串拷贝到堆（heap）的空间上，然后用把引用传给哈希表的get函数，然后drop掉这个String。

这个过程太荒谬，太浪费了。

* 分配空间
* 拷贝字符串
* 执行drop，销毁String

这些个操作都太浪费了，我们明明有"twenty-two" 这个str。所以最终标准库里面的最终版本如下：

```Rust
impl HashMap<K,V> where K: Eq  + Hash
{
	fn get<Q: ?Sized>(&self, key: &Q)->Option<&V>
	where K: Borrow<Q>,
				Q: Eq + Hash
	{...}
}
```

String 同时实现了Borrow<String>和 Borrow<str>,因此，get函数，无论是传&String类型，还是&str类型作为key，都是可以的，从而解决困境。
