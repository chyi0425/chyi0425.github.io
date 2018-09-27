---
title: Binary Tree
date: 2018-09-25 00:16:08
tags: [algorithm] #文章标签，多于一项时用这种格式
toc: true
---

## 什么是二叉树

二叉树是有限个节点的集合，这个集合可以是空集(空二叉树)，也可以是一个根节点和至多两个子二叉树组成的集合，其中一颗树叫做根的左子树，另一棵叫做根的右子树。

### 二叉树的特点

1. 每个节点最多有两棵子树,所以二叉树中不存在度大于2的节点.注意不是只有两棵子树,而是最多有两棵子树,所以说没有子树或者有一颗子树也是可以的.
2. 左子树和右子树是有顺序的,次序不能任意点到.就像人是双手、双脚,但显然左手、左脚和右手、右脚是不一样的,右手戴左手套、右脚穿左鞋都会及其别扭和难受.
3. 即使树中的某一个节点只有一棵子树,也要区分他是左子树还是右子树.

### 二叉树的五种基本形态

![示意图](/img/b-t-1.jpg)

### 特殊二叉树

有两种特殊的二叉树：

#### 满二叉树

> 定义：在一棵二叉树中,如果所有的分支节点都存在左子树和右子树,并且所有叶子都在同一层面上,这样的二叉树成为满二叉树.

##### 满二叉树的特点
1. 满二叉树的叶子只能出现在最下一层,出现在其它层就不可能达成平衡.
2. 非叶子节点的度一定是2.
3. 在同样深度的二叉树中,满二叉树的节点个数最多,叶子数最多

#### 完全二叉树

> 定义： 对于一棵具有 n 个节点的二叉树按层序编号,如果编号为 i ( 1<= i <= n)的节点与同样深度的满二叉树中编号为 i 的节点在二叉树中位置完全相同,则这棵二叉树成为完全二叉树.

##### 完全二叉树的特点

1. 完全二叉树的叶子节点只能出现在最下面的两层.
2. 最下层的叶子一定集中在左部的连续位置.
3. 倒数二层,若有叶子节点,则一定在右部的连续位置
4. 如果节点度为1,则该节点只有有孩子,即不存在只有右子树的情况.
5. 同样节点数的二叉树,完全二叉树的深度最小.

#### 完全二叉树与满二叉树的区别

看完上面的完全二叉树和满二叉树的定义和特点,可能对两种特殊的二叉树有所混淆,所以我们来区分一下两种二叉树,首先从字面上区分,"完全"和"满"的差异,满二叉树一定是一棵完全二叉树,但完全二叉树不一定是满的.其实,完全二叉树的所有节点与同样深度的满二叉树,他们按层序编号相同的节点,是一一对应的,这里有个关键词是按层序编号.

![示意图](/img/b-t-2.jpg)

## 二叉树的性质

### 二叉树性质1

> 内容:在二叉树的第 i 层上有至多 2^( i-1) 个节点 (i >= 1).

第一层是根节点,只有一个,所以是 2^(1-1) = 2^0 = 0.
第二层有两个,2^(2-1) = 2 ^ 1 = 2.
第三层有四个,2^(3-1) = 2 ^ 2 = 4.
所以通过数据归纳法的论证,很容易得到二叉树的第 i 层上至多有2(i -1) 个节点的结论.

![示意图](/img/b-t-3.jpg)

### 二叉树性质2

> 内容: 深度为 k 的二叉树至多有 2^k - 1 个节点(k >= 1).

如果有一层,至多有1 = 2 ^ 1 -1 个节点.
如果有两层,至多有1 + 2 = 2^2 -1个节点.
如果有三层,至多有1 + 2 + 4 = 2^3 -1个节点.
所以通过数据归纳法的论证,很容易得到深度为 k 的二叉树至多有 2^k - 1 个节点(k >= 1)的结论.

### 二叉树性质3

> 内容:对于任意一棵二叉树T,如果其终端节点数为n0,度为2的节点书为n2,那么有n0 = n2 + 1;
> (注:度的解释,一个节点有n个子节点,那么他就是度为n的节点)


终端节点数其实就是叶子节点数，除了叶子节点外，剩下的就是度为1或22的节点数了。我们设n1为度1的节点数，则树的总节点数为n=n1+n2+n0

我们来看下图,这个二叉树的总节点数为10,它是由A,B,C,D等度为2的节点,F,G,I,J等度为0的叶子节点和E这个度为1的节点组成.总和为4 + 1 + 5 = 10;

