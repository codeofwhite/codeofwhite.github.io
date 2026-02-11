---
title: "Pruning 例子"
date: 2025-02-11
categories: ["ai"]  
tags: ["LLM"]
draft: false
---

```text
LeNet(
  (conv1): Conv2d(1, 6, kernel_size=(3, 3), stride=(1, 1))
  (conv2): Conv2d(6, 16, kernel_size=(3, 3), stride=(1, 1))
  (fc1): Linear(in_features=400, out_features=120, bias=True)
  (fc2): Linear(in_features=120, out_features=84, bias=True)
  (fc3): Linear(in_features=84, out_features=10, bias=True)
)
```
在pytorch里面, prune通过对权重进行掩码来完成. 这个如何理解?

首先, 我们打印一下原始的conv1的权重看看:

```text
module = model.conv1
print(list(module.named_modules()))
print(list(module.named_buffers()))
print(list(module.named_parameters()))
```

这里列举了后面可能会用到三个方法, 这个可以查看当前的module到底是一个啥情况。

着重观察 `named_parameters` 因为参数都保存在这里, 打印完了之后可以看到:
```text
[('weight', Parameter containing:
tensor([[[[-0.2312,  0.2133, -0.1313],
          [-0.2980, -0.1838, -0.2902],
          [-0.3006,  0.1338, -0.0980]]],
        [[[-0.1239,  0.1060,  0.3271],
          [-0.0301, -0.0245,  0.0493],
          [-0.0160,  0.0397, -0.1242]]]], device='cuda:0', requires_grad=True)), ('bias', Parameter containing:
tensor([-0.2593, -0.0520,  0.0303,  0.0382, -0.0468, -0.1053], device='cuda:0',
       requires_grad=True))]
```

它有一个weight和一个bias, 这没错, 合乎常理. 我们甚至可以看看weights的尺寸是多少.
```text
for a in module.named_parameters():
    print(a[1].shape)
```

输出:
```text
torch.Size([6, 1, 3, 3])
torch.Size([6])
```

这个其实就是卷积的维度了, 6指的是channel, 1值得还是stride, 3指的是kernel size.
然后重点来了, 要开始做prune了, 在pytorch里面操作也很简单, 只需要一行代码:
```text
import torch.nn.utils.prune as prune
​
prune.random_unstructured(module, name='weight', amount=0.3)
​
```

这个可以从众多的剪枝方法中, 选择一个很好的手段来完成同样的目的. 然后我们再打印一下named_parameters:
```text
[('bias', Parameter containing:
tensor([-0.2281,  0.3085,  0.0937, -0.0540,  0.3295,  0.1107], device='cuda:0',
       requires_grad=True)), ('weight_orig', Parameter containing:
tensor([[[[ 0.1934, -0.0172, -0.1957],
          [ 0.1655,  0.1669, -0.2448],
          [-0.2250, -0.0963, -0.0195]]],
​
        [[[-0.3154,  0.1868,  0.0103],
          [-0.2245,  0.1548,  0.2567],
          [ 0.0713,  0.1262,  0.1547]]]], device='cuda:0', requires_grad=True))]
```
唯一的变化就是 `weights` 变成了 `weights_orig`, prune之后通过掩码的方式存放在了 `named_buffers`里面:

```text
print(list(module.named_buffers()))
```

可以看到:
```text
[('weight_mask', tensor([[[[1., 1., 1.],
          [0., 1., 0.],
          [0., 1., 1.]]],

        [[[0., 1., 0.],
          [0., 0., 1.],
          [0., 0., 1.]]]], device='cuda:0'))]
```
那么问题来了, 只是把权重进行了掩码, 那么我要知道你剪掉了哪几个channel怎么办? 而且你这个是剪的权重, 结构呢? 我怎么把这个结构找出来??

所以说这只是第一步, 接下来我们来看看结构化修剪. ==结构化修剪讲道理你可以知道你修剪了哪些结构==.

**一个比较好的结构化修建的例子是通过沿着Tensor的某个维度进行裁剪, 这样你可以直接看到维度的变化**. 

现在开始修剪模块, 比如上面的LeNet的conv1层, 首先我们可以从prune层里面拿一个我们喜欢的技术, 比如基于 **ln范数** 的评判标准来进行结构话的裁剪.

```text
prune.ln_structured(module, name='weight', amount=0.5, n=2, dim=0)
```

这个操作之后, 我们得到的将是一个新的权重, 和上面的非结构化的不同的地方在于, 这里是整个矩阵的一行为零, 上面我们用的dim=0, 那么就是channel这一个维度, 会有50%为零.

