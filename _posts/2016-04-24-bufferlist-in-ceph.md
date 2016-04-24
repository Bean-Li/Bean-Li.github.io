---
layout: post
title: ceph internal 之buffer list
date: 2016-04-24 21:34:40
categories: ceph internal
tag: ceph internal
excerpt: bufferlist是ceph中一个重要的基础的数据结构。
---

## 前言

buffer list是ceph中的一个基础的数据结构，代码中大量的使用。


## buffer::raw 
介绍buffer list之前，我们必须先介绍buffer::raw和buffer::ptr。相对于buffer list，这两个数据结构相对比较容易理解。


    class buffer::raw {
        public:
            char *data;
            unsigned len;
            atomic_t nref;

            mutable simple_spinlock_t crc_spinlock;
            map<pair<size_t, size_t>, pair<uint32_t, uint32_t> > crc_map;
            
            ...
    }
            

注意，这个数据结构，data，len，nref，这些成员变量不难猜测出其含义，相信我们设计数据结构也会有这几个变量，data是个指针，指向真正的数据，而len记录了该buffer::raw数据区数据的长度，nref表示引用计数。

注意，data指针指向的数据可能有不同的来源，最容易想到的当然是malloc，其次我们以可以使用mmap通过创建匿名内存映射来分配空间，甚至我们可以通过pipe管道＋splice实现零拷贝获取空间。有些时候，分配的空间时，会提出对齐的要求，比如按页对齐。

这是因为这些来源不同，要求不同，buffer::raw也就有了一些变体：

### buffer:raw_malloc

这个变体数据来源源自malloc，因此，创建的时候，需要通过malloc分配长度为len的空间，，而不意外，析构的时候，会掉用free释放空间。

```
    class buffer::raw_malloc : public buffer::raw {
        public:
            explicit raw_malloc(unsigned l) : raw(l) {
                if (len) {
                    data = (char *)malloc(len);
                    if (!data)
                        throw bad_alloc();
                } else {
                    data = 0;
                }
                inc_total_alloc(len);
                inc_history_alloc(len);
                bdout << "raw_malloc " << this << " alloc " << (void *)data << " " << l << " " << buffer::get_total_alloc() << bendl;
            }
            raw_malloc(unsigned l, char *b) : raw(b, l) {
                inc_total_alloc(len);
                bdout << "raw_malloc " << this << " alloc " << (void *)data << " " << l << " " << buffer::get_total_alloc() << bendl;
            }
            ~raw_malloc() {
                free(data);
                dec_total_alloc(len);
                bdout << "raw_malloc " << this << " free " << (void *)data << " " << buffer::get_total_alloc() << bendl;
            }
            raw* clone_empty() {
                return new raw_malloc(len);
            }
    };
```

### buffer::raw\_mmap_pages

顾名思义，也能够猜到，这个数据的来源是通过mmap分配的匿名内存映射。因此析构的时候，毫不意外，掉用munmap解除映射，归还空间给系统。

```
    class buffer::raw_mmap_pages : public buffer::raw {
        public:
            explicit raw_mmap_pages(unsigned l) : raw(l) {
                data = (char*)::mmap(NULL, len, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANON, -1, 0);
                if (!data)
                    throw bad_alloc();
                inc_total_alloc(len);
                inc_history_alloc(len);
                bdout << "raw_mmap " << this << " alloc " << (void *)data << " " << l << " " << buffer::get_total_alloc() << bendl;
            }
            ~raw_mmap_pages() {
                ::munmap(data, len);
                dec_total_alloc(len);
                bdout << "raw_mmap " << this << " free " << (void *)data << " " << buffer::get_total_alloc() << bendl;
            }
            raw* clone_empty() {
                return new raw_mmap_pages(len);
            }
    };
```

### buffer::raw\_posix_aligned

看名字也看出来了，对空间有对齐的要求。Linux下posix_memalign函数用来分配有对齐要求的内存空间。这种分配方式分配的空间，也是用free函数来释放，将空间归还给系统。

