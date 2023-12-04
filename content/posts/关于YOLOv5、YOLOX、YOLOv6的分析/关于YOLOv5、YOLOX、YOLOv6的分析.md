---
title: 关于YOLOv5、YOLOX、YOLOv6的分析 # 标题
date: 2022-07-10T12:00:00+08:00 # 时间
lastmod: 2022-07-10T12:00:00+08:00

draft: False
author: "帕里亚"

categories: # 分类
- Blog
tags: # 标签
- 目标检测
---

美团的技术团队在最近提出了YOLOv6网络模型，美团在[技术文档](https://tech.meituan.com/2022/06/23/yolov6-a-fast-and-accurate-target-detection-framework-is-opening-source.html)中重点对比了前两代的YOLOv5和YOLOX，以及百度的PP-YOLOE，在对coco数据集的验证中，YOLOv6不仅识别速度更快，且准确度也更高，此次提升的效果巨大。此处，我将尽可能详细地分析YOLOv6于YOLOv5和YOLOX的区别。

<!--more-->

## 关于YOLOv5、YOLOX、YOLOv6的分析

美团的技术团队在最近提出了YOLOv6网络模型，美团在[技术文档](https://tech.meituan.com/2022/06/23/yolov6-a-fast-and-accurate-target-detection-framework-is-opening-source.html)中重点对比了前两代的YOLOv5和YOLOX，以及百度的PP-YOLOE，在对coco数据集的验证中，YOLOv6不仅识别速度更快，且准确度也更高，此次提升的效果巨大。此处，我将尽可能详细地分析YOLOv6于YOLOv5和YOLOX的区别。

![对比图1](https://p0.meituan.net/travelcube/6e04535a78a8e341615ceab2dd474b18144058.png)

![对比图2](https://p0.meituan.net/travelcube/ae8d801a76f96eee5ddfd40d13901b0d141917.png)

### YOLOv5：[https://github.com/ultralytics/yolov5](https://github.com/ultralytics/yolov5)
在2020年，YOLOv4发布不久，YOLOv5就开源面世，因此网络上对YOLOv5的分析已经非常全面，如江大白老师的分析[https://zhuanlan.zhihu.com/p/172121380](https://zhuanlan.zhihu.com/p/172121380)，但后期YOLOv5又将主干网络中的Focus层换成了标准卷积，作者的解释是模型可导出性的改进，因为现在正式支持YOLOv5在11个不同的后端进行推理，其中许多为新的Conv实现提供了更好的支持，而Focus层是为了更快的初始层启动和更小的mAP影响，但其结果受到硬件设施的影响较大，许多消费卡和一些企业卡（如T4）使用Focus层可以观察到更快的性能，但在其他卡（如V100/A100）的实施中卷积的表现会更好。

![YOLOv5-v6.1](https://user-images.githubusercontent.com/31005897/172404576-c260dcf9-76bb-4bc8-b6a9-f2d987792583.png)

在最新的6.0/6.1版本中（[官方文档](https://github.com/ultralytics/yolov5/issues/6998#41)），主干网络和neck中选用了SiLU激活函数，并且修改了SPP模块，选用了更加简洁的SPPF作为neck，避免了三次卷积核的设置，作者也对SPP和SPPF进行了比较，SPPF在不影响mAP的情况下可以获得更快的速度和更少的FLOPs。

![SPP和SPPF的比较](https://user-images.githubusercontent.com/26833433/129478335-b221347a-4a52-4173-b378-12d004d7c2cd.png)

在P6版本中的head仍然使用的YOLOv3的检测头,但增加了一个，一定程度上提升了网络的对不同大小目标物检测的精确度。此外，仍然使用了Mosaic、Copy paste等在线数据增强方法，也在训练过程中使用了多尺度训练（0.5-1.5x）、预热和余弦LR调度程序、EMA（指数移动平均线）等方法提升效果。

### YOLOX: [https://github.com/Megvii-BaseDetection/YOLOX](https://github.com/Megvii-BaseDetection/YOLOX)

2021年，旷视研究院提出了YOLOX，并发表了论文[YOLOX: Exceeding YOLO Series in 2021](https://arxiv.org/abs/2107.08430)[1]。在论文中，YOLOX所选择的基线是YOLOv3-SPP版本，论文的说法是YOLOv4和YOLOv5已经过优化了，此外由于计算资源有限，各种实际应用中软件支持不足，且YOLOv3仍然是业界使用最广泛的检测器之一。

考虑到YOLOX也出了许多版本，此处我选择YOLOX-S于YOLOv5s的网络结构进行直观的比较。因为其主干网络使用了Focus和CSPDarknet53，且neck部分也采用了FPN+PAN的结构，激活函数也是YOLOv5-v6.0使用的SiLU。关于YOLOX论文中提出的YOLOX-Darknet与YOLOv3的区别，在江大白老师的文章中会更加详细[https://zhuanlan.zhihu.com/p/397993315](https://zhuanlan.zhihu.com/p/397993315)。

![YOLOX-S](https://vymnfdbwon.cn-02.visual-paradigm.com/rest/diagrams/shares/diagram/a0736986-b6fe-4368-901b-8a41669b6094/preview?p=1)
YOLOX-S的主要改进是修改了head部分的检测头,引入了Decoupled head解耦头的结构，并使用了anchor free自由锚框的方法，此外还加入了multi positives多正态和SimOTA最优传输等方法完善了预选框的筛选。在训练的方法中也使用了了EMA权值更新、Cosine学习率机制等训练技巧，而在损失函数的部分，YOLOX分别使用IOU损失函数训练reg分支，BCE损失函数训练cls与obj分支。

1. Decoupled head[2]
YOLO系列自YOLOv3开始就没有大幅度更改过检测头相关的内容，而解耦头的设计已经在目标检测和目标分类的网络中被验证有效。解耦头有着分类任务和定位任务两个分支，而采用不同的分支进行运算，有利于效果的提升。由于多分支的加入,会导致网络的计算复杂度增高，YOLOX所提出的方案是先使用1×1的卷积进行降维，来加快运算速度和减小模型大小。

![解耦头示例图](https://img-blog.csdnimg.cn/6601b6f812c744e38c4046041eec3a05.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5pybfg==,size_20,color_FFFFFF,t_70,g_se,x_16)

2. anchor free[3]
YOLOv3,4,5都是使用anchor base的方法来提取候选框，而YOLOX中将原有的3个anchor候选框缩减至1个，即直接由每个location对box的4个参数进行预测。将每个对象的中心位置所在的location视为正样本，并且将每个对象分配到不同的FPN层中，使得每个对象只有一个location进行预测工作，这样的修改同样是减少了GFLOPs并加快了推理速度。

3. multi positives[3]
如果每个对象只有一个location进行预测，会导致正负样本不均衡，且会抑制一些高质量的预测，YOLOX借鉴了FCOS的中心采样策略，将对象中心附近的location也纳入正样本的计算之中，此时每个对象就有了多个正样本的分布。

4. SimOTA[4]
在旷视自己的OTA算法中，他们将标签分配任务看作网络内部的最优传输任务，即多个目标与多个候选框的相互匹配。不同于OTA的是，SimOTA将原本使用的Sinkhorn-Knopp算法替换，使用动态top-k策略去计算最优传输的问题。这样的方法不仅减少了模型的训练时间，而且避免了Sinkhorn-Knopp算法中的额外求解器超参数。

### MT-YOLOv6：[https://github.com/meituan/YOLOv6](https://github.com/meituan/YOLOv6)

不同于YOLOX对检测头的修改，YOLOv6对YOLO系列的backbone和neck进行了全新的设计，参考RepVGG[5]网络设计了RepBlock来替代CSPDarknet53模块，基于COCO数据集的实验结果表明基于RepBlock设计的backbone有着更强的表征能力和更高效的GPU利用率。

![Rep算子](https://p0.meituan.net/travelcube/9f7878c7872787f9b8706b28e5e7c611237315.png)

RepVGG结构是在训练时进行多分支拓扑，而在进行推理时等效融合为一个3×3的卷积，这样就可以利用多分支训练时高性能的优势，和单路模型推理时速度快、省内存的优点。虽然YOLOv6对backbone的模块做了比较大的修改，但网络的组成变化并不大，仍然是采用单独的RepConv进行图片尺寸的缩放，使用多层RepConv组成的Block进行特征的提取。

![backbone结构](https://p0.meituan.net/travelcube/8ec8337d37c2545b8fcf355625854802145939.png)
此外，YOLOv6中所有的激活函数都为ReLU，从而提升网络训练的速度，且使用transpose反卷积进行上采样，并将neck中的CSP模块也使用RepBlock进行替代(Rep-PAN)，但仍然保留FPN-PAN的结构，这样的修改使得其训练的收敛速度更快，推理的速度也更快。

![Rep-Pan](https://p0.meituan.net/travelcube/c37c23c37fd094e05e8cab924659a9d9199592.png)
在head中仍然沿用了YOLOX的解耦头设计，不过并未对各个检测头进行降维的操作，而是选择减少网络的深度来减少各个部分的内存占用。此外，在anchor free的锚框分配策略中也沿用了SimOTA等方法来提升训练速度。参考了SloU边界框回归损失函数来监督网络的学习，通过引入了所需回归之间的向量角度，重新定义了距离损失，有效降低了回归的自由度，加快网络收敛，进一步提升了回归精度。[6]

![decoupled head yolov6](https://p0.meituan.net/travelcube/9a0fd7ba30522ce3ed24822e51b0e1a8109432.png)

## 总结

随着网络模型向着轻量化的方向改进，YOLO系列融合了RetinaNet、FCOS、RepVGG网络中有效的方法，可以实现非常好的检测效果。关于RepVGG结构，我认为可以运用到各类网络中来加快推理的速度，主要难点是多分支融合方式，如果将原本使用的CSPDarknet53进行Rep融合，是否会得到比YOLOv6更好的结果？YOLOv6使用的tricks看起来都是为了小型网路专门设计，如果当网络深度和宽度进行拓展，ReLU激活函数导致神经元失活是否会导致检测效果的下降？此外，anchor free确实缩减了模型的内存占用，但检测的效果并不如anchor base的方法好，在同样的tricks下，仍然需要比较它们的检测速度和精度，从而选取一个性价比最高的模型进行部署。

## Reference

1.YOLOX: Exceeding YOLO Series in 2021
2.[Rethinking Classification and Localization for Object Detection](https://arxiv.org/pdf/1904.06493v4.pdf)
3.[Fcos: Fully convolutional one-stage object detection](https://arxiv.org/pdf/1904.01355.pdf)
4.[OTA: Optimal Transport Assignment for Object Detection](https://arxiv.org/abs/2103.14259)
5.[RepVGG: Making VGG-style ConvNets Great Again](https://arxiv.org/abs/2101.03697)
6.[SloU Loss: More Powerful Learning for Bounding Box Regression](https://arxiv.org/abs/2205.12740)