再来看一下,下图的总的连接线数为9,用代数表达式就是分支总数= n -1 = n1 + n2 ,因为我们刚才有等式 n = n0 + n1 + n2 ; 所以可以推导出来 n0 +n1 +n2 -1 = n1 +2 * n2, 结论就是n0 = n2 + 1;

![示意图](/img/b-t-4.jpg)

### 二叉树性质4

> 内容:具有n个节点的完全二叉树的深度为 [log (2) n] + 1 ([x] 表示不大于x的最大整数) .

> 一个完全二叉树的节点数一定少于等于同样读书的满二叉树的节点数2k-1,但一定多于2(k-1) -1;
> 即为2^(k-1) -1 < n <= 2^k-1;由于节点数为整数,所以我们可以简化不等式,所以 2^(k-1) <= n < 2^k,不等式两边去对数 k-1<=log2n<k,因此 k = [log (2) n] + 1.(注:k为深度)

### 二叉树性质5

> 内容:如果对一棵有 n 个节点的完全二叉树(其深度为[log (2) n] + 1) 的节点按层序编号(从第1层到第[log (2) n] + 1 层,每一层从左到右),对任一节点i (1 <= i <= n) 有:
> 1. 如果 i = 1 ,则节点i是二叉树的根,无双亲;如果i > 1,则其双亲是节点 [ i /2 ];
> 2. 如果2i > n,则节点i无左孩子(节点i为叶子节点);否则其左孩子是节点2i;
> 3. 如果2i +1 > n,则节点i无右孩子;否则其右孩子是节点2i +1;

## 二叉树的遍历

### 前序遍历

>  前序遍历首先访问根节点然后遍历左子树，最后遍历右子树。在遍历左、右子树时，仍然先访问根节点，然后遍历左子树，最后遍历右子树

若二叉树为空则结束返回，否则：
1. 访问根节点。
2. 前序遍历左子树。
3. 前序遍历右子树 。

![示意图](/img/b-t-6.jpg)
```Java
/**
 * 先序遍历
 * @param node
 */
public void iterateFirstOrder(BinaryTreeNode node){
    if (node == null){
        return;
    }
    operate(node);
    iterateFirstOrder(node.getLeftChild());
    iterateFirstOrder(node.getRightChild());
}
```

### 中序遍历

> 中序遍历首先遍历左子树，然后访问根节点，最后遍历右子树。在遍历左、右子树时，仍然先遍历左子树，再访问根节点，最后遍历右子树。即：

若二叉树为空则结束返回,否则：
1. 中序遍历左子树
2. 访问根节点
3. 中序遍历右子树

![示意图](/img/b-t-7.jpg)
```Java
/**
 * 中序遍历
 * @param node
 */
public void iterateMediumOrder(BinaryTreeNode node){
    if (node == null){
        return;
    }
    iterateMediumOrder(node.getLeftChild());
    operate(node);
    iterateMediumOrder(node.getRightChild());
}
```

### 后序遍历

> 后序遍历首先遍历左子树，然后遍历右子树，最后访问根节点，在遍历左、右子树时，仍然先遍历左子树，然后遍历右子树，最后遍历根节点。即：

若二叉树为空则结束返回，否则：

1. 后序遍历左子树
2. 后序遍历右子树
3. 访问根节点
```Java
/**
 * 后序遍历
 * @param node
 */
public void iterateLastOrder(BinaryTreeNode node){
    if (node == null){
        return;
    }
    iterateLastOrder(node.getLeftChild());
    iterateLastOrder(node.getRightChild());
    operate(node);
}
```

![示意图](/img/b-t-8.png)

## 二叉树的Java实现
二叉树的实现比普通树简单，因为它最多只有两个节点
```Java
public class BinaryTreeNode {
    private int mData;
    private BinaryTreeNode mLeftChild;
    private BinaryTreeNode mRightChild;

    public BinaryTreeNode(int mData, BinaryTreeNode mLeftChild, BinaryTreeNode mRightChild) {
        this.mData = mData;
        this.mLeftChild = mLeftChild;
        this.mRightChild = mRightChild;
    }


    public int getmData() {
        return mData;
    }

    public void setmData(int mData) {
        this.mData = mData;
    }

    public BinaryTreeNode getmLeftChild() {
        return mLeftChild;
    }

    public void setmLeftChild(BinaryTreeNode mLeftChild) {
        this.mLeftChild = mLeftChild;
    }

    public BinaryTreeNode getmRightChild() {
        return mRightChild;
    }

    public void setmRightChild(BinaryTreeNode mRightChild) {
        this.mRightChild = mRightChild;
    }
}
```
