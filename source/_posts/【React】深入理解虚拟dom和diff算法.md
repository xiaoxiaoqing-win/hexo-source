---
title: 【React】深入理解虚拟dom和diff算法
top: false
cover: false
toc: true
mathjax: true
date: 2020-05-21 15:56:17
password:
summary: 
tags: 虚拟dom
categories: React
---

> 本篇文章同步发表在我的个人博客中：https://zhengyq.club

## 写在前面

在`React`中，`Virtual Dom`和`diff`的结合大大提高了渲染效率。`diff`算法由最初的`O(n^3)`复杂度变为了现在的`O(n)`，那么在这其中都做了哪些事情，本篇文章为你揭晓答案～

## 虚拟dom和diff

### 虚拟dom是什么？

`Virtual DOM`是一种编程概念。在这个概念里，`UI`以一种理想化的，或者说“虚拟的”表现形式被保存于内存中。在`React`中，`render`执行的结果得到的并不是真正的`DOM`节点，结果仅仅是轻量级的`JavaScript`对象。像这样：

```
<div class="box">
    <span>111</span>
</div>
```

上面这段代码会转换为这样的虚拟`DOM`结构

```
{
    tag: "div",
    props: {
        class: "box"
    },
    children: [
        "hello world!"
    ]
}
```

### diff算法又是什么？

`diff`算法，会对比新老虚拟`DOM`，记录下他们之间的变化，然后将变化的部分更新到视图上。其实之前的`diff`算法，是通过循环递归每个节点，然后进行对比，复杂程度为`O(n^3)`，`n`是树中节点的总数，这样性能是非常差的。

### dom-diff的原理

`diff`算法会比较前后虚拟`DOM`，从而得到`patches`(补丁)，然后与老`Virtual DOM`进行对比，将其应用在需要更新的地方，得到新的`Virtual DOM`，在网上有一张非常直观的图可以帮忙参考

