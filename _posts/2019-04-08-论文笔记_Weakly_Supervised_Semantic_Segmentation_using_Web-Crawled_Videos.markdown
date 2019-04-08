---
layout: post
date: 2019-04-08
author: cly
title: "论文笔记 Weakly Supervised Semantic Segmentation using Web-Crawled Videos"
subtitle: "弱监督学习"
catalog: true
tag:
  - weakly semantic segmantation
  - video
---

>paper: http://openaccess.thecvf.com/content_cvpr_2017/papers/Hong_Weakly_Supervised_Semantic_CVPR_2017_paper.pdf

### base
本文同时也是基于CAM进行弱监督语义分割。
本文模型采用的是类似强监督模型的SegNet的encode-decode结构。使用CAM的做法训练VGG并将其视为模型的encode部分(具备了良好的提取指定类别的能力)，而模型后半部分的decode阶段负责将分辨率较低的(encode结果/CAM结果)反卷积回原图分辨率大小的语义分割图。那么如何制作label以监督训练decode呢？这算是本文的核心部分之一，下面展开说明。本文的另一核心部分在于通过youtube的关键字搜索获得大量的对应label的视频，将其拆分为帧，用于训练decode部分，后面详述。
### generate labels of decode part
decode部分负责将低分辨率的图像恢复回原图大小的类别热力图。那么训练需要label，文中通过求解图模型的能量函数制作label。
1. 输入图像(来自video抽帧)，resize为不同大小，依次通过训练好的encode(由pacal voc data训练)得到若干张不同尺度的初始热力图，暴力resize回同样大小，maxpooling合成一张热力图。这一步骤是为了解决同一物体不同尺度的识别问题，获得更全面的热力信息(热力范围更大，在一定程度上缓解CAM的只聚焦最具辨识区域)。
2. 根据得到的热力图划分superpixel。每帧图像对应一张热力图(关键字搜索dog得到的视频，其抽出的帧使用encode得到的多个类别各自的热力图，这里只关注dog那张，忽略其他类别的热力图)。由于热力图分辨率较低，所以我们将原图划分为superpixel以让每个superpixel对应一个或多个热力图中的pixel,通过考虑pixel的值确定superpixel整个块的值(不知道有没理解错)
3. 创建图模型, 设计能量函数，对于一个视频，含有若干帧，每帧含有若干superpixel。
$$minE(L)=E_u(L)+E_p(L)$$
$E_u(L)$:元能量，表征判定结果与热力图识别结果的偏差程度。这里热力图的结果还进行了混合高斯模型的拟合以平滑热力图结果。这一部分的目的是为了最终结果的判定尽量接近热力图结果。
$E_p(L)$:对能量，表征判定两个物理上相邻的superpixel为不同结果的惩罚代价。这一能量函数是基于图模型的，同一帧图像上相邻的两个superpixel块连边的权重为他们的关联程度(颜色相似度+中心点距离)。相邻帧上的两个superpixel块连边的权重为他们的关联程度(光流法相似度+颜色相似度)。所以如果将两个有边相接的superpixel判定为不同的label(1/0)需要惩罚其连接边权重的值。这一部分的目的是为了最终结果的判定尽量倾向于将相近且相似的superpixel判定为同一个类。
4. 求解能量函数，得到一张非1即0的原图大小的label。用于训练decode。
### why get more data from  web-crawl?
爬取youtube视频，截帧，制作label，训练decode。这一系列操作的目的都是为了decode服务，但是为什么不直接将pascal voc的数据经过上章的类似处理用于训练decode呢？
1. 数据量更多，相较于pascal voc的数据，爬取的数据更多，而且更具鲁棒性(非同分布)。
2. 通过关键字获取到的视频能够保证单一类别，即pascal voc中一张图像往往包含了多个类别，作者认为如果用多个类别的热力图训练decode效果不佳，因为对于恢复dog信息时，cat也提供了多余的本该忽略的信息。所以如果一次训练中只需要考虑一个类别更有助于decode训练。
3. 视频具有序列关系，如果只用pascal voc的数据，那么能量函数那里的图模型的建立得修改，光流法等帧之间的关联关系部分得删去。












