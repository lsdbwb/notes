# COCO数据集labels
YoloPose的数据集的lables是人体框+人体骨骼关键点




# 模型输出结果
可见是四个head，每个head包括Box和Keypoints
![[Pasted image 20220706114638.png]]

以输入图片大小为960的Yolov5m6_pose_960模型为例，模型输出的结果如下图所示：
![[YoloPose实现细节 2022-07-06 11.55.53.excalidraw]]

- 一张输入的图片 ：3 x 960 x 960
- 两个输出：
1. out : 57375 x 57
2. train_out : 3 x 120 x 120 x 57