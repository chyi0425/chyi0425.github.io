---
title: 'Binary Sort Tree,BST'
date: 2018-09-25 00:15:44
tags: [algorithm,Java] #文章标签，多于一项时用这种格式
toc: true
---

我们知道，二分查找可以缩短查找的时间，但是有个要求就是 查找的数据必须是有序的。每次查找、操作时都要维护一个有序的数据集，于是有了二叉排序树这个概念。


## 什么是二叉排序树

二叉排序树，又称二叉查找树、二叉搜索树。

二叉排序树是具有下列性质的二叉树：

* 若左子树不空，则左子树上所有结点的值均小于它的根结点的值；
* 若右子树不空，则右子树上所有结点的值均大于或等于它的根结点的值；
* 左、右子树也分别为二叉排序树。

![示意图](/img/b-s-t-1.png)

也就是说，二叉排序树中，左子树都比节点小，右子树都比节点大，递归定义。

根据二叉排序树这个特点我们可以知道，二叉排序树的中序遍历一定是从小到大的，比如上图，中序遍历结果是：

> 1 3 4 6 7 8 10 13 14

## 二叉排序树的关键操作

### 1.查找

根据二叉排序树的定义，我们可以知道在查找某个元素时：

* 先比较它与根节点，相等就返回；或者根节点为空，说明树为空，也返回；
* 如果它比根节点小，就从根的左子树里进行递归查找；
* 如果它比根节点大，就从根的右子树里进行递归查找。

可以看到，这就是一个**二分查找**。

代码实现：
```Java
public class BinarySearchTree {
    // root
    private BinaryTreeNode mRoot;

    public BinarySearchTree(BinaryTreeNode mRoot) {
        this.mRoot = mRoot;
    }

    /**
     * search data in tree
     *
     * @param data
     * @return
     */
    public BinaryTreeNode search(int data) {
        return search(mRoot, data);
    }

    /**
     * search data in the given tree
     *
     * @param node
     * @param data
     * @return
     */
    private BinaryTreeNode search(BinaryTreeNode node, int data) {
        if (node == null || node.getmData() == data) {
            return node;
        }
        if (data < node.getmData()) {
            return search(node.getmLeftChild(), data);
        } else {
            return search(node.getmRightChild(), data);
        }
    }
}
```

### 2.插入

二叉树中的插入，主要分两步：查找、插入：

* 先查找有没有整个元素，有的话就不用插入了，直接返回；
* 没有就插入到之前查到（对比）好的合适的位置。

插入时除了设置数据，还需要跟父节点绑定，让父节点意识到有你这个孩子：比父节点小的就是左孩子，大的就是右孩子。

代码实现
```Java
/**
     * insert data into tree
     *
     * @param data
     */
    public void insert(int data) {
        // if tree is null, new
        if (mRoot == null) {
            mRoot = new BinaryTreeNode();
            mRoot.setmData(data);
            return;
        }
        searchAndInsert(null, mRoot, data);
    }

    /**
     * search and insert
     *
     * @param parent the parent need to bind
     * @param node   the compare node
     * @param data   data
     * @return
     */
    private BinaryTreeNode searchAndInsert(BinaryTreeNode parent, BinaryTreeNode node, int data) {
        // if the compare node is null,so this position is empty,insert
        if (node == null) {
            node = new BinaryTreeNode();
            node.setmData(data);
            if (parent != null) {
                if (data < parent.getmData()) {
                    parent.setmLeftChild(node);
                } else {
                    parent.setmRightChild(node);
                }
            }
            return node;
        }
        if (node.getmData() == data) {
            return node;
        } else if (data < node.getmData()) {
            return searchAndInsert(node,node.getmLeftChild(),data);
        }else {
            return searchAndInsert(node,node.getmRightChild(),data);
        }
    }
```

### 3.删除 *
