---
title: "2023 Pku Hpcgame Writeup"
date: 2023-01-25T19:19:17+08:00
featured: true
---

第一次接触hpc比赛, 好在赛题不算太难(~~有很多混分空间~~), 因此作为新手也玩的很开心.

<!--more-->

## 1A 欢迎参赛

签到题, 用 SCOW 文件管理下载即可

## 1B 实验室的新机器

`sbatch` 不如 `srun` 用起来方便

```shell
#!/bin/bash

srun -n1 -N1 -c1 -p compute -o output.dat ./program $1
seff "$(cat ./job_id.dat)" > seff.dat
```

## 1C

### Q-1

[Green500](https://www.top500.org/lists/green500/2022/11/) 的第一名

## 1D

题目刚上线时输入输出描述很含糊, 包括

1. 参数在 `stdin` 还是 `input.bin`
2. `N` 是否包括参数
3. 是 `mod 100001651` 意义下的加法, 还是求和结果 `mod 100001651`

因为理解错题意 `mmap()` 总是挂掉, 最后不用内存映射在错误的理解下勉强过了

## 2A 求积分

openmp 签到题, 可以用 `reduction(+ : sum_var)`

## 2B 乘一乘

把矩阵乘法最外层循环用 openmp 并行化就可以满分

## 2C 解方程

很奇怪, 写了一个单线程的 Gauss-Seidel 迭代也没有超时, 于是又开了 7 个线程空跑混到满分.

原始的 Gauss-Seidel 迭代[公式](https://zh.wikipedia.org/wiki/%E9%AB%98%E6%96%AF-%E8%B5%9B%E5%BE%B7%E5%B0%94%E8%BF%AD%E4%BB%A3)

$$
x_{m}^{k+1} = 
\frac {1}{a_{mm}} \left(b_{m} -
\sum_{j=1}^{m-1} a_{mj} \cdot x_{j}^{k+1} -
\sum_{j=m+1}^{n} a_{mj} \cdot x_{j}^{k} 
\right),
\ \ 1 \le m \le n.
$$

除了边界条件, 每个方程只涉及5个未知量, 这道题的矩阵比较稀疏, 迭代公式可以改写为

$$
u_{i,\ j}^{k+1} = 
\frac{1}{4} \left(
\frac{f(\frac{i}{n}, \frac{j}{n})}{n^2} +
u_{i-1,\ j}^{k+1} +u_{i,\ j-1}^{k+1} +
u_{i+1,\ j}^{k} +u_{i,\ j+1}^{k}
\right),
\ \ 1 \le i,\ j \le n-1
$$

$$
u_{ij}^{k+1} = 0,
\ \ i,\ j = 0, n
$$

伪代码

```python
u[n + 1][n + 1] all set to 0
for i in range(1, n):
    for j in range(1, n):
        y = f(i / n, j / n) / (n ** 2)
        u[i][j] = 0.25 * (y + u[i - 1][j] + u[i + 1][j]
                            + u[i][j - 1] + u[i][j + 1])
```

结果按题目要求的 eps 结束拿不到满分, 要减小到 1e-21 才可以, 不知道是我推的公式的问题还是不同的迭代顺序有影响

## 2D 道生一

被这道题卡了好久, 数据生成部分没想到什么并行化方法, 只能优化排序算法了, 依次尝试过

- 手搓并行化的归并排序 ( merge 无并行 )

- 手搓并行化的快排 ( partition 无并行 )

- rust rayon 的 `par_sort_unstable()`

- stl 的 `std::sort(std::execution::par_unseq, begin, end)`

- boost 的 `boost::sort::block_indirect_sort(begin, end)`

最后发现后三者性能差别不大, 似乎 boost 的略快, 但都没拿到满分

~~与其优化算法, 不如蹲凌晨测评~~

这道题题面后期更新过排序算法的提示, 不过我并不知道(~~知道也没有啥用~~), 可能思路出入比较大

## 2E

只是把卷积的最内层循环用 simd 指令改写了一下, 就拿到了绝大部分分, 果断选择见好就收 (谨防负优化)

## 2F

用题面的算法似乎是做不到满分的精度的, 这里用了  Bailey-Borwein-Plouffe 公式, 代码是从[这里](https://stackoverflow.com/a/4484573)搬运的, 收敛速度很快, 随便跑几次精度就符合要求了

## 3A

看起来很难, 交了个 handout 混了 30 分

## 3B

简单改写 handout,  加上几个 `cudaMalloc`,  `cudaMemcpy` 把数据拷贝到显存, 再分配给每条光线一个 thread (稍微调整下 block_size, grid_size), 计数光线时用 `atomicAdd()`. 这样就可以拿到基本分了

此时用 `ncu` 分析性能瓶颈, 发现核心占用和 cache 命中几乎都是 100%, 猜测要做算法优化, 然而也没什么思路, 遂拿个基本分跑路

## 3C

之前接触的深度学习基本为0, 炼丹过程不太顺利

### Section 1

首先老老实实的看 handout 尝试自己搭模型, 但正确率就没到过 80% (甚至一个模型不如一个), 于是开始 ctrl+c/v 编程.

在[这个仓库](https://github.com/akamaster/pytorch_resnet_cifar10/)找到了一个 CIFAR10 的模型, 用里面 resnet20 模型练了 100 多轮就能达到 90% 以上正确率了.

但是踩了一个坑, 搬运的模型 batch_size 是 128, 评测机似乎只接受 batch_size 64 的 result, 要修改下才能拿到 section 1 的分

### Section 2

在[这里](https://www.kaggle.com/code/rezasaatchi/plant-seedlings-classification-pytorch/notebook)发现了一个相同数据集的模型, 用了 pytorch 预训练的 resnet50, 搬运代码后, 训练不到 5 轮准确率就能达到 90% 以上

做题前没看到测试集存储在单独的路径下, 自己还特地从训练集里分出来的一部分当测试集, 白费了不少功夫 (pytorch 苦手只能面向 google 编程)

## 4A

扫雷题我的思路是从两方面优化, 主要是参考人做扫雷的决策

1. 性能上, 没必要每次迭代都遍历所有格子, 只需维护一个连通区域, 每次遍历连通区域的边缘, 因为只有这里可能扩展连通区域.

2. 准确性上, 通过周围的 8 个格子提供的信息估计中心格子为地雷的概率.
   
   1. 如果为 0 / 1, 则可以直接点击 / 标记为地雷.
   
   2. 如果为 0.x, 那也可以作为依据, 在没有绝对正确的选择的情况下择优点击
   
   3. 如果周围格子无法提供任何信息 (如全都未知), 那么暂时无法处理该中心格子

该算法的单线程版本可以在 N < 10000 的测试点拿到满分, 并行化也比较容易, 只需划分区域给多个线程, 每个线程在自己区域内不断扩展一个连通区域就可以, 此时划分的边界两侧要注意不能传播概率信息, 否则会有数据竞争问题 (影响正确性). 另外, 由于划分边界处概率信息的隔离, 可能会增加点到雷的数量, 不过似乎不影响拿到满分.

## 4B

最后一晚时做到这题, 交了 handout 就有 90+, 心满意足了已经是

## 4C

看了提示的优化小消息, 猜测就是要在发送 / 接收端增加缓冲, 合并发送 / 接收多个小消息, 不过想到之前已经混了不少分, 这题就不做了吧hh
