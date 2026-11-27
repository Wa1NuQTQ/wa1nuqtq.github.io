---
title: "SwiftCap: 多视角图像生成可重光照的人脸资产"
published: 2026-11-23
description: "SwiftCap 从日常拍摄的多视角照片中重建三维人脸，重建结果可以在不同光照下重新渲染，用作个人数字人的面部资产。"
tags: [人脸重建]
---

SwiftCap 从日常拍摄的多视角照片中重建三维人脸，重建结果可以在不同光照下重新渲染，用作个人数字人的面部资产。

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

这一阶段输入完整的多视角照片。后续先验阶段由于引用的第三方库强制使用 **512 × 512** 的分辨率，所以这一步我们需要把图片缩小到这个分辨率，但直接缩小整张照片会让背景占去一部分像素，使脸部的信息和细节更难保留。因此我们需要先找出脸部区域，进行一次裁剪，然后再缩小到 **512 × 512** 的分辨率。

### 人脸检测和头部朝向

先用 RetinaFace 在原图上找人脸，它给出检测框和五个关键点，分别是双眼、鼻尖和两个嘴角。SixDRep 根据检测区域估计头部旋转，图中用立方体标出朝向。

:::gallery
![RetinaFace 检测框和五个关键点](/images/SwiftCap/retinaface.png)

![SixDRep 头部朝向](/images/SwiftCap/head-orientation.png)
:::

### 语义分割和人脸掩码

检测框里还包含头发和背景，FaRL 把检测区域分成不同的语义类别。我们选出需要的脸部类别并合成同一张二值掩码，白色区域作为照片的有效区域。

:::gallery
![FaRL 人脸语义分割](/images/SwiftCap/face-parsing.png)

![二值人脸掩码](/images/SwiftCap/face-mask.png)
:::

### 正方形裁剪和缩放

用掩码的外接框确定裁剪中心，把短边扩展到和长边一样。再对照片和掩码使用同一个裁剪框，分别用线性插值和最近邻插值缩放到 **512 × 512**。

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

预处理给出了八张 **512 × 512** 的裁剪照片和对应的人脸掩码，但从八张照片直接学习出来几何和漫反射贴图是十分困难的事情，并且颜色里也混有拍摄时的光照。所以为了降低学习的成本，我们从这些裁剪图提取漫反射颜色、三维点云和相机射线，并建立从 FLAME UV 到各视角照片的初始对应来作为我们网络的输入。

### OpenDelight 漫反射图

我们先用掩码去掉背景，再用 OpenDelight 去除光照影响，输出用于后续融合的漫反射 RGB。

![侧面视角去光照后的漫反射图](/images/SwiftCap/opendelight-diffuse.png)

### RealDenseFace 的 UV 对应

RealDenseFace 为每个 UV 位置预测当前照片中的二维取样坐标，我们用深度筛选可见位置，再检查预测坐标是否落在人脸掩码内。

![RealDenseFace 对应点映射回原图坐标](/images/SwiftCap/rdf-grid-on-photo.png)

:::gallery
![UV 空间照片坐标图](/images/SwiftCap/rdf-photo-xy.png)

![UV 空间采样颜色图](/images/SwiftCap/rdf-uv-texture.png)
:::

### VGGT 的三维点和相机射线

VGGT 一次读取所有视角，预测相机参数和逐像素三维点，相机参数还用于计算每个像素的单位射线。

:::gallery
![视角 400953 · VGGT 点云可视化](/images/SwiftCap/vggt-400953-point-cloud.png)

![视角 400953 · 相机射线](/images/SwiftCap/vggt-400953-camera-rays.png)

![视角 400953 · VGGT 彩色点云](/images/SwiftCap/vggt-400953-point-cloud-rgb.png)
:::

得到的各视角点云已经共享一个坐标系，但由于不同样本之间点云的坐标系和尺度可能不同，所以为了降低学习难度，我们还需要让这套几何和 FLAME 模板的坐标系及尺度一致，于是我们用 RealDenseFace 的对应在多个视角中取出同一批 FLAME 锚点，估计共享的尺度、旋转和平移，将三维点对齐到 FLAME 模板。

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

经过先验阶段，我们有了八个视角的彩色点云，三维位置都已对齐到同一个 FLAME 坐标系，每个视角还带有 OpenDelight 的漫反射颜色、相机射线和人脸掩码，RealDenseFace（RDF）提供了初始对应，指出每个可见的 UV 位置应到当前照片的哪个像素取样。

我们现在的任务等同于把VGGT预测出来的点云及其采样得到的颜色融合成为一个表面，然后用融合后的点云生成固定 FLAME 拓扑的网格，同时得到漫反射贴图。

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

