> 译自：[ Demystifying Binary Search Tree Algorithms](https://about.sourcegraph.com/go/gophercon-2018-binary-search-tree-algorithms)。

# 揭开二分搜索树的神秘面纱

学习算法常常令人感到困惑和沮丧，但实际并不一定如此。在本次演讲中，*Kaylyn* 将使用Go代码以一种简单直接的方法来解释二分搜索树的算法。

## 介绍

*Kaylyn* 从去年开始愉快的算法学习之路，这对你来说可能觉得很奇怪，对她来说可能会更加怪异。她大学期间非常讨厌算法课程。她的教授会使用一些复杂的术语来解释一些“显而易见”的概念。结果，她只学到了足够的能够使她通过求职面试并继续生活的知识。

直到她开始在Go中实现一些算法，她对算法的看法才有所改变。将C或Java编写的算法转换成Go出奇地简单，这也使得她能够在大学之后还能这么好地理解这些算法。

*Kaylyn* 将解释为什么会有这样的感觉，并告诉你如何使用二分搜索树，但在我们这样介绍之前，我们必须先问下自己：为什么学习算法会感觉如此的糟糕？

## 学习算法太可怕了

![](https://user-images.githubusercontent.com/9022011/44757761-fdfd3100-aaed-11e8-8efb-bcac3d9aebb4.png)

此截图来自算法导论（也就是： CLRS）中关于二分搜索树的小节。CLRS被认为是算法界的“宝典”。据该书作者所说，在1989年出版之前，还没有很好的算法教科书。但是，任何读CLRS的人都会说它是由主要学院派的教授编写的。

举一些例子：

- 此页面引用了本书在其他地方定义的许​​多术语。所以你需要了解更多的概念，比如

  - 什么是卫星数据
  - 什么是链表
  - 什么是树的先序遍历和后序遍历

如果你之前没有阅读过这些内容，你根本不知道这些是什么。

- 如果你像 *Kaylyn* 一样，那么第一件事就是去书本中去寻找示例代码。但是，书本中唯一的代码解释了二分搜索树的方法，但是并没有告诉你二分搜索树真正的含义。

- 本页余下的部分都是一些定理和证明。这可能是善意而为之。许多教科书作者认为向你证明他们的陈述是真实性的是至关重要的；否则，你就不会相信他们。讽刺的是，CLRS本应该是一本入门算法教科书。但是，一个初学者事实上并不需要知道证明算法是否正确的各种细节。

- 值得称道的是，他们确实有一个两句话（以绿色高亮显示），解释了二分搜索树算法到底是什么。但它隐藏在一个几乎看不到的句子中，称之为二分搜索树的“属性”，这对于初学者而言是个令人困惑的属性。

结论：

1. 学院派教科书作者并不一定是好老师。好的老师通常也不写书。
2. 但大多数人都复制标准教科书使用的教学的代码风格和格式。在阅读二分搜索树之前，他们认为你应该知道的在线资源上的参考术语。实际上，大多数这些所谓的”必需知识”都不是必要的。

本次演讲接下来的部分将集中介绍什么是二分搜索树。如果你是Go新手或算法新手，你会发现它很有用。如果你不是，它仍然可以是一个很好的复习，你可以分享给其他对Go和算法感到好奇的人。

## 猜数游戏

这是你理解本次演讲唯一需要了解的内容。

![](https://user-images.githubusercontent.com/9022011/44758592-a01f1800-aaf2-11e8-9225-00c9d88ccaf9.png)

这就是“猜数游戏”，一个你小时候经常玩的游戏。你会要求你的朋友们猜出指定范围的一个数，比如1-100。你的朋友可能会选择一个数字，比如”57”。通常来说他们第一次猜测肯定是错的，因为范围太大了，但是你可以告诉他们所猜测的数高于或低于正确答案。然后他们继续猜测新的数字，通过你给予他们的提示，最终他们会猜测到正确的答案。

![](https://user-images.githubusercontent.com/9022011/44758764-7b777000-aaf3-11e8-92d4-ebb4e92c2832.png)

这个猜数游戏和我们将要介绍的二分搜索树算法过程类似。如果你理解了这个游戏，那么你应该也能理解二分搜索树背后的核心原理。他们猜测的数字就是树上的数字，并且“高于”/“低于”的提示是你需要移动的指令，是向左或向右移动到子节点的指令。

## 二分搜索树的规则

1. 每个节点都有一个唯一的键。你可以使用这个键和其他的节点做比较。一个键可以是任何可比较的类型：字符串，整型，等等。
2. 每个节点会有0个，1个或至多2个指向它的“子节点”。
3. 任意节点右边的键都大于该节点。
4. 任意节点左边的键都小于该节点。
5. 不能有重复的键。

二分搜索树有三个主要的操作：

- 搜索
- 插入
- 删除

在二分搜索树中上述三个操作非常的快 - 这也是为什么二分搜索树算法这么流行的原因。

### 搜索

![](https://cl.ly/dd19a7225c09/Screen%252520Recording%2525202018-08-29%252520at%25252009.03%252520PM.gif)

上面的GIF图片展示了如何在树中搜索”39”这个数字。

![](https://cl.ly/908ecf0f3854/Image%2525202018-08-29%252520at%2525209.31.02%252520PM.png)

需要注意的一点的是，你当前正在查看的节点的右侧的每个节点都大于当前节点的值。图中绿线右边的节点的值都大于“57”。绿线左边的节点的值都小于“57”。

![](https://cl.ly/61dfb3a92722/Image%2525202018-08-29%252520at%2525209.33.32%252520PM.png)

这个属性适用于树中的所有节点，并不只是根节点。在上面👆的图中，绿线右边的所有数字都大于”32”，红线左边的数字都小于“32”。

所以，目前为止我们知道了一些原理，那么让我们开始写代码吧。

```go
type Node struct {
  Key int
  Left *Node
  Right *Node
}
```

我们将使用一个结构体作为最基本的结构。如果你对结构体很陌生，那么可以将结构体理解成多个字段的集合。你需要的三个字段是`Key`（用来和其他节点比较），和`Left`、`Right`子节点。

当你想要定义一个节点时，你可以使用结构体字面量：

```go
tree := &Node{Key: 6}
```

这里我们创建了一个`Key`值为`6`的`Node`。你可能会好奇`Left`和`Right`节点在哪儿。事实上他们都被初始化成零值了。

```go
tree := &Node{
    Key:   6,
    Left:  nil,
    Right: nil,
}
```

你也可以这样初始化一个`Node`：

```go
tree := &Node{6, nil, nil}
```

在这个用例中，第一个参数是`Key`，第二个参数是`Left`，第三个参数是`Right`。

当你初始化`Node`之后，你可以像下面👇这样使用点号来访问结构体的字段：

```go
tree := &Node{6, nil, nil}
fmt.Println(tree.Key)
```

那么，先让我们写一个`Search`算法吧：

```go
func (n *Node) Search(key int) bool {
    // This is our base case. If n == nil, `key`
    // doesn't exist in our binary search tree.
    if n == nil {
        return false
    }

    if n.Key < key { // move right
        return n.Right.Search(key)
    } else if n.Key > key { // move left
        return n.Left.Search(key)
    }

    // n.Key == key, we found it!
    return true
}
```

### 插入

![](https://cl.ly/aaa1f718d537/Screen%252520Recording%2525202018-08-29%252520at%25252010.17%252520PM.gif)

上面👆的GIF图片展示了如何将`81`插入到树中。插入操作和搜索操作很类似。我们想要找到`81`在树中对应的位置，那么我们应该使用`search`中一样的方法来遍历树，一旦我们发现一个空白点，我们就插入`81`。

```go
func (n *Node) Insert(key int) {
    if n.Key < key {
        if n.Right == nil { // we found an empty spot, done!
            n.Right = &Node{Key: key}
        } else { // look right
            n.Right.Insert(key)
        }
    } else if n.Key > key {
        if n.Left == nil { // we found an empty spot, done!
            n.Left = &Node{Key: key}
        } else { // look left
            n.Left.Insert(key)
        }
    }

   // n.Key == key, don't need to do anything
    return
}
```

### 删除

![](https://cl.ly/e261dd30e743/Screen%252520Recording%2525202018-08-29%252520at%25252010.33%252520PM.gif)

上面👆的GIF图片展示如何从树中删除`78`。首先我们像之前那样来搜索`78`，但这个例子中，我们可以直接将`85`和`57`连接起来，从而删掉`78`。

```go
func (n *Node) Delete(key int) *Node {
    // search for `key`
    if n.Key < key {
        n.Right = n.Right.Delete(key)
    } else if n.Key > key {
        n.Left = n.Left.Delete(key)
    // n.Key == `key`
    } else {
        if n.Left == nil { // just point to opposite node
            return n.Right
        } else if n.Right == nil { // just point to opposite node
            return n.Left
        }

        // if `n` has two children, you need to
        // find the next highest number that
        // should go in `n`'s position so that
        // the BST stays correct
        min := n.Right.Min()

        // we only update `n`'s key with min
        // instead of replacing n with the min
        // node so n's immediate children aren't orphaned
        n.Key = min
        n.Right = n.Right.Delete(min)
    }
    return n
}
```

### 最小值

![](https://cl.ly/9f703767f7c9/Image%2525202018-08-29%252520at%25252011.20.37%252520PM.png)

如果逐步地向左移动，你将会得到最小数（在这个用例中，最小值是`24`）。

```go
func (n *Node) Min() int {
    if n.Left == nil {
        return n.Key
    }
    return n.Left.Min()
}
```

### 最大值

![](https://cl.ly/6e4021ed62d9/Image%2525202018-08-29%252520at%25252011.22.20%252520PM.png)

```go
func (n *Node) Max() int {
    if n.Right == nil {
        return n.Key
    }
    return n.Right.Max()
}
```

如果你逐步向右移动，你将会得到最大数（在本用例中，最大数是`96`）。

### 测试

至此我们已经为二分搜索树算法写好了初始方法，那么现在让我们真正地测试下代码吧！测试对于 *Kaylyn* 来说是编程最有趣的部分： Go中的测试比其他语言（比如Python和C）中的测试更加简单明了。

```go
// You only need to import one library!
import "testing"

// This is called a test table. It's a way to easily
// specify tests while avoiding boilerplate.
// See https://github.com/golang/go/wiki/TableDrivenTests
var tests = []struct {
    input  int
    output bool
}{
    {6, true},
    {16, false},
    {3, true},
}

func TestSearch(t *testing.T) {
    //     6
    //    /
    //   3
    tree := &Node{Key: 6, Left: &Node{Key: 3}}

    for i, test := range tests {
        if res := tree.Search(test.input); res != test.output {
            t.Errorf("%d: got %v, expected %v", i, res, test.output)
        }
    }

}
```

你可以使用`go test`来运行这个测试。

Go将会运行你的测试，并且会打印出非常nice的结果来告诉你你的测试是否通过，错误信息是什么，以及你的测试运行了多长时间。

### 基准测试

仅仅有测试不够的，我们还需要基准测试！在Go中使用基准测试也十分简单。下面👇的基准测试代码可能是你需要的：

```go
import "testing"

func BenchmarkSearch(b *testing.B) {
    tree := &Node{Key: 6}

    for i := 0; i < b.N; i++ {
        tree.Search(6)
    }
}
```

`b.N`将会一遍遍的运行`tree.Search()`直到测试程序观察到`tree.Search()`有一个稳定的执行时间。

你可以使用下面👇的命令来运行基准测试：

```bash
$ go test -bench=

goos: darwin
goarch: amd64
pkg: github.com/kgibilterra/alGOrithms/bst
BenchmarkSearch-4       1000000000               2.84 ns/op
PASS
ok      github.com/kgibilterra/alGOrithms/bst   3.141s
```

你需要要关注的应该是下面👇这行：

```bash
BenchmarkSearch-4       1000000000               2.84 ns/op
```

这行表示的是你运行的测试函数的速度。对我们这个测试用例来说，每次执行`test.Search()`大概花费2.84纳秒。

因为运行基准测试很容易，所以你可以做诸如以下的实验：

- 如果二分搜索树过大/过深会发生什么？
- 如果更改了需要搜索的key会发生什么？

*Kaylyn* 发现基准测试特别有助于理解map和切片之间的性能特征。它比你从网上搜索的各种文档来的更加快速，直观。

### 二分搜索树的相关术语

在本次演讲的最后，我们来了解下二分搜索树的相关术语。如果你想学习更多关于二分搜索树的细节，了解这些术语对于你理解二分搜索树很有帮助。

**树高度**: 从树的根节点到叶子节点中最长路径的边树，这个数值决定了算法的速度。

![](https://cl.ly/705355d982d4/Image%2525202018-08-30%252520at%25252012.05.11%252520AM.png)

上面👆这个数的高度是`5`。

**节点深度**： 从根节点到节点的边数

`48`节点的深度的是`2`。

![](https://cl.ly/a0058d294af0/Image%2525202018-08-30%252520at%25252012.08.04%252520AM.png)

**满二叉树**： 每个节点恰好有0个或2个子节点。

![](https://cl.ly/3bd94a056d8d/Image%2525202018-08-30%252520at%25252012.10.53%252520AM.png)

**完全二叉树**：每层节点都完全填满，在最后一层上如果不是满的，则只缺少右边的若干节点。

![](https://cl.ly/d78de1699704/Image%2525202018-08-30%252520at%25252012.12.03%252520AM.png)

**非平衡二叉树**：

![](https://cl.ly/1669851131fe/Screen%252520Recording%2525202018-08-30%252520at%25252012.14%252520AM.gif)

想象下如果你在这棵树中搜索`47`，你可以看到找到`47`需要花费7步，但是查找`24`只需要花费3步。这种问题会随着『不平衡』的增加而变得严重。解决办法就是让数变得平衡。

**平衡二叉树**：

![](https://img0.tuicool.com/aM3YneR.png)

此树拥有和上面👆非平衡二叉树相同的节点，但是在平衡二叉树上查找（平均耗时）要快于非平衡二叉树。
