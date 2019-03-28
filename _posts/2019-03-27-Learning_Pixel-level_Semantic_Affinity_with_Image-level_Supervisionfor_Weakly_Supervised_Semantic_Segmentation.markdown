---
layout: post
date: 2019-03-27
author: cly
title: "论文笔记 Learning Pixel-level Semantic Affinity with Image-level Supervisionfor Weakly Supervised Semantic Segmentation"
subtitle: "弱监督学习"
catalog: true
tag:
  - weakly semantic segmantation
---

> paper	https://arxiv.org/pdf/1803.10464.pdf
> code	https://github.com/jiwoon-ahn/psa 

## 背景
作者提出了弱监督语义分割模型，本篇的弱监督是image-level的，与大多数image-level的语义分割模型出发思路一样，基于CAM获得初始物体定位后在进行后续处理。CAM能够获得大致的物体定位，但是图像中仍然存在很多待定的区域(对于每个类别，包括背景，都低置信度)。文章的亮点主要在于CAM之后的处理，提出了AffinityNet用于判别图像中任一像素点x与图像中的任一其他像素点y之间的亲和度(即属于同一类别的概率)。输入nxm大小的图像，输出(nxm x nxm)大小的相关性矩阵，之后以CAM的结果作为初始化，使用随机游走进行迭代更新，获得更加精细的物体轮廓。

## AffinityNet 细节
论文中对AffinityNet的介绍较为概括，所以通过阅读源码了解了模型的基础框架。
AffinityNet基于基础CNN网络:VGG-16/ResNet-38。作者提到使用ResNet-38在PASCAL VOC12 val上能达到60.2%的mIoU(媲美强监督的FCN了)。

---

以VGG-16为例
VGG-16模型是经过5次(conv+maxpool)后接三个fc得到1000个识别输出。这里将pool4和pool5的stride设为1，使得conv4的特征图(8倍下采样)在之后的处理中都不被进一步下采样。
1. 将conv-pool_4的结果(512张特征图)接1x1卷积降维得到(64张特征图),conv-pool_5的结果(512张特征图)接1x1卷积核降维得到(128张特征图),conv-pool_5fc的结果(将conv5的结果进行droupout和激活得到512张特征图)接1x1的卷积降维得到(256张特征图)。
2. 将得到的同样大小的原图8倍下采样的64张+128张+256张特征图concate，得到448张原图八倍下采样的特征图。
3. 设指定搜索半径s=5和某个像素点位置$$x$$，则对于特征图的某个位置像素记(x)，计算其与相邻的34个点的$x_{neb1}$,$x_{neb2}$...$x_{neb34}$的差，之后对于每个临接对$x-x_{nebi}$有448个结果，求平均值记做$x-x_{neb1}$,$x-x_{neb2}$...$x-x_{neb34}$。
4. 所以设输出图像为nxm大小，经过步骤2得到了448张$\frac {n}{8}x\frac {m}{8}$
$$x+y$$
dsdasas
random variables \$X\_1, X\_2, X\_3\$ frof
