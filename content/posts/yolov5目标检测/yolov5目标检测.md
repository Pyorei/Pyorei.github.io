---
title: YOLOv5目标检测 # 标题
date: 2022-03-12T12:00:00+08:00 # 时间

draft: False
author: "帕里亚"

categories: # 分类
- Blog
tags: # 标签
- 目标检测
---

yolov1到v4的论文在[这篇文章](https://blog.csdn.net/weixin_39524959/article/details/111136986)里比较详细，此处不对其网络做更深入的介绍，重点在于如何训练以及如何用训练好的模型做检测。以下内容参考了[源码](https://github.com/ultralytics/yolov5)提供的教程，是对此前工作的技术总结。

<!--more-->

yolov1到v4的论文在[这篇文章](https://blog.csdn.net/weixin_39524959/article/details/111136986)里比较详细，此处不对其网络做更深入的介绍，重点在于如何训练以及如何用训练好的模型做检测。以下内容参考了[源码](https://github.com/ultralytics/yolov5)提供的教程，是对此前工作的技术总结。

## yolov5的安装与配置

```linux
git clone https://github.com/ultralytics/yolov5  # 下载源码
cd yolov5
pip install -r requirements.txt  # 安装所需要的环境
```

yolov5需要python3.7以上的环境版本，如果需要使用gpu加速，还需要安装英伟达的cuda，具体流程可以参考[这篇文章](https://blog.csdn.net/beifangc/article/details/108967239)。

## yolov5自定义数据集训练

首先是数据集的建立，数据集由训练集、测试集的图像以及标签组成，当然可以手动编写标签，以矩形框为例，格式为’class x y w h‘，分别代表目标框的类别、归一化后的x坐标以及y坐标，以及目标框的宽和高。[官方](https://github.com/ultralytics/yolov5/wiki/Train-Custom-Data)使用的是Roboflow网页标签工具，国内推荐使用[Make Sense](https://www.makesense.ai/)作为替代，或者是[使用labelImg标注数据](https://www.cnblogs.com/StarZhai/p/11926610.html)。

图片

生成数据集标签后，需要建立数据集文件夹以及yaml配置文件，其中images与labels内部的文件夹名需要对应，用于存放训练集和测试集的图片与标签，内部的文件名也需要一一对应

```linux
#文件夹配置
E:mydata\
├─images
│  ├─test
|    ├─01.jpg
│  └─train
└─labels
    ├─test
      ├─01.txt
    └─train
```

此外需要建立一个yaml配置文件方便yolo源码的调用，其中包含了训练集与测试集的绝对路径，以及数据集的类别数和名称，配置文件存放在源码的data文件夹下。

```yaml
train: E:\mydata\images\train
val: E:\mydata\images\test
nc: 2
names: ['cat', 'dog']
```

然后就是对源码的train.py文件进行配置，可以使用ide进行参数配置，也可以在控制台手动输入各种参数。其中img代表输入图像尺寸，batch为处理图像数量，epochs为迭代次数（默认为300），data为之前编写的配置文件，weights为权重模型。其余还有许多参数，分别代表什么可以查看源码，也可以参考[这篇文章](https://blog.csdn.net/weixin_41990671/article/details/107300314)。

```
python train.py --img 640 --batch 16 --epochs 3 --data mydata.yaml --weights yolov5s.pt
```

训练的时候会遇到内存不足的情况，可以尝试将batch改小，即一次处理的图片数量，或是增加电脑的虚拟内存，再不行可以将train.py内dataloader的num_workers参数修改为0。训练完成后会将最好的一次模型和最后一次模型以及各类数据存放在run\train文件夹内，有可能会遇到300次迭代仍得不到好的结果的情况，此时可以将best.pt存放到yolov5的根目录下，并使用best.pt进行下一次训练。

# yolov5检测验证

detect.py为目标检测的执行文件，与train步骤类似，可以使用ide配置参数，也可以在控制台输入命令，其中weights为权重模型，source为检测对象，运行完成后会将结果存放在run\detect文件夹下。

```linux
python path/to/detect.py --weights yolov5s.pt --source 0			#摄像头编号
													   img.jpg		#图片
													   vid.mp4		#视频
													   path			#路径下所有图
													   path/*.jpg	#指定类型图片
```

还有许多其它参数可以调节，比如save-txt是保存文字结果，conf-thres为置信度阈值，iou-thres为NMS的iou阈值，具体可以参考[这篇文章](https://blog.csdn.net/Q1u1NG/article/details/108093525)。

## 展望

由于可以保存txt文件，包含了类别以及xywh和置信度这些信息，结合先前的摄像机标定等工作可以得出检测对象的三维坐标，便于后续运动规划等任务。或者对detect源文件进行补充，添加ros的通信功能，方便后续的开发。yolov5的内容非常的丰富，有许多地方值得深究和修改，比如对于重复的多目标识别的任务中表现欠佳，这也是后续我工作的重点方向。