```
class buffer::raw_posix_aligned : public buffer::raw {
        unsigned align;
        public:
        raw_posix_aligned(unsigned l, unsigned _align) : raw(l) {
            align = _align;
            assert((align >= sizeof(void *)) && (align & (align - 1)) == 0);
#ifdef DARWIN
            data = (char *) valloc (len);
#else
            data = 0;
            int r = ::posix_memalign((void**)(void*)&data, align, len);
            if (r)
                throw bad_alloc();
#endif /* DARWIN */
            if (!data)
                throw bad_alloc();
            inc_total_alloc(len);
            inc_history_alloc(len);
            bdout << "raw_posix_aligned " << this << " alloc " << (void *)data << " l=" << l << ", align=" << align << " total_alloc=" << buffer::get_total_alloc() << bendl;
        }
        ~raw_posix_aligned() {
            ::free((void*)data);
            dec_total_alloc(len);
            bdout << "raw_posix_aligned " << this << " free " << (void *)data << " " << buffer::get_total_alloc() << bendl;
        }
        raw* clone_empty() {
            return new raw_posix_aligned(len, align);
        }
    };

```

后面还有基于pipe和splice的零拷贝方式，我们不赘述。从上面的函数不难看出，buffer::raw系列，就像他的名字一样，真的是很原生，并没有太多的弯弯绕，就是利用系统提供的API来达到分配空间的目的。

## buffer::ptr

buffer::ptr是在buffer::raw系列的基础上，这个类也别名bufferptr。


    src/include/buffer_fwd.h
    
    
    #ifndef BUFFER_FWD_H
    #define BUFFER_FWD_H
    
    namespace ceph {
      namespace buffer {
        class ptr;
        class list;
        class hash;
      }
    
      using bufferptr = buffer::ptr;
      using bufferlist = buffer::list;
      using bufferhash = buffer::hash;
    }
    
    #endif



这个类的成员变量如下,这个类是raw这个类的包装升级版本，它的_raw就是指向buffer::raw类型的变量。

```
        class CEPH_BUFFER_API ptr {
            raw *_raw;
            unsigned _off, _len;
          ......    
      }
```


![](/assets/ceph_internals/bufferptr.png)

很多操作都是很容易想到的：

```

   buffer::ptr& buffer::ptr::operator= (const ptr& p)
    {
        if (p._raw) {
            p._raw->nref.inc();
            bdout << "ptr " << this << " get " << _raw << bendl;
        }
        buffer::raw *raw = p._raw; 
        release();
        if (raw) {
            _raw = raw;
            _off = p._off;
            _len = p._len;
        } else {
            _off = _len = 0;
        }
        return *this;
    }
    
    buffer::raw *buffer::ptr::clone()
    {
        return _raw->clone();
    }
    
    void buffer::ptr::swap(ptr& other)
    {
        raw *r = _raw;
        unsigned o = _off;
        unsigned l = _len;
        _raw = other._raw;
        _off = other._off;
        _len = other._len;
        other._raw = r;
        other._off = o;
        other._len = l;
    }
    
   const char& buffer::ptr::operator[](unsigned n) const
    {
        assert(_raw);
        assert(n < _len);
        return _raw->get_data()[_off + n];
    }
    char& buffer::ptr::operator[](unsigned n)
    {
        assert(_raw);
        assert(n < _len);
        return _raw->get_data()[_off + n];
    }
    
    int buffer::ptr::cmp(const ptr& o) const
    {
        int l = _len < o._len ? _len : o._len;
        if (l) {
            int r = memcmp(c_str(), o.c_str(), l);
            if (r)
                return r;
        }
        if (_len < o._len)
            return -1;
        if (_len > o._len)
            return 1;
        return 0;
    }
```

## bufferlist

bufferlist才是我们的目的地，前两个类其实是比较容易理解的，但是bufferlist相对复杂一点。

bufferlist是buffer::list的别名：

```
#ifndef BUFFER_FWD_H
#define BUFFER_FWD_H

namespace ceph {
  namespace buffer {
    class ptr;
    class list;
    class hash;
  }

  using bufferptr = buffer::ptr;
  using bufferlist = buffer::list;
  using bufferhash = buffer::hash;
}

#endif


class CEPH_BUFFER_API list {
            // my private bits
            std::list<ptr> _buffers;
            unsigned _len;
            unsigned _memcopy_count; //the total of memcopy using rebuild().
            ptr append_buffer;  // where i put small appends

```

![](/assets/ceph_internals/bufferlist.png)

多个bufferptr形成一个list，这就是bufferlist。成员变量并无太多难以理解的地方，比较绕的是bufferlist的迭代器 ，理解迭代器，就不难理解bufferlist各个操作函数。

要理解bufferlist 迭代器，，首先需要理解迭代器成员变量的含义。

