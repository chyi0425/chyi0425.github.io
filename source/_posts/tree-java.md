---
title: tree-java
date: 2018-09-27 09:53:40
tags: [algorithm,Java]
toc: true
---

数据结构，指的是数据的存储形式，常见的有线性结构（数组、链表，队列、栈），还有非线性结构（树、图等）。

## 什么是树

线性结构中，一个节点至多只有一个头节点，至多只有一个尾节点，彼此连接起来是一条完整的线。

比如链表和数组：

![示意图](/img/tree-java.png)

而树，非线性结构的典型例子，不再是一对一，而变成了**一对多**（而图则可以是 多对多），如下图所示：

![示意图](/img/tree-java-2.png)

可以看到:


* 图中的结构就像一棵倒过来的树，最顶部的节点就是“根节点 (root 节点)”
* 每棵树至多只有一个根节点
* 根节点生出多个孩子节点，每个孩子节点只有一个父节点，每个孩子节点又生出多个孩子
* 父亲节点 (parent) 和孩子节点 (child) 是相对的
* 没有孩子节点的节点成为叶子节点 (leaf)

## 树的相关术语

根节点、父亲节点、孩子节点、叶子节点如上所述。

![示意图](/img/tree-java-3.png)

### 节点的度

一个节点**直接含有**的子树个数，叫做节点的度。比如上图中的 3 的度是 2，10 的度是 1。

### 树的度

一棵树中**最大节点的度**，即哪个节点的子节点最多，它的度就是 树的度。上图中树的度为 2 。

### 树的高度

树的高度是从叶子节点开始，自底向上增加。

### 树的深度

与高度相反，树的深度从根节点开始，自顶向下增加。

> 整个树的高度、深度是一样的，但是中间节点的高度 和 深度是不同的，比如上图中的 6 ，高度是 2 ，深度是 3。

## 树的两种实现

从上述概念可以得知，树是一个递归的概念，从根节点开始，每个节点至多只有一个父节点，有多个子节点，每个子节点又是一棵树，以此递归。

树有两种实现方式：

* 数组
* 链表

### 数组实现

我们可以利用每个节点至多只有一个父节点这个特点，使用**父节点表示法**来实现一个节点：

```Java
public class TreeNodeArrImpl {
    private Object mData;   // the data
    private int mParent;    // parent node

    public TreeNodeArrImpl(Object mData, int mParent) {
        this.mData = mData;
        this.mParent = mParent;
    }

    public Object getmData() {
        return mData;
    }

    public void setmData(Object mData) {
        this.mData = mData;
    }

    public int getmParent() {
        return mParent;
    }

    public void setmParent(int mParent) {
        this.mParent = mParent;
    }

    public static void main(String[] args) {
        TreeNodeArrImpl[] arrayTree = new TreeNodeArrImpl[10];
    }
}
```

用数组实现的树表示下面的树
![示意图](/img/tree-java-4.png)

![示意图](/img/tree-java-5.png)

数组实现的树节点使用角标表示父亲的索引，下面用链表表示一个节点和一棵树：

### 链表表示的节点：

```Java
public class LinkedTreeNode {
    // the data
    private Object mData;
    // the parent node
    private LinkedTreeNode mParent;
    // the children nodes
    private List<LinkedTreeNode> mChildNodeList;

    public LinkedTreeNode(Object mData, LinkedTreeNode mParent) {
        this.mData = mData;
        this.mParent = mParent;
    }

    public Object getmData() {
        return mData;
    }

    public void setmData(Object mData) {
        this.mData = mData;
    }

    public LinkedTreeNode getmParent() {
        return mParent;
    }

    public void setmParent(LinkedTreeNode mParent) {
        this.mParent = mParent;
    }

    public List<LinkedTreeNode> getmChildNodeList() {
        return mChildNodeList;
    }

    public void setmChildNodeList(List<LinkedTreeNode> mChildNodeList) {
        this.mChildNodeList = mChildNodeList;
    }
}
```
使用引用，而不是索引表示父亲与孩子节点。

使用一个 List, 元素是 LinkedTreeNode，就可以表示一棵链表树

## 树的几种常见分类及使用场景

树，为了更好的查找性能而生。

常见的树有以下几种分类：

* 二叉树
* 平衡二叉树
* B 树
* B+ 树
* 哈夫曼树
* 堆
* 红黑树
