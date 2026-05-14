# Dance 操作的伸展树实现原理与对比

本文说明 `dance2.cpp` 如何用隐式下标伸展树（Splay）实现 `I / S / R / F` 四种操作，并对比 `dance.cpp` 的 `vector` 实现优势。

## 1. 核心建模：把序列映射为一棵“按位置有序”的树

`dance2.cpp` 不是按数值排序，而是按“中序遍历顺序 = 序列顺序”维护元素。

每个节点维护：
- `val`：舞者编号
- `sz`：子树大小（用于按排名找第 k 个）
- `rev`：区间翻转懒标记
- `ch[2] / fa`：左右儿子与父节点

并使用两个哨兵节点：
- `head_id`：序列最左侧前哨兵
- `tail_id`：序列最右侧后哨兵

哨兵的作用是让任意区间 `[l, r]` 都能统一写成：
1. 找到 `kth(l+1)`（区间左边界前一个）
2. 找到 `kth(r+3)`（区间右边界后一个）
3. 两次 `splay` 后，目标区间恰好落在固定位置 `right.ch[0]`

这样区间操作不需要写很多边界分支。

## 2. 基础能力

### 2.1 `kth(k)`
依据左子树大小 `sz` 在树上走，找到中序第 `k` 个节点。

### 2.2 `splay(x, goal)`
把节点 `x` 旋转到 `goal` 的子节点位置（`goal=0` 表示旋到根）。

### 2.3 懒翻转
`applyRev(x)`：交换左右儿子并翻转 `rev` 标记。
`pushDown(x)`：访问子节点前把 `rev` 下传，保证结构正确。

## 3. 四个操作如何实现

## 3.1 `I label pos`（插入）
代码入口：`insertAt(pos, value)`

步骤：
1. `left = kth(pos + 1)`，`right = kth(pos + 2)`
2. `splay(left, 0)`，再 `splay(right, left)`
3. 在 `right.ch[0]` 挂入新节点

复杂度：均摊 `O(log n)`。

对应 `dance.cpp`：`vector.insert(begin()+pos, label)`，最坏 `O(n)`（需要搬移后续元素）。

## 3.2 `S i j`（交换两个位置）
代码入口：`swapAt(i, j)`

步骤：
1. `a = kth(i + 2)`，`b = kth(j + 2)`（`+2` 因为有左哨兵）
2. 直接交换 `tr[a].val` 与 `tr[b].val`

复杂度：`O(log n)`（两次 `kth`）。

对应 `dance.cpp`：`swap(dancer[i], dancer[j])` 是 `O(1)`。该操作单独看 `vector` 更快，但不是瓶颈。

## 3.3 `R r`（整体循环右移）
代码入口：`rotateRight(r)`

利用恒等式：右移 `r` 位等价于
1. `reverse(0, n-1)`
2. `reverse(0, r-1)`
3. `reverse(r, n-1)`

在 Splay 中，每次 `reverseRange(l, r)` 都是一次区间懒翻转，均摊 `O(log n)`；三次仍是 `O(log n)`。

对应 `dance.cpp`：新建 `res` 再重排赋值，`O(n)` 时间且 `O(n)` 额外空间。

## 3.4 `F a b`（翻转区间，支持跨尾首）
代码入口：
- 非环形：`reverseRange(a, b)`
- 环形（`a > b`）：先旋转再翻转再旋回

### 非环形 `a <= b`
与上面区间模板一致：
1. 定位边界外侧两个节点
2. `splay` 后给中间子树打 `rev` 标记

复杂度：均摊 `O(log n)`。

### 环形 `a > b`
`dance2.cpp` 的处理：
1. 右移 `n-a`，使原 `a` 到头部
2. 对前缀长度 `len = n-a+b+1` 做一次 `reverseRange(0, len-1)`
3. 再右移 `a` 旋回

每步都是 `O(log n)`，总计仍 `O(log n)`。

对应 `dance.cpp`：
- 非环形 `reverse`：`O(k)`
- 环形需要拷贝 + reverse + 写回：`O(k)` 且有额外临时数组

## 4. 对比 `dance.cpp` 的主要优点

1. 总体复杂度更低
- `dance.cpp`：`I/R/F` 常见为线性，操作序列长时总体趋近 `O(mn)`。
- `dance2.cpp`：除个别常数外，主要操作均摊 `O(log n)`，总体接近 `O(m log n)`。

2. 大规模数据更稳定
- `vector` 方案在中间插入、整体旋转、长区间翻转时会频繁搬移大量元素。
- Splay 方案主要做树旋转与少量指针改动，不依赖大块内存搬移。

3. 区间操作统一
- 借助哨兵 + `splay`“夹区间” + `rev` 懒标记，`R/F` 都归约为少量标准原语。
- 逻辑复用高，复杂操作可组合（例如环形翻转 = 旋转 + 区间翻转 + 旋回）。

4. 空间使用更可控
- `dance.cpp` 的 `R`、环形 `F` 频繁构造临时 `vector`。
- `dance2.cpp` 基本在原树上原地修改，仅节点池常驻，额外空间主要为 `O(n)` 节点存储。

## 5. 结论

当操作数 `m` 大、且 `I/R/F` 占比较高时，`dance2.cpp` 的伸展树实现会明显优于 `dance.cpp` 的 `vector` 实现，尤其在时间复杂度和长序列操作的稳定性上优势显著。
