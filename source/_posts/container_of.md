---
title: container_of探究
date: 2022-04-10 12:42:27
tags: kernel
---

From linux 4.14

<!-- more -->

```
/**
 * container_of - cast a member of a structure out to the containing structure
 * @ptr:        the pointer to the member.
 * @type:       the type of the container struct this is embedded in.
 * @member:     the name of the member within the struct.
 *
 */
#define container_of(ptr, type, member) ({                              \
        void *__mptr = (void *)(ptr);                                   \
        BUILD_BUG_ON_MSG(!__same_type(*(ptr), ((type *)0)->member) &&   \
                         !__same_type(*(ptr), void),                    \
                         "pointer type mismatch in container_of()");    \
        ((type *)(__mptr - offsetof(type, member))); })
```

The **offsetof** is defined as:

```
#undef offsetof
#ifdef __compiler_offsetof
#define offsetof(TYPE, MEMBER)  __compiler_offsetof(TYPE, MEMBER)
#else
#define offsetof(TYPE, MEMBER)  ((size_t)&((TYPE *)0)->MEMBER)
#endif
```

**container_of**宏可以把一个结构体成员变量的指针转成该结构体的指针，为了了解它是怎么实现的，首先需要知道结构体在内存中的布局：

![layout](layout.png)

首先无论是在栈或者堆上，结构体占用一段连续的内存，内部的成员依次排布（考虑 padding），例如：

```
stuct ST
{
	int a;
	char b;
	int c;
};
```

在 32 位机器上，按 4 字节对齐，结构体 ST 一共占用 12 字节，其中 a 占用 4 个字节，b 占用 1 个字节，后面有三个字节作为对齐，c 也同样占用 4 个字节。

其次，结构体的起始地址和第一个成员变量的起始地址一致，所以在下面的代码里&s == &s.a:

```
struct ST s = { 0 };
printf("s is at %p", &s);
printf("a is at %p", &s.a);
```

所以基于上面的结论，通过结构体成员的地址，推算出结构体的起始地址就不难了，只需要知道该成员在结构体中的偏移量就可以了，而偏移量其实就如前面说明的，在编译阶段，或者说编译器就可以知道。

计算偏移量是通过**offsetof**宏来实现的，来看具体的代码：

```
#undef offsetof
#ifdef __compiler_offsetof
#define offsetof(TYPE, MEMBER)  __compiler_offsetof(TYPE, MEMBER)
#else
#define offsetof(TYPE, MEMBER)  ((size_t)&((TYPE *)0)->MEMBER)
#endif
```
首先检查如果有编译器内置的方法__compiler_offsetof就直接调用，否则就模拟一个从0地址开始的结构体，得到成员变量的地址，因为偏移量就是成员变量的地址减去起始地址，而这个方法的精妙就在于把起始地址设置成了0，所以成员变量的地址就是该成员在结构体内部的偏移量。打个比方，用尺子量东西的长度，通常把起始点对准尺子的零刻度，这样另一端对准的尺子刻度就是该物体的长度。

最后，另外一个条件是，结构体的内存布局是从低地址到高地址的，换句话说，偏移量总为正。

OK，然后看**container_of**的实现
```
        void *__mptr = (void *)(ptr);                                   \
        BUILD_BUG_ON_MSG(!__same_type(*(ptr), ((type *)0)->member) &&   \
                         !__same_type(*(ptr), void),                    \
                         "pointer type mismatch in container_of()");    \
```
首先内部声明一个__mptr来指向传入的ptr，接着做了一个类型检查，确保ptr和对应的成员变量类型一致，或者ptr兼容void类型（？），这两步只是预备工作。关键的在最后一步：
```
        ((type *)(__mptr - offsetof(type, member))); })
```
offsetof宏会计算成员member在结构体里的偏移量，然后结构体成员的指针减去偏移量，就得到了结构体的指针。

总结下，**container_of**的实现依赖了下面三点：
* 结构体在内存中的存放布局是连续的
* 结构体的起始地址和第一个成员变量的地址是一致的（编译器没有添加额外的信息在里面）
* 结构体的布局是从低地址到高地址增长的，偏移量为正

