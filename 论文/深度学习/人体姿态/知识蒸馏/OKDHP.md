# Online Knowledge Distillation for Efficient Pose Estimation
## 论文整体结构
> [! 提出知识蒸馏的原因:]
> 现有人体姿态模型过大，对内存和计算资源要求高
> 知识蒸馏能利用已有的大模型产出精度不错的小模型

> [! 现有姿态知识蒸馏方法的问题]
> 现有的是离线方法
> 需要两阶段学习
> - 先预训练一个大的高精度的teacher模型
> - 然后使用teacher模型训练学生模型

> [! 提出在线姿态知识蒸馏方法]
> 只需要训练一个网络
> - 该网络是多分支的,每个分支生成一个heatmap
> - 使用Feature Assemble Unit(FAU)将多个heatmap聚合生成高质量heatmap
> - 高质量heatmap作为teacher反过来teach每一个分支
![[Pasted image 20220330110615.png]]

>[! 实验结果]
>![[Pasted image 20220330112648.png]]


## 论文细节
