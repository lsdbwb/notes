# YOLO-Pose: Enhancing YOLO for Multi Person Pose Estimation Using Object Keypoint Similarity Loss

## 摘要
### 现有方法的问题
1. 现有的基于heatmap的方法不是端到端训练的，并且使用的L1损失函数和人体关键点检测的评估指标OKS并不对应
2. 现有的bottom-up方法在得到heatmap后还需要进行关键点分组
3. 现有的bottom-up方法为了提升准确率，会同时输出多分辨率的heatmap再进行融合，在测试时还会进行flip test,multi-scale test以提升准确率。这些操作都很耗时，对应用部署不友好。
4. 如果两个人在空间上十分接近，现有的基于heatmap方法很难将十分接近的关节点划分到正确的人上。
### 提出的方法
1. 提出了一个端到端模型，联合人体bounding box检测和人体关键点检测
2. 相比于传统bottom-up方法，不需要进行关键点分组
3. 相比于传统top-down方法，**一次推理**即可同时得出**多人**的bounding box和对应的人体关键点
4. 模型直接优化人体姿态估计评估指标OKS（Object Keypoints Similarity），即不仅使用OKS作为模型评估指标，还使用OKS作为损失函数来训练。
5. keypoints和anchors是联系在一起的，因此人和人之间的keypoints通过anchors分组，不需要使用复杂的后处理分组算法
### 方法的思想
1. 目标检测和人体关键点检测的困难是相似的：
	- 尺度变换问题
	- 遮挡问题
	- 人体结构多变问题
2. 如果有一个能解决上述问题的**人体**检测器，那么人体关键点检测也能顺势被解决。
## 实现细节
### 整体结构
![[Pasted image 20220419150609.png]]
1. 首先观察最终输出的head,每个head由box head和keypoints head组成
- box head : 
     一个人的bounding box坐标和置信度 (Cx, Cy, W,H, BOXconf, CLASSconf)
- keypoints head
	一个人的17个关键点的坐标和置信度。每个关键点（x,y,conf）
	一个head整体的预测向量为:
	![[Pasted image 20220419151233.png]]
 2. 使用CSP-darknet53作为backbone产生多尺度feature maps,然后送入PANet融合不同尺度的特征
 ### 损失函数
 1. bounding box损失函数：使用CIoU
 ![[Pasted image 20220419152251.png]]
 2. 人体关键点损失函数
 - heatmap是概率图，使用的L1 loss不能体现尺度变换
 - 只有使用基于回归的方法才能使用OKS作为损失函数，本论文**使用anchor为中心直接回归人体关键点**
	 1. 关键点损失函数
 ![[Pasted image 20220420104112.png]]
	2. 关键点置信度（visible flags）损失函数（一个人的不可见的关键点置信度为0，可见的关键点置信度为1）本模型和基于heatmap的方法不一样，对于一个人每次都会输出17个人体关键点结果，其中有些关键点事实上是不可见的，所以是无效的。
 ![[Pasted image 20220420104323.png]]
 3. 总的损失函数：
 ![[Pasted image 20220420104750.png]]
### 相比于Top-down方法的另一优点
![[Pasted image 20220420105015.png]]
如上图所示，本方法可以检测到超出人体检测框的关键点。而top-down方法只会在人体检测框之内进行检测，就会遗漏一些因为遮挡超出检测框的关键点。