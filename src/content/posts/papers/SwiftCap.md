---
title: "SwiftCap: 多视角图像生成可重光照的人脸资产"
published: 2026-11-23
description: "SwiftCap 从日常拍摄的多视角照片中重建三维人脸，重建结果可在不同光照下重新渲染，用作个人数字人的面部资产。"
tags: [人脸重建]
---

SwiftCap 从日常拍摄的多视角照片中重建三维人脸，重建结果可在不同光照下重新渲染，用作个人数字人的面部资产。

<nav class="project-resource-links no-heti" aria-label="SwiftCap 项目资源">
  <a href="https://github.com/Wa1NuQTQ/SwiftCap" target="_blank" rel="noopener noreferrer" aria-label="SwiftCap GitHub 源码" title="SwiftCap GitHub 源码">
    <img class="project-github-mark" src="/images/github-mark.svg" width="40" height="40" alt="">
  </a>
</nav>

## 输入

这组照片来自 Meta Reality Labs 的 Ava-256 数据集，对应样本 XIK500 的第 094990 帧。八台相机在同一时刻拍摄，每张原图的尺寸是 **1334 × 2048**。 [Ava-256 数据说明（CC BY-NC 4.0）](https://github.com/facebookresearch/ava-256/blob/main/DATASHEET.md)

下方展示其中三个视角。后文的单视角结果均以相机 400953 为例。

:::gallery
![视角 400953 · 侧面](/images/SwiftCap/input-400953.png)

![视角 401175 · 正面](/images/SwiftCap/input-401175.png)

![视角 401045 · 俯视](/images/SwiftCap/input-401045.png)
:::

## 预处理

预处理的输入是完整的多视角照片，为适配所用的先验模型，所以我们需要统一采用 **512 × 512** 的分辨率的照片。直接缩小整张照片会让背景占去一部分像素，减少保留的脸部信息和细节。因此，我们先定位并裁剪人脸，再把裁剪结果缩放到目标尺寸。

### 人脸检测和头部朝向

先用 RetinaFace 在原图上定位人脸，得到检测框和五个关键点，分别对应双眼、鼻尖和两个嘴角。SixDRep 根据检测结果估计头部旋转。

:::gallery
![RetinaFace 检测框和五个关键点](/images/SwiftCap/retinaface.png)

![SixDRep 头部朝向](/images/SwiftCap/head-orientation.png)
:::

### 语义分割和人脸掩码

检测框里还包含头发和背景。FaRL 将检测区域的像素划分为不同的语义类别。我们把需要保留的脸部类别合成一张二值掩码，白色表示有效的脸部区域。

:::gallery
![FaRL 人脸语义分割](/images/SwiftCap/face-parsing.png)

![二值人脸掩码](/images/SwiftCap/face-mask.png)
:::

### 正方形裁剪和缩放

以掩码外接框的中心为裁剪中心，将短边延长至与长边等长，得到正方形裁剪框。照片和掩码共用这个裁剪框，再分别用线性插值和最近邻插值缩放到 **512 × 512**。

:::gallery
![裁剪后的 RGB 照片](/images/SwiftCap/face-crop-512.png)

![与裁剪照片对齐的人脸掩码](/images/SwiftCap/face-mask-512.png)
:::

### 预处理总流程

```mermaid
flowchart TD
    accTitle: 人脸预处理
    accDescr: 从原始照片检测并分割人脸，再用同一裁剪框生成 512 × 512 的照片和掩码。

    A@{ img: "/images/SwiftCap/input-400953.png", label: "原始照片", h: 94, constraint: "on" }
    B@{ img: "/images/SwiftCap/retinaface.png", label: "检测框与五个关键点", h: 94, constraint: "on" }
    C@{ img: "/images/SwiftCap/head-orientation.png", label: "头部朝向", h: 94, constraint: "on" }
    D@{ img: "/images/SwiftCap/face-parsing.png", label: "语义分割", h: 94, constraint: "on" }
    E@{ img: "/images/SwiftCap/face-mask.png", label: "原图人脸掩码", h: 94, constraint: "on" }
    F@{ img: "/images/SwiftCap/face-crop.png", label: "裁剪照片", h: 94, constraint: "on" }
    G@{ img: "/images/SwiftCap/face-crop-512.png", label: "512 × 512 照片", h: 94, constraint: "on" }
    H@{ img: "/images/SwiftCap/face-mask-512.png", label: "512 × 512 掩码", h: 94, constraint: "on" }

    A --> I["RetinaFace"]
    I --> B
    B --> J["SixDRep"]
    J --> C
    B --> K["FaRL"]
    K --> D
    D --> L["二值化"]
    L --> E
    A --> M["正方形裁剪"]
    E --> M
    M --> F
    F --> N["线性插值"]
    N --> G
    M --> O["最近邻插值"]
    O --> H
```

## 先验

预处理得到八张 **512 × 512** 的裁剪照片和对应的人脸掩码。照片中的颜色混有拍摄时的光照，并且直接从这些照片学习几何和漫反射贴图十分困难。为此，我们通过一些先验模型先提取出漫反射颜色、三维点云和相机射线，并建立从 FLAME UV 到各视角照片像素的初始对应，将这些信息一起送入重建网络。

### OpenDelight 漫反射图

我们先用掩码去掉背景，再用 OpenDelight 估计去光照后的漫反射 RGB，供后续融合使用。

![侧面视角去光照后的漫反射图](/images/SwiftCap/opendelight-diffuse.png)

### RealDenseFace 的 UV 对应

RealDenseFace 为每个 UV 位置预测它在当前照片中的二维取样坐标。我们根据深度筛选可见位置，再检查预测坐标是否落在人脸掩码内。

![RealDenseFace 对应点映射回原图坐标](/images/SwiftCap/rdf-grid-on-photo.png)

:::gallery
![UV 空间照片坐标图](/images/SwiftCap/rdf-photo-xy.png)

![UV 空间采样颜色图](/images/SwiftCap/rdf-uv-texture.png)
:::

### VGGT 的三维点和相机射线

VGGT 一次读取所有视角的照片，预测相机参数和逐像素三维点。我们再用相机参数计算每个像素的单位射线。

:::gallery
![视角 400953 · VGGT 点云可视化](/images/SwiftCap/vggt-400953-point-cloud.png)

![视角 400953 · 相机射线](/images/SwiftCap/vggt-400953-camera-rays.png)

![视角 400953 · VGGT 彩色点云](/images/SwiftCap/vggt-400953-point-cloud-rgb.png)
:::

同一样本的各视角点云共享一个坐标系，但不同样本的坐标系和尺度可能不同，这会增加网络的学习难度。为统一表示，我们借助 RealDenseFace 的对应，在各视角中找到同一批 FLAME 锚点的三维位置，再估计一组共享的缩放、旋转和平移，将点云对齐到 FLAME 模板的坐标系和尺度。

:::gallery
![对齐后的八视角 VGGT 点云](/images/SwiftCap/vggt-point-cloud.png)

![对齐后的八视角 VGGT 彩色点云](/images/SwiftCap/vggt-point-cloud-rgb.png)
:::

### 先验处理总流程

```mermaid
flowchart TD
    accTitle: 漫反射、UV 对应与几何先验
    accDescr: OpenDelight 输出漫反射图，RealDenseFace 提供 UV 对应，VGGT 的三维点和射线基于这些对应对齐到 FLAME。

    A@{ img: "/images/SwiftCap/face-crop-512.png", label: "多视角裁剪照片", h: 92, constraint: "on" }
    B@{ img: "/images/SwiftCap/face-mask-512.png", label: "裁剪人脸掩码", h: 92, constraint: "on" }
    C@{ img: "/images/SwiftCap/opendelight-diffuse.png", label: "漫反射 RGB", h: 92, constraint: "on" }
    D@{ img: "/images/SwiftCap/rdf-photo-xy.png", label: "UV 到照片坐标", h: 92, constraint: "on" }
    E@{ img: "/images/SwiftCap/vggt-xyz.png", label: "标准化 XYZ", h: 92, constraint: "on" }
    F@{ img: "/images/SwiftCap/vggt-camera-rays.png", label: "相机射线", h: 92, constraint: "on" }

    A --> G["OpenDelight"]
    B --> G
    G --> C
    A --> H["RealDenseFace"]
    B --> H
    H --> D
    A --> I["VGGT"]
    B --> I
    I --> J["基于锚点对齐 FLAME 模板"]
    D --> J
    J --> E
    J --> F
```

## Q/K/V 特征

经过先验阶段，我们有了八个视角的彩色点云，三维位置都已对齐到同一个 FLAME 坐标系。每个视角还提供 OpenDelight 的漫反射颜色、相机射线和人脸掩码。RealDenseFace（RDF）给出初始对应，指出每个可见的 UV 位置应在当前照片的哪个像素取样。

所以接下来我们的任务等同于把 VGGT 预测的三维点和采样得到的颜色融合为一个表面，再用融合后的点云构建固定 FLAME 拓扑的网格，同时生成漫反射贴图。

**八视角 VGGT 彩色点云**

:::gallery
![相机 401175 · VGGT 彩色点云](/images/SwiftCap/qkv-inputs/vggt-401175.png)

![相机 401292 · VGGT 彩色点云](/images/SwiftCap/qkv-inputs/vggt-401292.png)

![相机 400953 · VGGT 彩色点云](/images/SwiftCap/qkv-inputs/vggt-400953.png)

![相机 401466 · VGGT 彩色点云](/images/SwiftCap/qkv-inputs/vggt-401466.png)

![相机 401463 · VGGT 彩色点云](/images/SwiftCap/qkv-inputs/vggt-401463.png)

![相机 400981 · VGGT 彩色点云](/images/SwiftCap/qkv-inputs/vggt-400981.png)

![相机 400974 · VGGT 彩色点云](/images/SwiftCap/qkv-inputs/vggt-400974.png)

![相机 401045 · VGGT 彩色点云](/images/SwiftCap/qkv-inputs/vggt-401045.png)
:::

一个简单的方法就是使用 RDF 的对应来把点云和颜色变换到 UV 空间，但它提供的对应仍有比较大的偏差。对于几何来说，轻微偏移通常只会带来较小的三维位置误差。但漫反射贴图对这种偏移更敏感，同一个 UV 位置可能在一张照片中采到唇缘，在另一张照片中采到邻近的皮肤。平均这两种颜色后，原本清晰的唇线就会变宽、变模糊。

**八视角 RDF 照片坐标图**

:::gallery
![相机 401175 · RDF 照片坐标](/images/SwiftCap/qkv-inputs/rdf-401175.png)

![相机 401292 · RDF 照片坐标](/images/SwiftCap/qkv-inputs/rdf-401292.png)

![相机 400953 · RDF 照片坐标](/images/SwiftCap/rdf-photo-xy.png)

![相机 401466 · RDF 照片坐标](/images/SwiftCap/qkv-inputs/rdf-401466.png)

![相机 401463 · RDF 照片坐标](/images/SwiftCap/qkv-inputs/rdf-401463.png)

![相机 400981 · RDF 照片坐标](/images/SwiftCap/qkv-inputs/rdf-400981.png)

![相机 400974 · RDF 照片坐标](/images/SwiftCap/qkv-inputs/rdf-400974.png)

![相机 401045 · RDF 照片坐标](/images/SwiftCap/qkv-inputs/rdf-401045.png)
:::

我们按 RDF 的初始坐标采样八个视角的 OpenDelight 漫反射 RGB，再对每个 UV 位置的有效观测做等权平均。

![直接按 RDF 对应融合的漫反射贴图](/images/SwiftCap/rdf-direct-diffuse.png)

:::gallery
![视角 401175 · RDF 采样的嘴部](/images/SwiftCap/rdf-mouth-401175.png)

![视角 401045 · RDF 采样的同一 UV 区域](/images/SwiftCap/rdf-mouth-401045.png)

![八视角直接融合 · 唇缘颜色过渡变宽](/images/SwiftCap/rdf-mouth-mean.png)
:::

两张单视角图的唇缘在 UV 上略有错位，直接平均后，唇线和唇纹都变得更模糊。因此，我们把 RDF 对应作为局部搜索的起点，先修正各视角的取样位置，再融合几何和颜色。

### UV 查询与照片特征

为了找到与每个 UV 位置匹配的照片像素，网络分别在 UV 空间和照片空间编码特征。

- **UV 空间的 Q**。将 UV 坐标和模板有效掩码输入 UV U-Net，得到查询特征。
- **照片空间的 K 和 V**。把每张照片的漫反射 RGB、FLAME XYZ、单位射线和掩码拼成 10 个通道。照片 U-Net 提取共享特征，再由两个独立的 1 × 1 卷积生成 K 和 V。

Q、K、V 的空间分辨率均为 **128 × 128**，各有 **32 个通道**。Q 和 K 用于计算匹配分数，V 提供匹配后汇入 UV 空间的特征。

![Q/K/V 编码 · UV 网格生成 Q，照片先验生成 K 和 V](/images/SwiftCap/qkv-feature-domains.png)

## 局部匹配与坐标修正

这一阶段已经得到每个 UV 位置的 Q、各视角的 K 和 V，以及 RDF 给出的初始取样坐标。初始坐标通常落在目标五官附近，因此我们在局部搜索更合适的取样位置。对每个 UV 查询，我们以初始坐标为中心，按 **4 像素** 的间隔在横纵两个方向各取 9 个位置，得到 **9 × 9** 个候选点。落在图像或人脸掩码范围之外的点会被排除。

用 Q 和候选位置的 K 计算缩放点积，再对有效候选做 softmax。用同一组权重汇总 V 特征，并计算取样坐标的加权偏移。

将偏移上采样到 **512 × 512** 的 UV 网格，叠加到初始坐标上。随后用修正后的坐标重新采样每个视角的三维点和漫反射 RGB。

![局部匹配 · 候选特征和坐标偏移共享注意力权重](/images/SwiftCap/local-cross-attention.png)

## 多视角融合与 UV 解码

修正取样坐标后，各视角的特征、三维点和漫反射颜色都已映射到同一套 UV 布局。各视角的脸部覆盖范围不同，对同一位置的观测质量也可能有差异。我们将有效观测融合后，解码出一张 UV 几何图和一张漫反射贴图。

### 多视角融合

我们在每个 UV 位置给有效视角打分，沿视角维度归一化权重，再用同一组权重融合特征、XYZ 和漫反射 RGB。最后加入有效视角比例和 UV 模板掩码，得到共 **40 个通道** 的融合结果。

![多视角融合 · 在每个 UV 位置按有效视角加权](/images/SwiftCap/view-fusion.png)

### UV 解码

融合后的特征先经过共享的多尺度编码器，再分别送入几何解码器和漫反射解码器。两者各输出一张 **512 × 512** 的图。几何图在每个有效 UV 位置记录三维坐标，漫反射图在对应位置记录 RGB。

![UV 解码 · 两个解码器分别输出 XYZ 和漫反射 RGB](/images/SwiftCap/uv-decoding.png)

<style>
#post-content:has(img[src^="/images/SwiftCap/"]) .gallery-container {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(min(100%, 7.5rem), 1fr));
  align-items: start;
}

#post-content:has(img[src^="/images/SwiftCap/"]) .gallery-item {
  min-width: 0;
  max-width: none;
}

#post-content .gallery-container:has(img[src^="/images/SwiftCap/qkv-inputs/"]) {
  grid-template-columns: repeat(2, minmax(0, 1fr));
}

@media (min-width: 768px) {
  #post-content .gallery-container:has(img[src^="/images/SwiftCap/qkv-inputs/"]) {
    grid-template-columns: repeat(4, minmax(0, 1fr));
  }
}

#post-content img[src="/images/SwiftCap/rdf-grid-on-photo.png"] {
  max-height: 40rem;
}
</style>
