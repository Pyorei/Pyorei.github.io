---
title: ros serial的学习 # 标题
date: 2021-11-03T12:00:00+08:00 # 时间
lastmod: 2021-11-03T12:00:00+08:00

draft: False
author: "帕里亚"

categories: # 分类
- Blog
tags: # 标签
- ros
---

[这篇文章](https://blog.csdn.net/weixin_43046653/article/details/84334529)提到可以使用两种方法对Arduino开发板进行通信，一种是基于ros通讯机制的rosserial进行通信，一种是直接使用串口进行通信。两种方法各有优缺点，本文对两种方法进行了简单的整理和总结。

<!--more-->

# rosserial的使用

[这篇文章](https://blog.csdn.net/qq_38441692/article/details/97814194)较为详细的演示了如何使用rosserial arduino，首先需要安装rosserial arduino。

```
sudo apt-get install ros-(rosvision)-rosserial-arduino
sudo apt-get install ros-(rosvision)-rosserial
```

其次在Arduino库中安装ros_lib，需要在Arduino库目录下的sketchbook/libraries下执行以下命令。

```
rosrun rosserial_arduino make_libraries.py
```

随后就可以在arduino开发板上使用ROS库下的方法了，如创建发布者和订阅者等等。由于使用了ROS库下的方法，rosserial可以使得arduino发布TF树、自身传感器数值等信息，且不需要修改串口号等内容，即插即用。

# 串口的使用

[这篇文章](https://blog.csdn.net/u014695839/article/details/81209082)演示了如何使用串口直接与arduino相互通讯，串口的使用相对快捷简单些，且不需要对已经编写好的Arduino开发板进行修改。只需要了解串口信息的含义，便可以通过串口实现机械臂的控制。缺点就是需要对ROS系统的消息进行整理，转化为字符串传输给开发板。

ROS与Arduino之间的通信相对简单，更为重要的是如何获取、处理两者之间的信息。
