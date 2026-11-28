---
title: "EyeGaze: 单眼遮挡下的注视方向估计"
published: 2026-11-19
description: "EyeGaze 从普通 RGB 照片或视频中估计每只可见眼睛的三维注视方向。单眼被遮挡时，EyeGaze 可以记录另一只眼的注视变化，为大规模斜视筛查提供量化的眼动数据。"
tags: [眼动估计]
---

EyeGaze 从普通 RGB 照片或视频中估计每只可见眼睛的三维注视方向。单眼被遮挡时，EyeGaze 可以记录另一只眼的注视变化，为大规模斜视筛查提供量化的眼动数据。

<nav class="project-resource-links no-heti" aria-label="EyeGaze 相关论文和源码">
  <a href="https://ai.nejm.org/doi/10.1056/AIoa2401205" target="_blank" rel="noopener noreferrer" aria-label="斜视测量论文（NEJM AI）" title="斜视测量论文（NEJM AI）">
    <span class="eyegaze-journal eyegaze-nejm" aria-hidden="true"><strong>NEJM<br>AI</strong></span>
  </a>
  <a href="https://jamanetwork.com/journals/jamaophthalmology/fullarticle/2854458" target="_blank" rel="noopener noreferrer" aria-label="上睑下垂评估论文（JAMA Ophthalmology）" title="上睑下垂评估论文（JAMA Ophthalmology）">
    <span class="eyegaze-journal eyegaze-jama" aria-hidden="true"><strong>JAMA</strong><small>Ophthalmology</small></span>
  </a>
  <a href="https://github.com/Wa1NuQTQ/EyeGaze" target="_blank" rel="noopener noreferrer" aria-label="EyeGaze GitHub 源码" title="EyeGaze GitHub 源码">
    <img class="project-github-mark" src="/images/github-mark.svg" width="40" height="40" alt="">
  </a>
</nav>

## 输入图片

我们在 NASA 宇航员 Eileen Collins 的公开肖像上合成了一只手，盖住她的右眼。全文按人物自身区分左右。 [NASA 原始肖像](https://www.nasa.gov/image-detail/amf-s99-00858/)

![画面左侧是被手遮住的右眼](/images/EyeGaze/single-eye-occlusion.jpg)

## 人脸定位与眼部裁剪

从这张单眼被遮挡的照片出发，我们先提取两类信息。一组人脸关键点用于拟合 FLAME 人脸并估计头部位姿，两张眼部裁剪图用于提取更精细的眼部关键点。

在早期测试中，手挡住眼周时，我们自训练的关键点检测器预测的被遮挡部分会出现较大误差。这类误差可能使拟合出的 FLAME 人脸出现异常旋转，进而影响注视方向的估计。

在下面的对照样本中，MediaPipe 在被遮挡的眼周给出的关键点偏移更小，因此我们用它提取人脸关键点。 [MediaPipe Face Landmarker](https://developers.google.com/edge/mediapipe/solutions/vision/face_landmarker)

![自训练检测器输出 68 个关键点，被遮挡区域的定位出现偏差](/images/EyeGaze/occluded-custom-face.jpg)

![MediaPipe 在同一张照片上输出 478 个人脸关键点](/images/EyeGaze/occluded-mediapipe.jpg)

我们根据每只眼的关键点确定轮廓包围框，以包围框中心为裁剪中心，从原图中分别裁出两块正方形眼部区域。MediaPipe 仍会为被遮挡的眼睛输出关键点，因此这一步保留两侧的裁剪结果，留到后续步骤判断遮挡。

![双眼裁剪图，左侧是被遮挡的右眼，右侧是可见的左眼](/images/EyeGaze/occluded-eye-crops.jpg)

## 大角度偏转时的虹膜定位

有了单眼裁剪图，下一步要找到能约束眼球拟合的虹膜关键点。眼球大幅偏转时，虹膜靠近眼角，可见边缘可能只剩一小段。在下面的样本中，MediaPipe 预测的虹膜点明显偏离了可见边缘，用这些点拟合眼球会降低注视方向的估计精度。

![大角度偏转时 MediaPipe 的虹膜关键点偏离了可见边缘](/images/EyeGaze/mediapipe-extreme-gaze-eye.png)

为此，我们用自训练的单眼模型在裁剪图上预测 **19 个虹膜点**和 **24 个眼睑点**。下面用同一张公开侧视照片对照两种模型。在这个样本中，MediaPipe 有一个虹膜边缘点落到靠近眼白的位置，EyeGaze 的点更贴近可见的虹膜边界。 [原论文图](https://journals.plos.org/plosone/article/figure?id=10.1371/journal.pone.0249137.g001) · [CC BY 许可](https://creativecommons.org/licenses/by/4.0/)

![公开侧视照片的右眼，依次为裁剪图、MediaPipe 和 EyeGaze，黄点标记眼睑，粉点标记虹膜](/images/EyeGaze/side-gaze-landmark-comparison.png)

## 用虹膜热图判断遮挡

自训练的单眼模型先为每个关键点预测一张热图，再从热图中解码出坐标。但即使裁剪图里只有手指，也能从热图的微弱响应中取到一个峰值位置。因此，我们在解码前检查虹膜热图的响应强度，据此排除被遮挡的眼睛。

1. 取 19 张虹膜热图各自的峰值，以这些峰值的中位数作为该眼的分数，减小个别亮点的影响。
2. 分数低于 **0.4** 时，将该眼标记为遮挡，并跳过关键点解码和眼球拟合。

![被遮挡的右眼只有微弱响应，可见的左眼呈现清晰的环状响应](/images/EyeGaze/occluded-iris-heatmaps.png)

## FLAME 拟合与注视方向

经过筛选，我们得到了整张脸的二维关键点和可见眼的虹膜点。虹膜在照片中的位置同时受头部朝向和眼球转动影响。我们希望从这些二维观测中估计眼睛相对于头部的三维注视方向。

我们先用眼角、鼻部等锚点估计头部位姿，再根据二维重投影误差拟合 FLAME 的脸型和表情。 [FLAME 模型](https://flame.is.tue.mpg.de/)

固定头部位姿后，只拟合通过筛选的眼睛。我们调整眼球的水平角和俯仰角，使模型虹膜圆环在原图上的投影贴合检测到的虹膜关键点。

![拟合后的 FLAME 脸部和双眼球网格](/images/EyeGaze/occluded-flame-mesh.png)

虹膜圆环的朝向给出头部坐标系中的三维注视方向。

![左眼绘制注视方向，右眼保留遮挡裁剪图](/images/EyeGaze/occluded-eye-gaze.png)

## 应用

斜视评估中的遮盖试验会遮挡一只眼，观察另一只眼的运动。EyeGaze 可以从这类视频中自动提取可见眼的注视方向变化，为后续依据预设规则开展大规模筛查提供眼动数据。

<style>
.eyegaze-journal {
  display: grid;
  place-content: center;
  justify-items: center;
  width: 52px;
  height: 42px;
  border-radius: 7px;
  color: #fff;
  text-align: center;
  line-height: 1;
}

.eyegaze-journal strong {
  font: bold 13px/1 Georgia, serif;
}

.eyegaze-nejm {
  background: #8a2835;
}

.eyegaze-jama {
  background: #245976;
}

.eyegaze-jama small {
  font: 6.5px/1.45 "Segoe UI", sans-serif;
}
</style>
