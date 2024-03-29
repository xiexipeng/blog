---
title: HashMap核心思想
date: 2019-07-13 18:07:00  
tags: 
- Java
categories: Java
keywords:  'HashMap'
description: HashMap的底层原理、核心思想和使用注意事项
cover:  https://s2.ax1x.com/2019/07/13/Z4wsmD.jpg
abbrlink: 
top_img: https://i.loli.net/2019/07/13/5d29b147aedc855326.jpg
---

# HashMap核心思想

> HashMap作为我们在日常开发中使用频率较高的一种集合，本文将介绍HashMap的基本原理和它的核心思想，以及在并发场景下的线程安全问题。



## 什么是HashMap

HashMap是一个通过Key-Value形式进行数据存储的集合，它通过hash算法，计算key对象的hash值，从而确定value值所在的内存地址。通过put(K, V)方法添加集合元素和get(Object)方法获取key对应value值对象。

HashMap的特性：

1. 允许保存key为null值的键值对，key为null的hash值为0
2. 支持自动扩容

## HashMap的核心思想

HashMap的核心思想将从三个方面进行介绍，分别是HashMap的数据结构、hash算法和HashMap的扩容。

### HashMap的数据结构

HashMap底层通过数组的方式进行数据的存储，`Node<K, V>[ ] table`，数组table的默认打下是16，数组中保存的Node节点包含key-value，Node节点中的`K key`和`V value`保存了实际put进去集合的数据。HashMap在添加数据的时候需要首先计算key的hash值，再对计算结果与table数组的大小取余，得到包含key-value值节点Node在数组中的下标位置。但是因为hash算法本身存在着hash碰撞，可能出现不相等的key得到的hash值是一样的，这样将导致不同Node需要保存进数组的同一个下标位。HashMap通过拉链法的方式解决hash碰撞，在添加元素的时候，计算新元素的数组下标位子，然后比较新添加元素的key是否与当前数组位子中Node节点的key想等，如果想等，则直接覆盖当前节点中的value值，如果不想等，则将新添加元素Node节点的下一节点指针指向当前数组位子中的Node节点，然后将新添加元素的Node节点赋值给当前数组下标，这样存在hash碰撞的Node将形成一个链表。最终HashMap的数据结构将形成这种数组+链表的结构。
![HashMap.png](https://i.loli.net/2019/07/13/5d2990671d64780619.png)

### hash算法

HashMap通过hash算法确定Node节点在数组的下标位子，hash算法具有以下特性：

1. hash算法需要效率高
2. hash算法计算的hash值需要尽可能散列

结合HashMap源码参考HashMap是如何做到这两点的

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

```

计算hash值的时候，首先判断key是否为空，如果为空，则返回0；如果不为空，则获取key的hashcode，将key的hashcode和hashcode向右移16位后，进行**异或**运算。将得到的hash值和数组大小的n-1做**与运算**，`i = (n-1) & hash`，其中n就是数组大小，i就是根据hash值计算所得数组下标。

通过以上两步可以确定key所在数组的下标，在以上两步中，i是通过n-1与hash值做与运算所得，数组默认大小是16，16-1也就是15，15转换成二进制也就是`0000 0000 00001111`，**与运算**的特点是运算位只要有一个为1结果就是1，任何一个数值与15的二进制进行**与运算**的结果一定在0-15之间，保证了数组不会越界。

而在计算hash值的过程中，需要将key的hashcode与hashcode移位后进行**异或**运算，这样做的目的是为了是hash值尽可能散列。假设一个hashcode的二进制值为`0010 1010 10001101`，如果将它直接将它(n-1)进行与运算，即：

```
0010 1010 10001101
		&
0000 0000 00001111
```

可以看到，当数组容量不大的时候，hash值的前几位没有参与到数组下标的计算，考虑一种极限情况，如果存在一组key，它们的hashcode转换的二进制位后四位一致，那么它们和`0000 0000 00001111`计算的数组下标将相等，根据HashMap的底层数据结构，它们将保存在同一个数组下标，从而导致HashMap退化成一个链表。

通过将key的hashcode向右移16位（一般来说HashMap的大小不会超过16位），使得hashcode的前面几位也能参与到数组下标的计算过程，使用**异或运算**（如果运算位不一致则为1，如果一致则为0），使得计算过程既不偏向1也不偏向0，假设另一种极端情况，如果存在一组key，它们的hashcode转换的二进制后的前面几位都是1，或者都是0，例如：

```
1111 1111 1111 1111 1000 1101 10001101

1111 1111 1111 1111 0000 0101 10101101

1111 1111 1111 1111 0101 0101 00101111
```

这样的一组hashcode向右移16位后进行与运算，结果就是：

```
0000 0000 0000 0000 1000 1101 10001101

0000 0000 0000 0000 0000 0101 10101101

0000 0000 0000 0000 0101 0101 00101111
```

可以看到后面16位是没有变化的，如果这一组hashcode的后16位散列程度不够，这样依旧存在HashMap的退化问题。而使用**异或运算**，不管前16位都是1，还是都是0，都将对后16位进行重新运算得到新的结果。

HashMap为了提高hash算法的散列性，尽可能是hashcode的每一位都参与hash运算，保证数据在添加到数组的时候足够均匀。同时在计算hash值的时候使用位运算，极大的提高了运算效率。



### HashMap的扩容



## HashMap的并发问题