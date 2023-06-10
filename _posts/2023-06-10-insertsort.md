---
layout: post
title: 插入排序 C实现
date: 2023-06-10 22:00
categories: 算法
tag: 算法
excerpt: 插入排序的C实现


---

# 前言

复习《算法：C语言实现》一书，本文实现了插入排序。插入排序的思路和人打牌的时候，摸牌，把新摸到的牌插入到指定位置很相似。

```c
#ifndef _SORT_BASE_H_
#define _SORT_BASE_H_

typedef int Item ;

#define Key(A) (A)
#define less(A,B) (Key(A) < Key(B))
#define exch(A,B) { Item tmp = A; A=B ; B=tmp;}
#define compexch(A, B) if(less(B, A)) exch(A,B)


#endif

```



```c
#ifndef _INSERTSORT_H
#define _INSERTSORT_H

#include "sort_base.h"

void insertsort_slow(Item a[], int l , int r)
{
    int i ,j ;
    for(i = l+1 ; i <=r ; i++)
        for(j = i; j > l; j--)
            compexch(a[j-1], a[j]);
}

void insertsort(Item a[], int l,  int r)
{
    int i ;

    //put minimal value to the leftmost position , as sentinel key
    for(i = r ; i  > l; i--)
        compexch(a[i-1], a[i]);

    for( i = l+2 ; i <=r ; i++)
    {
        int j = i ;
        Item v = a[i] ;

        /* no need add j-1 >= l condition, because a[0] is alreaay the most minimal value already*/
        /* use while for break immediately to skip useless compare*/
        while(less(v, a[j-1]))
        {
            a[j] = a[j-1] ;
            j-- ;
        }
        a[j] = v ;
    }
}

#endif

```



insertsort_slow 这个版本，更容易理解，但是效率不高。

* 内层循环是从右往左比较，而左边的子序列实际上已经是有序的了，当碰到关键字不大于整备插入的数据项的关键字的时候，其实没有继续比较了，缺少了break机制，增加了无谓的比较。
* j > l 这个比较，大部分情况下是无用的，只有要插入的元素是最小的并且要插入数组的最前端的时候，该条件才有真正的意义。通常的一个改进思路是把最小的元素放在a[0] ,作为哨兵，只需要对a[1]~a[N]的元素进行排序即可。这种改进思路，可以大面积减少比较。

通过上述思路分析，将insertsort改进成最终的版本。