RDF 其实给出的对应能把各视角大致对齐，但局部仍有取样偏差。对于几何来说，轻微偏移通常只会带来较小的三维位置误差。但是对于漫反射贴图这种对偏移更敏感的来说。同一个 UV 位置可能在一张照片中取到唇缘，在另一张照片中取到旁边的皮肤。把两种颜色混在一起，清晰的唇线就会变宽、变模糊。

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

下图使用当前样本的八个视角，按 RDF 初始坐标采样 OpenDelight 的漫反射 RGB，再对每个 UV 位置的有效视角等权平均。黄框标出了下面放大的嘴部区域。

![直接按 RDF 对应融合的漫反射贴图 · 黄框标出嘴部对照区域](/images/SwiftCap/rdf-direct-diffuse.png)

:::gallery
![视角 401175 · RDF 采样的嘴部](/images/SwiftCap/rdf-mouth-401175.png)

![视角 401045 · RDF 采样的同一 UV 区域](/images/SwiftCap/rdf-mouth-401045.png)

![八视角直接融合 · 唇缘颜色过渡变宽](/images/SwiftCap/rdf-mouth-mean.png)
:::

这两个视角的唇缘落在略有偏差的 UV 位置，直接平均后，唇线和唇纹都变得更模糊。我们因此想到的方法是把 RDF 对应作为局部搜索的起点，先修正各视角的取样位置，再融合几何和颜色。

### UV 查询与照片特征

为了在照片中搜索与同一 UV 位置匹配的像素，网络分别在两个空间里编码特征。下方的编码、匹配和解码示意对应同一组八视角照片的初始化网络前向运行。

- **UV 空间的 Q**。将 UV 坐标和模板有效掩码输入 UV U-Net，得到查询特征。
- **照片空间的 K 和 V**。把每张照片的漫反射 RGB、FLAME XYZ、单位射线和掩码拼成 10 个通道。照片 U-Net 提取共享特征，再由两个独立的 1 × 1 卷积生成 K 和 V。

Q、K、V 的空间分辨率均为 **128 × 128**，各有 **32 个通道**。Q 和 K 用于计算匹配分数，V 提供匹配后汇入 UV 空间的特征。

![Q/K/V 编码 · UV 网格生成 Q，照片先验生成 K 和 V](/images/SwiftCap/qkv-feature-domains.png)

## 局部匹配与坐标修正

这一阶段已有每个 UV 位置的 Q、各视角的 K 和 V，以及 RDF 给出的初始取样坐标。初始坐标通常已落在目标五官附近，我们要解决的是局部错位。因此，每个 UV 查询以初始坐标为中心，在 **9 × 9** 的窗口内按 **4 像素** 间隔取候选，排除落在图外或人脸掩码以外的位置。

用 Q 和候选位置的 K 计算缩放点积，再对有效候选做 softmax。同一组权重用来汇总 V，也用来加权计算取样坐标的偏移。

将偏移上采样到 **512 × 512** 的 UV 网格，叠加到初始坐标上。随后用修正后的坐标重新采样每个视角的三维点和漫反射 RGB。

![局部匹配 · 候选特征和坐标偏移共享注意力权重](/images/SwiftCap/local-cross-attention.png)

## 多视角融合与 UV 解码

修正坐标后，各视角的特征、三维点和漫反射颜色都已采样到同一套 UV 布局。各视角对脸部的覆盖不同，同一位置的观测质量也可能有差异。这个阶段把有效观测融合起来，最终生成一张 UV 几何图和一张漫反射贴图。

### 多视角融合

我们在每个 UV 位置给有效视角打分，再沿视角维度归一化权重，用同一组权重融合特征、XYZ 和漫反射 RGB。融合结果另外带上有效视角比例和 UV 模板掩码，共 **40 个通道**。

![多视角融合 · 在每个 UV 位置按有效视角加权](/images/SwiftCap/view-fusion.png)

### UV 解码

融合后的特征先进入共享的多尺度编码器，再分到几何和漫反射两个解码器。最终得到两张 **512 × 512** 的图。几何图的每个有效位置保存三维坐标，漫反射图在同一个 UV 位置保存 RGB。

![UV 解码 · 两个解码器分别输出 XYZ 和漫反射 RGB](/images/SwiftCap/uv-decoding.png)

下面的初始化网络输出用于核对 UV 布局和有效区域。

:::gallery
![初始化网络输出的 UV XYZ，颜色编码三维坐标](/images/SwiftCap/decoded-geometry.png)

![初始化网络输出的 UV 漫反射 RGB](/images/SwiftCap/decoded-diffuse.png)
:::

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