![](https://tva1.sinaimg.cn/large/007S8ZIlly1gf04qwykaoj30zk0hc0vm.jpg)

来解释下这张图：现有一个真实的`DOM`，首先会映射为虚拟`DOM`，这个时候，我们删除了最后一个`p`节点和`son2`的节点，得到了新的一个虚拟`DOM`，新的`vdom`会和旧的`vdom`进行差异对比，得到了`pathes`对象，之后，对旧的真实`dom`进行操作，得到了新的`DOM`。

### diff的几种策略

- `Web UI`中`DOM`节点跨层级的移动操作特别少，可以忽略不计。

- 拥有相同类的两个组件将会生成相似的树形结构，拥有不同类的两个组件将会生成不同的树形结构。

- 对于同一层级的一组子节点，它们可以通过唯一`id`进行区分。

基于以上三个策略，`React`分别对`tree diff`、`component diff`以及`element diff`进行了算法优化。

#### tree diff

基于第一个策略，`react`只会对同一层次的节点进行比较，如下图中，只对颜色相同框内的`DOM`节点进行比较，当发现节点不存在时，就会删除整个节点及其子节点，不会再进行比较，这样就只需要遍历一次，就能完成对整个`DOM`树的比较

![](https://tva1.sinaimg.cn/large/007S8ZIlly1gf04rik74jj30zk0fgtai.jpg)

如果出现了`DOM`节点的跨层级的移动操作，`React diff`会怎样呢？

`React`只会简单的考虑同层级节点的位置变换，对于不同层级的节点，只有创建和删除操作。如果A节点整个被移动到D节点下，根节点发现子节点中A不见了，就会销毁A；然后D 发现自己多了一个子节点，就会创建新的子节点（包含其中属于自己的子节点）作为其子节点。`react diff`就会按照这样的次序执行：`craete a -> create b -> create c -> delete a`。这种跨层级的节点移动，并不会出现移动的情况，而是会有创建、删除这些操作。这种操作会影响到React的性能，所以React官方也并不建议进行这种操作。在开发组件时，保持稳定的dom结构会有助于性能的提升

![](https://tva1.sinaimg.cn/large/007S8ZIlly1gf04rrsjusj30zk0fgdgu.jpg)

#### component diff

`React`对于组件间的比较采取的策略也是简洁高效

- 如果是同一类型的组件，按照原策略继续比较虚拟dom树

- 如果不是，则将该组件判断为`dirty component`，从而替换整个组件下的所有子节点

- 对于同一类型的组件，有可能其`Virtual DOM`没有任何变化，如果能够确切的知道这点那可以节省大量的`diff`运算的时间，因此`React`允许用户通过`shouldComponentUpdate()`判断该组件是否需要进行`diff`

举个例子来说，当下图中`componentD`改变为`componentG`时，即使这两个`compoent`结构很相似，但是`react`会判断D和G并不是同类型组件，也就不会比较二者的结构了，而是直接删除了d，重新创建G及其子节点，这个时候会影响react的性能

![](https://tva1.sinaimg.cn/large/007S8ZIlly1gf04xjy5gwj30zk0fgt9y.jpg)

#### element diff

当节点处于同一层级时，`React diff`提供了三种节点操作：插入、移动和删除

- 插入：新的`component`类型不在老集合里 -> 全新的节点，需要对新节点执行插入操作

- 移动：在老集合里有新`component`类型，且`element`是可更新的类型，`generateComponentChildren`已调用`receiveComponent`，这种情况下`prevChild=nextChild`，就需要做移动操作，可以复用以前的`dom`节点

- 删除：老的`component`类型，在新集合中也有，但对应的`element`不同则不能直接复用和更新，需要执行删除操作，或者老`component`不在新集合里，也需要执行删除操作


举个🌰：看下图中，老集合中包含节点A、B、C、D，更新后的集合中包含节点B、A、D、C，此时进行新老集合差异化对比，发现B不等于A，则创建并插入了B至新集合，删除老集合A，以此类推。。。这样做很繁琐，因为这些都是相同的节点，只是位置发生了变化，针对这一现象，react提出了优化策略，允许开发者对同一层级的同组子节点，添加唯一key进行区分【**注意：这里就体现了key的作用～**】

![](https://tva1.sinaimg.cn/large/007S8ZIlly1gf04so01taj30rs0dw3zg.jpg)

上面我们叙述的情况是没`key`的情况，如果有`key`了(假设`key`为上图中每个节点对应的名字，例如节点A，对应`key`为A)，它就会这么对比：

`diff`会通过`key`发现新老集合中的节点是相同的节点，因此无需进行节点删除和创建，只需要将老集合中节点的位置进行移动，更新为新集合中节点的位置，此时`React`给出的`diff`结果为：B、D不做任何操作，A、C进行移动操作。

首先对新集合的节点进行循环遍历，通过唯一`key`可以判断新老集合中是否存在相同的节点，如果存在，则进行移动操作，但在移动前需要将当前节点在老集合中的位置与`lastIndex`进行比较，如果`节点当前的位置<lastIndex`，则进行节点移动操作，否则不执行该操作。这是一种顺序优化手段，`lastIndex`一直在更新，表示访问过的节点在老集合中最右的位置（即最大的位置），如果新集合中当前访问的节点比`lastIndex`大，说明当前访问节点在老集合中就比上一个节点位置靠后，则该节点不会影响其他节点的位置，因此不用添加到差异队列中，即不执行移动操作，只有当访问的节点比`lastIndex`小时，才需要进行移动操作

下面我们来说一下大致的过程（下面所说的`_mountIndex`是当前节点所在位置，`lastIndex`为参考位置）

1、从新集合中取到B，判断老集合中存在相同的节点，通过对比节点位置判读是否进行了移动操作，B在老集合中的位置为`_mountIndex=1`，此时`lastIndex = 0`，不满足`child._mountIndex < lastIndex`的条件，因此不对B进行移动操作；此时更新`lastIndex = Math.max(prevChild._mountIndex, lastIndex)`，[其中prevChild.mountIndex表示B在老集合中的位置]。`lastIndex=1`，并将B的位置更新为新集合中的位置`prevChild._mountIndex=nextIndex`，此时新集合中`B._mountIndex=0`，`nextIndex++`进入下一个节点的判断

2、从新集合中取得A，判断老集合中存在相同节点A，通过对比节点位置判断是否进行进行移动操作，A在老集合中的位置`A._mountIndex=0`，此时`lastIndex=1`，满足`child.mountIndex<lastIndex`的条件，因此对A进行移动操作`enqueueMove(this, child._mountIndex, toIndex)`，其中 `toIndex` 其实就是 `nextIndex`，表示 A 需要移动到的位置；更新 `lastIndex = Math.max(prevChild._mountIndex, lastIndex)`，则 `lastIndex ＝ 1`，并将 `A` 的位置更新为新集合中的位置 `prevChild._mountIndex = nextIndex`，此时新集合中`A._mountIndex = 1，nextIndex++` 进入下一个节点的判断。

3、从新集合中取到D，判断老集合中是否存在相同节点，通过对比位置判断是否进行移动操作，D在老集合中的位置`D._mountIndex=3`，此时`lastIndex=1`，不满足`child._mountIndex<lastIndex`的条件，因此不对D进行移动操作；更新`lastIndex=Math.max(prevChild._mountIndex, lastIndex)`，则`lastIndex=3`，并将D的位置更新为新集合中的位置`prevChild._mountIndex=nextIndex`，此时新集合中`D._mountIndex=2`，`nextIndex++`进入下一个节点的判断

4、C节点同理

**如果新老集合中，不只是存在位置互换的关系呢？React diff又如何进行操作？（下图中各节点的key为对应节点名称）**

![](https://tva1.sinaimg.cn/large/007S8ZIlly1gf04t3n66vj30rs0dwt9n.jpg)

1、从新集合中取到B，判断老集合中是否存在相同的节点，找到B在老集合中的位置`B._mountIndex=1`，此时`lastIndex=0`，因此不对B进行移动操作；更新`lastIndex=1`，并将B的位置更新为新集合中的位置`B._mountIndex=0`，nextIndex++进入下一个节点的判断

2、取到新集合中E节点，由于老集合中不存在相同的节点，则创建新节点E；更新`lastIndex=1`，并将E的位置更新为新集合中的位置，`nextIndex++`进入下个节点的判断。

3、取到新集合中C节点，老集合中存在相同节点，`C._mountIndex=2`，`lastIndex=1`，此时`C._mountIndex > lastIndex`，因此不对C进行移动操作；更新`lastIndex=2`，并将C的位置更新为新集合中的位置，nextIndex++进入下一个节点的判断。

4、取到新集合中A节点，老集合中存在相同的节点，`A._mountIndex=0`，`lastIndex=2`，此时`A._mountIndex < lastIndex`，对A进行移动操作；更新`lastIndex=2`，并将A的位置更新为新集合中的位置，`nextIndex++`进入下一个节点的判断。

5、当完成新集合中的所有节点的`diff`时，最后还需要对老集合进行循环遍历，判断是否存在新集合中没有但老集合中仍存在的节点，发现存在这样的节点D，因此删除节点D，到此`diff`全部完成。

## 总结

基于`diff`这样的策略，所以`react`建议我们用添加唯一`key`的方式来进行优化，这里面可以牵扯出来一个问题：

**如果用index作为key会有什么问题呢？**

`index`作为`key`，如果我们删除了一个节点，那么数组的后一项可能会前移，这个时候移动的节点和删除的节点就是相同的`key`了，在`react`中，如果`key`相同，就会视为相同的组件，但这两个组件并不是相同的，这就会出现一些我们不想看到的问题～所以`key`的值我们要考虑好再确定哦~

## 最后

本篇文章详细的叙述了虚拟`dom`和`diff`算法，我们要分清有`key`没`key`是如何进行对比的，也还要知道为什么时间复杂度得到了很大的提升等～如果文章有不对之处，还请大家指出～我们共同学习，共同进步～～～

如果你想更快的收到文章的推送，欢迎关注我的公众号「web前端日记」～

![](https://tva1.sinaimg.cn/large/007S8ZIlly1gf04tavlfaj30zk0bq0uo.jpg)

## 参考文章

- [React 源码剖析系列 － 不可思议的 react diff](https://zhuanlan.zhihu.com/p/20346379)

- [React源码分析与实现(三)：实操DOM Diff](https://github.com/Nealyang/PersonalBlog/issues/2)

