## 前置知识

**OI 常用的可持久化平衡树** 一般就是 **可持久化无旋转 Treap** 所以推荐首先学习 [**无旋转 Treap**](./treap.md)。

## 思想/做法

对于非旋转 Treap，可通过 **Merge** 和 **Split** 操作过程中复制路径上经过的节点（一般在 **Split** 操作中复制，确保不影响以前的版本）就可完成可持久化。

对于旋转 Treap，在复制路径上经过的节点同时，还需复制受旋转影响的节点（若其已为这次操作中复制的节点，则无需再复制），对于一次旋转一般只影响两个节点，那么不会增加其时间复杂度。

上述方法一般被称为 path copying。

「一切可支持操作都可以通过 **Merge Split Newnode Build** 完成」，而 **Build** 操作只用于建造无需理会，**Newnode**（新建节点）就是用来可持久化的工具。

我们来观察一下 **Merge** 和 **Split**，我们会发现它们都是由上而下的操作！

因此我们完全可以 **参考线段树的可持久化操作** 对它进行可持久化。

## 可持久化操作

**可持久化** 是对 **数据结构** 的一种操作，即保留历史信息，使得在后面可以调用之前的历史版本。

对于 **可持久化线段树** 来说，每一次新建历史版本就是把 **沿途的修改路径** 复制出来

那么对可持久化 Treap（目前国内 OI 常用的版本）来说：

在复制一个节点 $X_{a}$（$X$ 节点的第 $a$ 个版本）的新版本 $X_{a+1}$（$X$ 节点的第 $a+1$ 个版本）以后：

-   如果某个儿子节点 $Y$ 不用修改信息，那么就把 $X_{a+1}$ 的指针直接指向 $Y_{a}$（$Y$ 节点的第 $a$ 个版本）即可。
-   反之，如果要修改 $Y$，那么就在 **递归到下层** 时 **新建**  $Y_{a+1}$（$Y$ 节点的第 $a+1$ 个版本）这个新节点用于 **存储新的信息**，同时把 $X_{a+1}$ 的指针指向 $Y_{a+1}$（$Y$ 节点的第 $a+1$ 个版本）。

## 可持久化

需要的东西：

-   一个 `struct` 数组 存 **每个节点** 的信息（一般叫做 `tree` 数组）；（当然写 **指针版** 平衡树的大佬就可以考虑不用这个数组了）

-   一个 **根节点数组**，存每个版本的*树根*，每次查询版本信息时就从 **根数组存的节点** 开始；

-   `split()` 分裂 **从树中分裂出两棵树**

-   `merge()` 合并 **把两棵树按照随机权值合并**

-   `newNode()` 新建一个节点

-   `build()` 建树

### Split

对于 **分裂操作**，每次分裂路径时 **新建节点** 指向分出来的路径，用 `std::pair` 存新分裂出来的两棵树的根。

`split(x,k)` 返回一个 `std::pair`;

表示把 $_x$ 为根的树的前 $k$ 个元素放在 **一棵树** 中，剩下的节点构成在另一棵树中，返回这两棵树的根（first 是第一棵树的根，second 是第二棵树的）。

-   如果 $x$ 的 **左子树** 的 $key \geq k$，那么 **直接递归进左子树**，把左子树分出来的第二颗树和当前的 $x$  **右子树** 合并。
-   否则递归 **右子树**。

```cpp
static std::pair<int, int> _split(int _x, int k) {
  if (_x == 0)
    return std::make_pair(0, 0);
  else {
    int _vs = ++_cnt;  // 新建节点（可持久化的精髓）
    _trp[_vs] = _trp[_x];
    std::pair<int, int> _y;
    if (_trp[_vs].key <= k) {
      _y = _split(_trp[_vs].leaf[1], k);
      _trp[_vs].leaf[1] = _y.first;
      _y.first = _vs;
    } else {
      _y = _split(_trp[_vs].leaf[0], k);
      _trp[_vs].leaf[0] = _y.second;
      _y.second = _vs;
    }
    _trp[_vs]._update();
    return _y;
  }
}
```

### Merge

`merge(x,y)` 返回 merge 出的树的根。

同样递归实现。如果 **x 的随机权值**>**y 的随机权值**，则 `merge(x_{rc},y)`，否则 `merge(x,y_{lc})`。

```cpp
static int _merge(int _x, int _y) {
  if (_x == 0 || _y == 0)
    return _x ^ _y;
  else {
    if (_trp[_x].fix < _trp[_y].fix) {
      _trp[_x].leaf[1] = _merge(_trp[_x].leaf[1], _y);
      _trp[_x]._update();
      return _x;
    } else {
      _trp[_y].leaf[0] = _merge(_x, _trp[_y].leaf[0]);
      _trp[_y]._update();
      return _y;
    }
  }
}
```

## 例题

???+ note "[洛谷 P3835【模版】可持久化平衡树](https://www.luogu.com.cn/problem/P3835)"
    你需要实现一个数据结构，要求提供如下操作（最开始时数据结构内无数据）：
    
    1.  插入 $x$ 数；
    2.  删除 $x$ 数（若有多个相同的数，应只删除一个，如果没有请忽略该操作）；
    3.  查询 $x$ 数的排名（排名定义为比当前数小的数的个数 + 1）；
    4.  查询排名为 $x$ 的数；
    5.  求 $x$ 的前驱（前驱定义为小于 $x$，且最大的数，如不存在输出 $-2\,147\,483\,647$）；
    6.  求 $x$ 的后继（后继定义为大于 $x$，且最小的数，如不存在输出 $2\,147\,483\,647$）。
    
    以上操作均基于某一个历史版本，同时生成一个新的版本（操作 3, 4, 5, 6 即保持原版本无变化）。而每个版本的编号则为操作的序号。特别地，最初的版本编号为 0。

就是 **普通平衡树** 一题的可持久化版，操作和该题类似。

只是使用了可持久化的 merge 和 split 操作。

## 推荐的练手题

1.  [「Luogu P3919」可持久化数组（模板题）](https://www.luogu.com.cn/problem/P3919)

2.  [「Codeforces 702F」T-shirt](http://codeforces.com/problemset/problem/702/F)

3.  [「Luogu P5055」可持久化文艺平衡树](https://www.luogu.com.cn/problem/P5055)
