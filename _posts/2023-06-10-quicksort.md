---

layout: post
title: 快速排序 C实现
date: 2023-06-05 10:29
categories: 算法
tag: 算法
excerpt: 快速排序的C实现

---
# 前言
最近在复习数据结构，参考的是《算法：C语言实现》。
本文实现了quicksort，对于某些特殊序列，quicksort恶化的情况，采用了三者取中的方式，防止快排性能恶化。

对于选择整个输入中第K小的元素，可以不必完成整个排序之后，再来取对应位置的值，可以参考快排的思路，快速获得整个序列中第Kth的元素。同时实现了递归和非递归的版本。


```C
#ifndef _SORT_BASE_H_
#define _SORT_BASE_H_

typedef int Item ;

#define Key(A) (A)
#define less(A,B) (Key(A) < Key(B))
#define exch(A,B) { Item tmp = A; A=B ; B=tmp;}
#define compexch(A, B) if(less(B, A)) exch(A,B)


#endif
```



```C
#ifndef _QUICKSORT_H_
#define _QUICKSORT_H_

#include "sort_base.h"
#include "insertsort.h"

#define CUTOFF (8)

int partition(Item a[], int l, int r)
{
    int i = l-1;
    int j = r ;
    Item v = a[r];
    for(;;)
    {
        while(less(a[++i],v));
        while(less(v, a[--j]))
        {
            if(j == l)
                break;
        }
        if(i >=j)
            break;
        exch(a[i], a[j]);
    }
    exch(a[i], a[r]);
    return i ;
}

void _quicksort(Item a[], int l, int r)
{
    if(r - l <=CUTOFF)
        return ;

    /* find the middle element of (a[l], a[mid] , a[r])*/
    exch(a[(l+r)/2], a[r-1]);

    compexch(a[l], a[r-1]);
    compexch(a[l], a[r]);
    compexch(a[r-1], a[r]);

    /* a[r-1] is a[mid]
     * a[l] < a[r-1]
     * a[r-1] < a[r]
     * a[l] and a[r] no need to appear in partition function
     */
    int i = partition(a, l+1, r -1);
    _quicksort(a, l , i-1);
    _quicksort(a, i+1 , r);

}


void quicksort(Item a[], int l , int r)
{
    _quicksort(a, l , r);
    insertsort(a, l , r);
}


/* recursive version of find kth element*/
Item find_kth_element(Item a[], int l, int r, int k)
{
    int pivot = partition(a, l , r);
    if (pivot == k)
        return a[pivot];

    if (pivot > k)
        return find_kth_element(a, l, pivot-1, k);
    else
        return find_kth_element(a, pivot+1, r, k);

}

/* non-recursive version of find kth element*/
Item find_kth_element_n(Item a[], int l, int r, int k)
{
    while(l < r)
    {
        int pivot = partition(a, l , r);
        if (pivot == k)
            return a[pivot];
        if (pivot > k)
            r  = pivot - 1;
        else
            l =  pivot + 1 ;
    }
    
    return a[l];
}

#endif

```

```c++
#include <algorithm>
#include <cassert>
#include <cstdint>
#include <cstdio>
#include <cstdlib>
#include <ctime>
#include <vector>
#include "shellsort.h"
#include "quicksort.h"


std::vector<uint32_t> generate_test_vec(size_t n) {
    auto test_vec = std::vector<uint32_t> {};
    while (test_vec.size() < n) {
        test_vec.push_back((uint32_t) std::rand() %100000);
    }
    return test_vec;
}

int cmp_int(const void* a, const void *b)
{
    return ( *(int*)a - *(int*)b );
}

int main() {
    for (int round = 0; round < 3; round++) {
        std::printf("round %d:\n", round);

        auto test_vec1 = generate_test_vec(2000000);
        auto test_vec2 = test_vec1;
        auto test_vec3 = test_vec1;
        auto test_vec4 = test_vec1;

        auto start_time1 = std::clock();
        //std::sort(test_vec1.begin(), test_vec1.end());
        qsort(test_vec1.data(), test_vec1.size(), 4, cmp_int);
        std::printf("qsort(glibc) time: %.3lf sec\n", (std::clock() - start_time1) * 1.0 / CLOCKS_PER_SEC);

        auto start_time2 = std::clock();
        shellsort_2((Item*)test_vec2.data(), 0, test_vec2.size()-1);
        std::printf("shellsort time: %.3lf sec\n", (std::clock() - start_time2) * 1.0 / CLOCKS_PER_SEC);

        auto start_time3 = std::clock();
        quicksort((Item*)test_vec3.data(), 0, test_vec3.size()-1);
        std::printf("quicksort time: %.3lf sec\n", (std::clock() - start_time3) * 1.0 / CLOCKS_PER_SEC);

        auto start_time4 = std::clock();
        std::sort(test_vec4.begin(), test_vec4.end());
        std::printf("std::sort time: %.3lf sec\n", (std::clock() - start_time4) * 1.0 / CLOCKS_PER_SEC);

        assert(test_vec1 == test_vec2);
        assert(test_vec1 == test_vec3);
        assert(test_vec1 == test_vec4);

    }
}

```

```

ROG-Manjaro C/sort » ./test
round 0:
qsort(glibc) time: 0.556 sec
shellsort time: 0.898 sec
quicksort time: 0.454 sec
std::sort time: 0.989 sec
round 1:
qsort(glibc) time: 0.555 sec
shellsort time: 0.800 sec
quicksort time: 0.393 sec
std::sort time: 1.064 sec
round 2:
qsort(glibc) time: 0.507 sec
shellsort time: 0.900 sec
quicksort time: 0.498 sec
std::sort time: 1.145 sec

```

