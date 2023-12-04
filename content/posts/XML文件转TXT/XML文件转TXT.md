---
title: XML文件转TXT # 标题
date: 2022-04-06T12:00:00+08:00 # 时间

draft: False
author: "帕里亚"

categories: # 分类
- Blog
tags: # 标签
- tool
---

一个简单的XML标签转TXT标签方法

<!--more-->

## XML文件转TXT

网络上有很多xml转txt的文章，不过有的xml文件不包括size信息，即图片本身的宽高，因此在前辈的基础上添加了图像size的读取，无需对xml文件进行处理，直接将图片的宽高读到数组中，这个方法的局限性在于标签和图片必须一一对应，考虑到数据集通常是规整的，因此无伤大雅。

```python
import xml.etree.ElementTree as ET
import os
import cv2
from tqdm import tqdm

classes = ["holothurian", "echinus", "scallop", "starfish"]  # 类别
xml_path = "xml标签文件夹路径"
txt_path = "txt标签存储路径"
image_path = "图像文件夹路径"


# 将原有的xmax,xmin,ymax,ymin换为x,y,w,h
def convert(size, box):
    dw = 1. / size[0]
    dh = 1. / size[1]
    x = (box[0] + box[1]) / 2.0
    y = (box[2] + box[3]) / 2.0
    w = box[1] - box[0]
    h = box[3] - box[2]
    x = x * dw
    w = w * dw
    y = y * dh
    h = h * dh
    return (x, y, w, h)


# 输入时图像和图像的宽高
def convert_annotation(image_id, width, hight):
    in_file = open(xml_path + '\\{}.xml'.format(image_id), encoding='UTF-8')
    out_file = open(txt_path + '\\{}.txt'.format(image_id), 'w')  # 生成txt格式文件
    tree = ET.parse(in_file)
    root = tree.getroot()
    size = root.find('size') # 此处是获取原图的宽高，便于后续的归一化操作
    if size is not None:
        w = int(size.find('width').text)
        h = int(size.find('height').text)
    else:
        w = width
        h = hight

    for obj in root.iter('object'):
        cls = obj.find('name').text
        # print(cls)
        if cls not in classes:
            print(cls)
            continue
        cls_id = classes.index(cls)
        xmlbox = obj.find('bndbox')
        b = (float(xmlbox.find('xmin').text), 
             float(xmlbox.find('xmax').text), 
             float(xmlbox.find('ymin').text),
             float(xmlbox.find('ymax').text))
        bb = convert((w, h), b)
        out_file.write(str(cls_id) + " " + " ".join([str(a) for a in bb]) + '\n')


# 此处获取图像宽高的数组，tqdm为处理的可视化
def image_size(path):
    image = os.listdir(path)
    w_l, h_l = [], []
    for i in tqdm(image):
        if i.endswith('jpg'):
            h_l.append(cv2.imread(os.path.join(path, i)).shape[0])
            w_l.append(cv2.imread(os.path.join(path, i)).shape[1])
    return w_l, h_l


# 遍历xml文件，将对应的宽高输入convert_annotation方法
if __name__  == "__main__":
    img_xmls = os.listdir(xml_path)
    w, h = image_size(image_path)
    i = 0
    for img_xml in img_xmls:
        label_name = img_xml.split('.')[0]
        print(label_name)
        convert_annotation(label_name, w[i], h[i])
        i += 1
```
