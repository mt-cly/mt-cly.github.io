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

## 算法流程
1. 根据image-level标签训练分类网络，之后将全局池化删去，将FC的权重和特征图相乘后相加。得到CAM的粗略定位结果。(细节参照CAM论文)
2. 根据CAM的识别结果创建AffinityNet的label，训练AffinityNet收敛。此时对于一张原始图像，我们可以通过步骤1的方法得到其粗略的语义信息，又可以通过训练好的AffinityNet得到该原始图像的的像素相关矩阵。
3. 对于原始图像，将其CAM的粗略语义通过像素相关矩阵进行随机游走，得到精细的语义信息。即最终的结果。

## AffinityNet 细节
论文中对AffinityNet的介绍较为概括，所以通过阅读源码了解了模型的基础框架。
AffinityNet基于基础CNN网络:VGG-16/ResNet-38。作者提到使用ResNet-38在PASCAL VOC12 val上能达到60.2%的mIoU(媲美强监督的FCN了)。

---

以VGG-16为例
VGG-16模型是经过5次(conv+maxpool)后接三个fc得到1000个识别输出。这里将pool4和pool5的stride设为1，使得conv4的特征图(8倍下采样)在之后的处理中都不被进一步下采样。
1. 将conv-pool_4的结果(512张特征图)接1x1卷积降维得到(64张特征图),conv-pool_5的结果(512张特征图)接1x1卷积核降维得到(128张特征图),conv-pool_5fc的结果$($将conv5的结果进行droupout和激活得到512张特征图$)$接1x1的卷积降维得到(256张特征图)。
2. 将得到的同样大小的原图8倍下采样的64张+128张+256张特征图concate，得到448张原图八倍下采样的特征图。
3. 设指定搜索半径$s=5$和某个像素点位置$x$，则对于特征图的某个位置像素记$x$，计算其与相邻的34个点的$x_{neb1}$,$x_{neb2}$...$x_{neb34}$的相似程度=$e^{-(x-x_{nebi})}$，对于每个临接对$(x\-x_{nebi})$在448张上分别计算得到448个结果，求448个差的平均值记做$(x-x_{neb1})$,$(x-x_{neb2})$...$(x-x_{neb34})$。
4. 所以设输出图像为nxm大小，经过步骤2得到了448张$\frac {n}{8}$ x $\frac {m}{8}$大小的特征图，经过步骤3得到一系列的点对及其亲和值。根据这些值填充到点的相关性矩阵affmat,affmat是一个$\frac {mn}{64}$ x $\frac {mn}{64}$大小的矩阵。比如$(x-x_{neb1})$的值就填充到affmat的$(x,x_{neb1})$和$(x_{neb1},x)$两个位置，对角线(x,x)都填1。所以affmat是一个稀疏矩阵，同时是一个对称矩阵。当然，得到了affmat之后要进行归一化处理。
5. 将先前CAM的结果(21张原图大小的特征图)经过8倍下采样以满足使用affmat的尺寸要求,之后将每张特征图各个点线性展开，这样就得到了一个$(21$ x $\frac {mn}{64})$ 的矩阵。之后将其与affmat进行矩阵运算$($代码中运行了8次，即CAM\*affmat\*affmat...\*affmat$)$。运算后得到的还是$(21$ x $\frac {mn}{64})$大小的矩阵，将其恢复成21张特征图，上采样8倍回原图大小。之后还是按照同样的方式，各个点取21张中置信度最高的类别作为判定类别。

## 根据CAM的结果生成AffinityNet的标签
将图像输入训练过的CAM网络$($作者也是基于VGG-16/resnet-38$)$得到了21张图像的分类$($对于图像中不存在的类别那么对应的特征图即使有星星点点的位置也全置为0，比如图中不存在id=2的car,那么car对应的特征图置为纯黑色，也没必要参与后面的制作标签工作了$)$。
1. 假设一张输入图像有car和chair，CAM得到了有用的两张原图大小的特征图记$fm_1,fm_2$，分别对应两个类别。
2. 基于以上的两张特征图，我们倾向于将在$fm_1,fm_2$中置信度都很低的点视为背景，但是有些点在$fm_1,fm_2$中置信度中规中矩，不高不低，对于这些区域我们是将其视为前景还是视为背景呢？作者给的定位是将这些点视为待定区域，不参与后续的制作Affinity的label。综上，给定一张图像，我们根据CAM将其划分为三个部分 $(前景(值设为classid)，背景(值设为0)，待定区域(值设为255))$
3. 那么怎么由$fm_1,fm_2$推断得到背景和待定区域呢？通过$(1-max(fm_1,fm_2))^{alpha}$得到背景特征图。$alpha$作者取了2个值:8/32。分别计算$(加CRF后场处理)$得到了2张背景特征图记做$lowalpha$和$highalpha$。由于$(1-max(fm_1,fm_2))$的值小于1，所以对于任一位置,$lowalpha$的值大于$highalpha$的值。对于背景上的某个位置$x$，如果$highalpha_x\lt lowalpha_x \lt threshold$，那$x$定义为背景。如果$highalpha_x\lt threshold\lt lowalpha_x$，则将$x$定义为待定区域。而如果$threshold \lt highalpha_x\lt lowalpha_x$，即$max(fm_{1x},fm_{2x})$较大，则标记为对应类别的前景。
4. 现在我们分析了输入图像，将其划分为了三个部分，现在就可以制作标签了。和论文中论述的方法一致。得到的标签可以分为四类$(同一前景类别-同一前景类别标签，背景-背景标签，不同类别间的标签，不需关心的标签)$。
5. 上一部分讨论中，AffinityNet接受输入$m$x$n$的图像，经过卷积8倍下采样得到448张$\frac {n}{8}$ x $\frac {m}{8}$大小的特征图，对于每个像素点有34个邻接对，计算其在448张特征图上的448个结果的均值。所以对于某种类型的邻接方式，计算每个x,可以得到一张亲和度评定图，图中的每个值都代表这个点与其对应邻接点的相似程度。一共34种邻接方式。就是34张亲和度评定图。
6. 步骤4得到了三类标签，可以理解为三类，每类对应34张亲和度评定图label。对于同一前景类别标签，计算$fg_{loss}=\frac{\sum_{i=0}^{33} label与predict的交叉熵}{同一前景类别标签数量}$，计算背景标签$bg_{loss}=\frac{\sum_{i=0}^{33} label与predict的交叉熵}{背景标签数量}$,计算不同类别之间的标签$neg_{loss}=\frac{\sum_{i=0}^{33} label与predict的交叉熵}{不同类别间标签数量}$。所以最终的$loss=\frac{bg_{loss}}{4} + \frac{fg_{loss}}{4} + \frac{neg_{loss}}{2}$。






