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

那么用 *n 个概率值去描述一条边*，可以显示出模型对一个位置的定位模糊估计，*越尖锐的分布说明这个位置几乎没有模糊性*（比如大象的上边界），*越平坦的分布说明这个位置有很强的模糊性*（大象的下边界）。当然不光是 bbox 分布的平坦度，形状上还可分为单峰型，双峰型，甚至多峰型。

# **VLR**（valuable localization region）
划分区域：对传递分类信息友好的区域、对传递定位信息友好的区域

![[Pasted image 20220628105346.png]]
1. Main distillation region :即检测器的 positive location，通过 label assignment 获得。
2. Valuable localization region:与一般的 label assignment 做法类似，但区域更大，包含了 Main region，但去掉了 Main region。VLR 可以视为是 Main region 的向外扩张。

## VLR的选择
分类蒸馏KD和定位蒸馏LD作用在不同区域时的效果：
Main表示Main distillation region
VLR表示Valuable localization region
![[Pasted image 20220629095735.png]]
1. Main distillation region既包含有价值的分类信息，也包含有价值的定位信息。因此在Main distillation region进行KD或者LD蒸馏都有效，使用LD蒸馏效果更好一点。当然两者一起使用最好
2. VLR上使用LD也有效，但是当已经使用**Main KD + Main LD + VLR LD**之后（倒数第二行），再加上VLR KD反而效果会变差。


# 整体蒸馏过程
**损失函数如下：**
![[Pasted image 20220628111846.png]]
1. 前三个损失函数是任何基于回归的检测器都有的：
	Lcls是分类损失
	Lreg是bounding box回归损失
	LDFL是分布的focal loss
2. Imain和Ivl表示主蒸馏区域和VLA蒸馏区域的mask
3. KD表示分类蒸馏的损失。Cs表示的是student的分类logit，Ct表示teacher的分类logit。
4. LD表示定位蒸馏的损失。Bs表示student的bounding box 的logits,Bt表示teacher的bounding box的logits

# 实验结果
## 位置蒸馏LD和Feature Imitation的比较
![[Pasted image 20220629101226.png]]
表格上半部分是SOTA的Feature Imitation的几个方法的AP结果。表格下半部分是在这些方法的基础上加上本文提出的LD方法，可见AP都有较大提升

说明了 logit mimicking 多年以来的蒸馏效率低下的原因是**缺少有效的定位知识传递**。当引入了 LD 后，弥补了这一缺陷，logit mimicking 居然可以超过 Feature imitation。

最优的蒸馏策略依然是 logit mimicking 与 Feature imitation 都用上。只是可以注意到的是，在有了 logit mimicking 之后，各个 Feature imitation 的性能差异也不是很明显了，*选择哪个蒸馏区域都差不了多少*。

##  student 与 teacher 的平均分类误差与定位误差
![[Pasted image 20220629101239.png]]
1. 可以看到一些代表性 Feature imitation 方法（如 Fine-Grained，GI）确实可以同时降低分类误差与定位误差
2. 在只用上LD蒸馏时，定位误差得到明显下降，分类误差变化不明显；在同时加上KD后，两种误差都得到了明显下降

## 在两个 FPN 层级上的定位误差可视化
![[Pasted image 20220629101252.png]]
越暗说明student和teacher的误差越小