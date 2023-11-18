# 关于YOLOv7的分析


关于YOLOv7的分析，此篇文章是在7月初编写，可能会与现有的源码有所出入，如在月末增加了关于head部分阴性参数的融合，但总体的出入不会太大。

<!--more-->

YOLOv6刚刚推出不久，YOLOv7就带着论文和源码来了，并且获得了YOLO社区各位大佬的认可，在速度和精度上的提升很大。和YOLO系列类似，YOLOv7也分别对边缘GPU、常规GPU和云端GPU做了几种不同深度和宽度的模型，论文的最后也与YOLOv6进行了比较，也是有较大的提升，但性能曲线仍然需要进行测试与更新完善。同样的，这篇文章我也将从网络结构和tricks的角度与YOLO系列进行对比分析，鉴于目前论文与源码有较大出入，此处我选择使用源码为主要的参考对象。此篇文章是在7月初编写，可能会与现有的源码有所出入，如在月末增加了关于head部分阴性参数的融合。

论文: [https://arxiv.org/abs/2207.02696](https://arxiv.org/abs/2207.02696)

源码: [https://github.com/WongKinYiu/yolov7](https://github.com/WongKinYiu/yolov7)

![YOLOv7](https://img-blog.csdnimg.cn/93ebf3d93d864b8392a793f2e79643ad.jpeg#pic_center)![](https://img-blog.csdnimg.cn/img_convert/cae7d8edf37a08019d3f11065860d40d.jpeg)![对比YOLOv6](https://img-blog.csdnimg.cn/7753c1e29a5d43b6ba28762a06ea7f69.png#pic_center)

## backbone & neck

在主干网络和neck上最大的区别是使用了E-ELAN代替原来的CSPDarknet53和设计了DownC代替普通卷积进行下采样，在结构方面与YOLOv5相差并不算大，以及参考CSP结构重新设计了SPP模块，最后也是输出三种尺寸的特征图和使用FPN-PAN进行特征融合与提取。在YOLOv7的模型中并没有使用论文中标准的E-ELAN结构，而是设计了简化的ELAN，但是在YOLOv7-E6E的yamI文件中有单独算子组成的结构配置。

在设计高效网络时，作者认为不仅可以从参数量、计算量和计算密度考虑，还可以分析输入和输出的信道比、架构的分支数和元素级操作对网络推理速度的影响。因此，作者提出了基于ELAN网络扩展的E-ELAN，采用了expand、shuffle、merge cardinality结构，使得在不破坏原始梯度路径的前提下，提高网络的学习能力。论文的思路是使用分组卷积来扩展计算模块的通道和基数，将相同的组参数和通道参数用于计算每一层中的所有模块(expand)；然后将每个模块计算出的特征图根据之前设定的分组数打乱成X组，再将它们连在一起(shuffle)；最后，添加X组特征将每个模块融合在一起(merge)。以上内容在目前的源码中还未更新，甚至ELAN的论文：**Designing network design strategies**[1]也尚未发表，因此仍然需要持续的关注。

![ELAN](https://img-blog.csdnimg.cn/20676da7bd83453e8f8dd8134fb823e7.png#pic_center)


DownC模块使用了三种最基本的结构，包括1x1和3×3两种卷积核和Maxpool，源码中作者将Maxpool分为两种，一种是k=2，s=2的MP，另一种是k=3，p=1，s=1的SP，而DownC使用的是MP方案。同样参考了多支路的方案，一条路使用MP进行特征图尺度的缩放，另一条使用s=2的3×3卷积进行尺寸缩放，最后在通道方向上进行concat合并。

![DownC](https://img-blog.csdnimg.cn/95acfc5f412b40b2b8bf9561e208ffed.jpeg#pic_center)


利用金字塔池化操作与CSP结构设计的SPPCSP来代替原有的SPP也是为了创造多支路的模块，由1×1和3×3卷积核以及SPP模块组成，CSP结构[2]的加入可以实现更丰富的梯度组合，同时减少计算量。此处的Maxpool为SP池化方案，分别取k = 5, 9,13，并令P = k // 2实现金字塔池化操作。

![SPPCSP](https://img-blog.csdnimg.cn/9aac8fd3005f4d6b877ab10e71411476.jpeg#pic_center)

## head

在head部分，YOLOv7加入了RepConv,RepConv在训练时有3个支路分别为1×1、3×3卷积和BN，而在模型部署时可以将3个支路的卷积和BN进行等效融合，形成VGG结构的3×3卷积，从而加速模型推理的速度。论文中也详细研究了RepConv在多支路网络中的效果，发现RepConv自带的多支路会削弱如CSP和残差网络的特征提取能力，因此选择了只有1×1、3x3卷积两支路的RepConvN结构在多支路网络中使用。RepConvN在源码中有体现，但在现有的YOLOv7网络中，并没有运用到多支路的结构。

![Repconv](https://img-blog.csdnimg.cn/3a17bd6c8e9c42e6b66055386c612c5b.png#pic_center)


此外，在head部分还引入了深度监督（Deep supervision）[3]和标签分类器（label assigner）等tricks，虽然添加了很多可训练的内容，但不会过多的增加网络的计算量和参数量，因此作者称之为Trainable bag-of-freebies。论文中还提到了YOLOR[4]中隐式知识结合卷积特征映射和乘法方式；并使用了EMA模型[5]作为最终的推理模型，利用多种tricks结合来提升推理速度和精度。

深度监督即在网络中添加额外的辅助头（auxiliary head），并以浅层网络权重的辅助损失为指导，用于辅助网络的训练；作者也将最终输出结果的检测头称为引导头（lead head)。不过YOLOv7中只使用了与YOLOR相似的IDetect作为检测头，作者将auxiliary head和lead head集成到了IAUXDetect中，在YOLOv7-E6E等较大的模型中有所展现。作者还认为，可以通过较浅的辅助头直接学习引导头已经学习到的东西，使得引导头可以更加专注于尚未学习到的残余信息。

标签分类器首先使用引导头的预测作为指导，生成从粗到细的层次标签，分别用于辅助头和引导头的学习。设计软标签的好处时使得引导头有较强的学习能力，且软标签更能代表源数据与目标之间的分布差异和相关性。此处，作者生成了两种标签，粗标签和细标签，其中细标签与引导头在标签分类器上生成的软标签相同，而粗标签是通过降低正样本分配的约束，允许更多的网络作为正目标。

![AUXhead](https://img-blog.csdnimg.cn/2bf2bd9ba5d14ba68aeeee7812100e32.png#pic_center)

## 总结

YOLOv7的作者是CSPNet、YOLOR等论文的作者之一，因此YOLOv7的文章包括代码都保留了他们先前的工作经验，尤其是在设计多支路的特征提取层和YOLOR的隐式知识检测头方面。源码中提供了CSPNet中提到的3种不同的多支路排列方式A、B、C，且对于RepConv也做出了相当多的实验和改进；也对YOLOR的隐式知识检测头进行了扩展，有I-Detect、I-AUXDetect和I-Bin三种。至于为什么RepConv删去identity支路在多支路模块中会有更好的表现结果，以及ELAN网络的设计思路和主要优势等问题，仍然需要论文 **Designing network design strategies** 发表后做出解释。

此外，源码还需要等待更新，不同模型和数据集上的实验也需要社区的人跟进，目前与论文仍有较大的出入，如深度监督和完整的E-ELAN并没有应用到边缘GPU的模型中，现在的网络设计训练速度较慢等问题。鉴于YOLOv7可以连接多种任务头进行训练和部署，也期待其在如分类、分割、3D姿态估计等方面的表现。

## Reference

1.Designing network design strategies
2.[CSPNet: A New Backbone that can Enhance Learning Capability of CNN](https://arxiv.org/abs/1911.11929)
3.[Deeply-Supervised Nets](https://arxiv.org/abs/1409.5185)
4.[You Only Learn One Representation: Unified Network for Multiple Tasks](https://arxiv.org/abs/2105.04206)
5.[Mean teachers are better role models: Weight-averaged consistency targets improve semi-supervised deep learning results](https://arxiv.org/abs/1703.01780)