```
                        bl_t* bl;
                        list_t* ls;  // meh.. just here to avoid an extra pointer dereference..
                        unsigned off; // in bl
                        list_iter_t p;
                        unsigned p_off;   // in *p
```

* bl：指针，指向bufferlist
* ls：指针，指向bufferlist的成员 _buffers
* p: 类型是std::list<ptr>::iterator，用来迭代遍历bufferlist中的bufferptr
* p_off ： 当前位置在对应的bufferptr的偏移量
* off： 如果将整个bufferlist看成一个buffer::raw,当前位置在整个bufferlist的偏移量


这个递进关系比较明显，从宏观的bufferlist，递进到内部的某个bufferptr，再递进到bufferptr内部raw数据区的某个偏移位置。
此外还包含了当前位置在整个bufferlist的偏移量off。



注意p_off和off容易产生误解，请阅读seek函数仔细揣摩

seek(unsigned o)，顾名思义就是将位置移到o处，当然o指的是整个bufferlist的o处。ceph实现了一个更通用的advance，接受一个int型的入参。

如果o>0,表示向后移动，如果o小于0，表示想前移动。移动的过程中可能越过当前的bufferptr之指向的数据区。

```
    template<bool is_const>
        void buffer::list::iterator_impl<is_const>::advance(int o)
        {
            //cout << this << " advance " << o << " from " << off << " (p_off " << p_off << " in " << p->length() << ")" << std::endl;
            if (o > 0) {
                p_off += o;
                while (p_off > 0) {
                    if (p == ls->end())
                        throw end_of_buffer();
                    if (p_off >= p->length()) {
                        // skip this buffer
                        p_off -= p->length();
                        p++;
                    } else {
                        // somewhere in this buffer!
                        break;
                    }
                }
                off += o;
                return;
            }
            while (o < 0) {
                if (p_off) {
                    unsigned d = -o;
                    if (d > p_off)
                        d = p_off;
                    p_off -= d;
                    off -= d;
                    o += d;
                } else if (off > 0) {
                    assert(p != ls->begin());
                    p--;
                    p_off = p->length();
                } else {
                    throw end_of_buffer();
                }
            }
        }

    template<bool is_const>
        void buffer::list::iterator_impl<is_const>::seek(unsigned o)
        {
            p = ls->begin();
            off = p_off = 0;
            advance(o);
        }
```

除此以外，获取当前位置的ptr也很有意思，理解该函数也有帮助理解迭代器五个成员的含义。

    template<bool is_const>
        buffer::ptr buffer::list::iterator_impl<is_const>::get_current_ptr() const
        {
            if (p == ls->end())
                throw end_of_buffer();
            return ptr(*p, p_off, p->length() - p_off);
        }
        

相当于多个bufferptr对应的buffer::raw组成了一个可能不连续的buffer列表，因此使用起来可能不方便，ceph处于这种考虑，提供了rebuild的函数。该函数的作用是，干脆创建一个buffer::raw，来提供同样的空间和内容。

```
    void buffer::list::rebuild()
    {
        if (_len == 0) {
            _buffers.clear();
            return;
        }
        ptr nb;
        if ((_len & ~CEPH_PAGE_MASK) == 0)
            nb = buffer::create_page_aligned(_len);
        else
            nb = buffer::create(_len);
        rebuild(nb);
    }

    void buffer::list::rebuild(ptr& nb)
    {
        unsigned pos = 0;
        for (std::list<ptr>::iterator it = _buffers.begin();
                it != _buffers.end();
                ++it) {
            nb.copy_in(pos, it->length(), it->c_str(), false);
            pos += it->length();
        }
        _memcopy_count += pos;
        _buffers.clear();
        if (nb.length())
            _buffers.push_back(nb);
        invalidate_crc();
        last_p = begin();
    }
```
从下面测试代码中不难看出rebuild的含义，就是划零为整，重建一个buffer::raw来提供空间

```
  {
    bufferlist bl;
    const std::string str(CEPH_PAGE_SIZE, 'X');
    bl.append(str.c_str(), str.size());
    bl.append(str.c_str(), str.size());
    EXPECT_EQ((unsigned)2, bl.buffers().size());
    bl.rebuild();
    EXPECT_EQ((unsigned)1, bl.buffers().size());
  }
```

理解了上述的内容，bufferlist剩余的上千行代码，基本也就变成了流水账了，不难理解了，在此就不再赘述了。

