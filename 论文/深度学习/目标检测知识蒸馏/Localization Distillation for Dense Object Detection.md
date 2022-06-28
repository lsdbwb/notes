# 原来方法的问题
 原来的目标检测的方法专注于使用feature imitation[[目标检测知识蒸馏技术综述#Feature Imitation]],较少使用logit Mimicking[[目标检测知识蒸馏技术综述#Logit Mimicking]]，因为其对目标检测中定位知识的信息传递效果很差。（logit主要是目标的分类信息）

# 提出的新方法
1. 重新设计了目标检测中对定位信息进行蒸馏的过程，提出了**LD**(localization distilation,定位蒸馏)方法。

2. 引入**VLR**（valuable localization region，有价值的定位区域）的概念，帮助划分知识蒸馏的区域

提出了一个分而治之的蒸馏方法，根据VLR将蒸馏区域进行划分，在对分类有利的区域进行分类蒸馏，对定位有利的区域进行定位蒸馏。对两者都有利的区域仍然进行Feature Imitation。


#  LD : localization distilation
将Logit Mimicking应用于传递定位知识（将原来的bbox的每一条边变成类似分类任务中的一组logits，然后建模出每一条边的分布，之后对bbox的KD就和对分类的KD类似了）

## bbox分布与定位模糊性
说起 LD，就不得不说起 bbox 分布建模，它主要来源于 GFocalV1（NeurIPS 2020）[1] 与 Offset-bin（CVPR 2020）[2] 这两篇论文。

我们知道 bbox 的表示通常是 4 个数值，一种如 FCOS 中的点到上下左右四条边的距离（tblr），还有一种是 anchor-based 检测器中所用的偏移量，即 anchor box 到 GT box 的映射（encoded xywh）。

GFocalV1 针对 tblr 形式的 bbox 建模出了 bbox 分布，Offset-bin 则是针对 encoded xywh 形式建模出了 bbox 分布，它们共同之处就在于尝试**将 bbox 回归看成一个分类问题**。并且这带来的好处是可以建模出 bbox 的*定位模糊性*
![[Pasted image 20220628104342.png]]

那么用 n 个概率值去描述一条边，可以显示出模型对一个位置的定位模糊估计，*越尖锐的分布说明这个位置几乎没有模糊性*（比如大象的上边界），*越平坦的分布说明这个位置有很强的模糊性*（大象的下边界）。当然不光是 bbox 分布的平坦度，形状上还可分为单峰型，双峰型，甚至多峰型。

# **VLR**（valuable localization region）
划分区域：对传递分类信息友好的区域、对传递定位信息友好的区域

![[Pasted image 20220628105346.png]]
1. Main distillation region :
2. Valuable localization region:


# 整体蒸馏过程
![[Pasted image 20220628111846.png]]