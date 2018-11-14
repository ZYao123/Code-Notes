---
layout: post
title:  openidk_11_hashmap代码分析
date:   2018-11-13 14:45:00 +0800
categories: 源代码
tag: jdk
---

* content
{:toc}


Constructs							{#Constructs}
====================================
hashmap的构造器共有四种

```java
public HashMap(){...}
public HashMap(int initialCapacity){...}
public HashMap(int initialCapacity, float loadFactor){...}
public HashMap(Map<? extends K, ? extends V> m){...}
```
作用分别为：
1. 创建一个hashmap，加载因子`loadFactor`为默认的0.75。
2. 创建一个hashmap，map初始大小为`initialCapacity`。0 < initialCapacity<(1 << 30)
3. 创建一个hashmap，map初始大小为`initialCapacity`，加载因子为`loadFactor`
4. 从已有map创建hashmap。

put							{#put}
====================================

概要						{#Profile}
------------------------------------
hashmap的储存原理为，先通过hashmap计算key值，将key值放在hashmap中，从而使得取值时直接计算key的hash值。
当key的hash值相同时，则创建链表或二叉树储存，进行遍历以尽可能节约时间。

代码						{#Code}
------------------------------------
先看一下代码：
```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```
即：调用的实际是putVal方法，以下解析写在putVal注释中
```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
//      判断table是否存在，如不存在则创建，其中`table`为hashmap核心容器
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
//      hashmap当前节点是否为null，若为空则直接放进去，否则继续
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
//          判断key值是否相等（注1）
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
//      判断树节点是否相同（注2）        
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
//          计算当前链表的长度
            for (int binCount = 0; ; ++binCount) {
//              找到空节点，放入键值对
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
//              发现完全相同的key，跳出
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

hash值计算						{#hash}
------------------------------------
putVal方法中，第一个参数为`hash(key)`，调用方法如下：
```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
这是hashmap中生成key的hash值方法。
首先检查key是否为空，若为空则返回0。
生成hash值的核心代码为
```java
(h = key.hashCode()) ^ (h >>> 16)
```
首先直接取key的hashcode，这是一个native方法，在此不做讨论。
生成的hashcode为int型，共32位，将其无符号右移16位，即取hashcode的高16位。之后与hashcode进行或运算。
现说一个公式：当i=2^n时，有X%i=X&(i-1)。其中X为hashcode，即32位int型。
证明如下：
X可表示为二进制数：xn-xn-1-...-x2-x1-x0。
转为10进制表示方式：xn*2^n + x(n-1)*2^(n-1) + x(n-2)*2^(n-2) + ... x2*2^2 + x1*2^1 + x0*2^0。             (1)
所以当i为2^k时，X/i即为：式(1)/(2^k)                                                                      (2)
式(2)的商为：
xn*2^(n-k) + x(n-1)*2^(n-k-1) +... xk*2^0
余数为:x(k-1)*2^(k-1) + x(k-2)*2^(k-2) +... x1*2^1 + x0*2^0                                              (3)




备注							{#Note}
------------------------------------
1. hashcode不相同为对象不相同的必要不充分条件，即hashcode不同，对象必不相同，反之则不然。
如：
```java
System.out.println("Aa".hashCode());
System.out.println("BB".hashCode());
```
可见"Aa"、"BB"的hashcode相同，但"Aa"、"BB"显然不同。所以源代码中对比hashcode的同时，还比较了对象是否"=="或"equals"。
2. 当有多组key的hash值相同的键值对进行储存时，默认在hash表的该位置创建链表。
但当链表过长时，遍历链表时间复杂度为n。故，当链表过长时使用二叉树储存，将时间复杂度减少为log(n)。
如果判定链表过长? hashmap类中定义了`TREEIFY_THRESHOLD`，默认值为8，当链表长度大于`TREEIFY_THRESHOLD`时转为二叉树。



