# 非极大值抑制
- nms要解决的问题：
1. 模型对图片中的同一个目标可能预测出多个候选bbox，这些bbox都大于置信度阈值，不进行筛选的话都会输出
2. nms就是从同一个目标的这些候选框中选出最优的那个框，筛出次优的框
![[Pasted image 20220426110158.png]]
- non maximum suppression（NMS）算法处理流程：
1. 将所有的框按置信度从小到大排序，每次选出置信度最高的box
2. 将剩下的boxes和置信度最高的box分别算IoU，去掉IoU大于阈值的boxes
3. 重复上述1，2步骤直到候选框为空

- pytorch代码实现
```python
from torch import Tensor
import torch

def box_area(boxes: Tensor) -> Tensor:
    return (boxes[:, 2] - boxes[:, 0]) * (boxes[:, 3] - boxes[:, 1])
 
def box_iou(boxes1: Tensor, boxes2: Tensor) -> Tensor:
    area1 = box_area(boxes1)  # 每个框的面积 (N,)
    area2 = box_area(boxes2)  # (M,)
 
    lt = torch.max(boxes1[:, None, :2], boxes2[:, :2])  # [N,M,2] # N中一个和M个比较； 所以由N，M 个
    rb = torch.min(boxes1[:, None, 2:], boxes2[:, 2:])  # [N,M,2]
 
    wh = (rb - lt).clamp(min=0)  # [N,M,2]  #小于0的为0  clamp 钳；夹钳；
    inter = wh[:, :, 0] * wh[:, :, 1]  # [N,M]  
 
    iou = inter / (area1[:, None] + area2 - inter)
    return iou  # NxM， boxes1中每个框和boxes2中每个框的IoU值；
 
def nms(boxes: Tensor, scores: Tensor, iou_threshold: float):
    keep = []  # 最终保留的结果， 在boxes中对应的索引；
    idxs = scores.argsort()  # 值从小到大的 索引
    while idxs.numel() > 0:  # 循环直到null； numel()： 数组元素个数
        # 得分最大框对应的索引, 以及对应的坐标
        max_score_index = idxs[-1]
        max_score_box = boxes[max_score_index][None, :]  # [1, 4]
        keep.append(max_score_index)
        if idxs.size(0) == 1:  # 就剩余一个框了；
            break
        idxs = idxs[:-1]  # 将得分最大框 从索引中删除； 剩余索引对应的框 和 得分最大框 计算IoU；
        other_boxes = boxes[idxs]  # [?, 4]
        ious = box_iou(max_score_box, other_boxes)  # 一个框和其余框比较 1XM
        idxs = idxs[ious[0] <= iou_threshold]
 
    keep = idxs.new(keep)  # Tensor
    return keep
 
box =  torch.tensor([[2,3.1,7,5],[3,4,8,4.8],[4,4,5.6,7],[0.1,0,8,1]]) 
score = torch.tensor([0.5, 0.3, 0.2, 0.4])
output = nms(boxes=box, scores=score, iou_threshold=0.3)
print(output)
```