---
title: "EyeGaze: 单眼遮挡下的注视方向估计"
published: 2026-11-24
description: "记录EyeGaze如何在遮挡下定位人脸、用专用模型细化眼部关键点，并用虹膜热图判断眼睛遮挡。"
tags: [眼动估计]
---

手挡住一只眼睛后，检测器仍可能在手上画出眼部关键点。EyeGaze 用虹膜热图判断哪只眼有足够的信号，再对通过筛选的眼睛估计相对于头部的注视方向。

[Github](https://github.com/Wa1NuQTQ/EyeGaze)

## 单眼遮挡结果

我们在 NASA 宇航员 Eileen Collins 的公开肖像上合成了一只手，遮住她的右眼。全文的左右按人物自身区分。[NASA 原始肖像](https://www.nasa.gov/image-detail/amf-s99-00858/)

![手遮住人物右眼，画面右侧的左眼仍然可见](/images/EyeGaze/single-eye-occlusion.jpg)

最终只有左眼输出了注视方向，右眼保留裁剪图并标记为遮挡。

## 人脸定位与眼部裁剪

早期测试中，我们自训练的人脸关键点检测器遇到手遮挡时，有时整张脸的定位都会失稳，进而影响头姿拟合和眼部裁剪，造成过度旋转以及眼睛注视方向的失准。MediaPipe在同类样本中能更稳定地定位鼻部等可见区域，也能为双眼给出大致位置。两套模型在遮挡侧都可能给出眼睑点，因此是否检测到坐标无法证明眼睛可见。[MediaPipe Face Landmarker](https://developers.google.com/edge/mediapipe/solutions/vision/face_landmarker)

![自训练检测器68点](/images/EyeGaze/occluded-custom-face.jpg)

![MediaPipe478点](/images/EyeGaze/occluded-mediapipe.jpg)

以两眼轮廓包围框的中心为裁剪中心，取两块同样大小的正方形，交给眼部模型分别处理。

![双眼裁剪图，左侧是被遮挡的右眼，右侧是可见的左眼](/images/EyeGaze/occluded-eye-crops.jpg)

## 大角度偏转时的眼部定位

眼球大幅偏转时，虹膜靠近眼角，有时只露出一小段边缘。图片中显示，MediaPipe 的虹膜轮廓越出了两条眼睑之间的可见区域。

![早期视频的单眼裁剪，粉紫色的 MediaPipe 虹膜轮廓越出可见眼部](/images/EyeGaze/mediapipe-extreme-gaze-eye.png)

我们继续用 MediaPipe 定位裁剪范围，再用自训练的眼部模型为每只眼预测 **19 个虹膜点**和 **24 个眼睑点**。下图用另一张公开的侧视照片[原论文图 1](https://journals.plos.org/plosone/article/figure?id=10.1371/journal.pone.0249137.g001)·[CC BY 许可](https://creativecommons.org/licenses/by/4.0/)对比两套模型。MediaPipe 的一个虹膜边缘点落到靠近眼白的位置，EyeGaze 的点在这个样本里更贴近可见的虹膜边界。

![公开侧视照片的右眼，依次为裁剪图、MediaPipe 和 EyeGaze，黄点标记眼睑，粉点标记虹膜](/images/EyeGaze/side-gaze-landmark-comparison.png)

## 虹膜热图与遮挡判定

眼部模型为每个关键点输出一张热图。即使裁剪里只有手指，热图里的弱响应取最大值也能得到坐标。我们先把眼图缩放至128×128，并在送入模型前水平翻转右眼，然后分三步判断。

1. 取每张虹膜热图的峰值，用19个峰值的中位数给该眼打分，减少个别亮点的影响。
2. 分数低于**0.4**时标记为遮挡，跳过关键点解码和眼球拟合。
3. 通过筛选的左眼在各热图峰值附近做加权平均，再将关键点坐标映射回原图。

![被遮挡的右眼仅有微弱热图响应，可见的左眼形成清晰的环状响应](/images/EyeGaze/occluded-iris-heatmaps.png)

## FLAME 拟合与注视方向

先用眼角、鼻部等锚点估计头姿，再根据二维重投影误差拟合 FLAME 的脸型和表情。大范围遮挡可能影响用于拟合的面部关键点和头姿。[FLAME 模型](https://flame.is.tue.mpg.de/)

![拟合后的 FLAME 脸部和双眼球，遮挡侧的右眼球仅展示模型几何](/images/EyeGaze/occluded-flame-mesh.png)

固定头姿后，调整左眼的水平角和俯仰角，使模型虹膜圆环的投影贴合检测到的虹膜点。圆环的朝向给出头部坐标系里的注视方向。这一帧跳过了右眼转角的拟合。

![双眼输出保留右眼的遮挡裁剪，左眼用黄点标记眼睑、粉点标记虹膜、青色箭头标记注视方向](/images/EyeGaze/occluded-eye-gaze.png)

## 应用

项目面向斜视筛查，目标是记录一只眼被遮挡时另一只眼的注视变化，为后续按预设规则开展筛查提供输入。
