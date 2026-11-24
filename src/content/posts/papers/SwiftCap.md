---
title: "SwiftCap: Fast Facial Appearance Capture from Multiview Images"
published: 2026-11-19
description: "Reconstructing relightable 4K facial assets from eight casual photographs."
tags: [论文]
---

<figure>
  <img src="/images/SwiftCap/teaser.png" width="1500" height="398" alt="SwiftCap facial appearance capture overview" loading="eager" />
  <figcaption>A public WildCap example from eight input photographs to a reconstructed face, 4K material maps, and two relighting setups.</figcaption>
</figure>

## Abstract

Facial appearance capture from casual photographs remains costly because geometry reconstruction, registration and reflectance fitting are repeated for each capture. SwiftCap accelerates this workflow by reconstructing a face mesh and diffuse atlas from eight shared-expression photographs. Small correspondence errors can mix different surface regions across views and blur high-frequency texture. We use cross-view context and local attention to refine UV-to-image sampling coordinates before feature fusion and decoding. Cameras fitted to the first reconstruction reproject a common surface, supplying consistent UV correspondences and point maps for a second learned round. Five final texture updates, followed by separate material and super-resolution networks, produce 4K assets.

## Overview

<figure>
  <img src="/images/SwiftCap/video-pipeline.png" width="1260" height="528" alt="Input photographs and image priors feed query and key construction, UV correction, view fusion, and decoding. The first mesh provides geometry feedback for round two." loading="lazy" />
  <figcaption>The reconstruction pipeline from eight photographs of the same expression to a second geometry-guided pass.</figcaption>
</figure>

The first round aligns the views to predict a face mesh and diffuse atlas. The mesh gives the second round a common 3D surface to reproject into each camera. After the final texture updates, SwiftCap exports 4K diffuse, normal, and specular maps.

## Key Idea

### Q/K for UV Correspondence Refinement

Near the lip boundary, a small error in a sampling coordinate can mix lip and skin features across views. SwiftCap refines those coordinates before fusing the views.

RealDenseFace supplies an initial sampling coordinate for each UV texel in each view. The **query (Q)** describes the texel using its UV location and cross-view context. The **key (K)** describes a candidate image pixel using delit RGB, DINOv3 features, and VGGT geometry. Matching Q against nearby keys weights the candidate offsets and shifts the sampling coordinate in each photo while keeping the UV texel fixed.

<figure>
  <img src="/images/SwiftCap/video-uv-refinement.png" width="1004" height="518" alt="A close-up of the lip boundary showing candidate image positions and their attention weights" loading="lazy" />
  <figcaption>Larger orange markers represent candidate positions with higher weights in the mouth patch.</figcaption>
</figure>

The coarse pass searches a **9 × 9** grid at **12-pixel** spacing. The refinement pass searches a **5 × 5** grid at **6-pixel** spacing around the corrected coordinate. The decoder uses the fused features to predict geometry and diffuse texture.

### Geometry-Guided Second Round

SwiftCap fits one camera per image using the first-round mesh and the refined 2D correspondences. Reprojecting the same surface samples into each view updates the UV-to-image anchors, visibility masks, point maps, and camera rays.

<figure>
  <img src="/images/SwiftCap/video-geometry-guided.png" width="1119" height="503" alt="Round-one image correspondences beside the same surface samples reprojected from the first mesh" loading="lazy" />
  <figcaption>Left shows the first-round coordinates. Right shows mesh reprojections that become the starting coordinates for round two.</figcaption>
</figure>

Round two refines the correspondences and reconstructs the face using these geometry-guided inputs. SwiftCap then refits the cameras and uses five texture updates to refine the diffuse atlas before 4K material export.

## Pipeline Details

:::note[Work in progress]
These diagrams show the intermediate outputs for one input view.
:::

### Preprocessing

```mermaid
flowchart LR
    A@{ img: "/images/SwiftCap/00.png", label: "Input photo", h: 100, constraint: "on" };
    B@{ img: "/images/SwiftCap/00_landmarks.png", label: "Face detections", h: 100, constraint: "on" };
    C@{ img: "/images/SwiftCap/00_pose.png", label: "Head pose", h: 100, constraint: "on" };
    D@{ img: "/images/SwiftCap/00_face_mask.png", label: "Face mask", h: 100, constraint: "on" };
    E@{ img: "/images/SwiftCap/00_mask.png", label: "Crop mask", h: 100, constraint: "on" };
    F@{ img: "/images/SwiftCap/00_crop.png", label: "512px crop", h: 100, constraint: "on" };
    G@{ img: "/images/SwiftCap/00_crop_masked.png", label: "Masked crop", h: 100, constraint: "on" };
    A -->|RetinaFace| B
    B -->|SixDRep| C
    B -->|FARL/LAPA| D
    D --> E
    A --> F
    D --> F
    E --> G
    F --> G
```

### Pretrained Priors

```mermaid
flowchart LR
    A@{ img: "/images/SwiftCap/00_mask.png", label: "Crop mask", h: 100, constraint: "on" };
    B@{ img: "/images/SwiftCap/00_crop.png", label: "Cropped face", h: 100, constraint: "on" };
    C@{ img: "/images/SwiftCap/00_opendelight.png", label: "Delit", h: 100, constraint: "on" };
    D@{ img: "/images/SwiftCap/00_dino.png", label: "DINOv3 patch features", h: 100, constraint: "on" };
    E@{ img: "/images/SwiftCap/00_vggt.png", label: "VGGT world points", h: 100, constraint: "on" };
    F@{ img: "/images/SwiftCap/00_camera_rays.png", label: "VGGT camera rays", h: 100, constraint: "on" };
    G@{ img: "/images/SwiftCap/00_realdenseface.png", label: "RealDenseFace UV-to-Photo map", h: 100, constraint: "on" };
    B -->|OpenDelight| C
    A -->|face mask| C
    B -->|DINOv3| D
    B -->|VGGT| E
    B -->|VGGT| F
    B -->|RealDenseFace| G
```

## Results

<figure>
  <img src="/images/SwiftCap/in_the_wild.png" width="1600" height="1273" alt="Seven public WildCap captures with input photos, reconstructed geometry, and two relighting results per capture" loading="lazy" />
  <figcaption>Each WildCap row shows eight photographs, the reconstructed geometry, and key-and-fill and colored split lighting.</figcaption>
</figure>

SwiftCap achieved lower input-view LPIPS than the evaluated eight-view learned baselines on H3DS (**0.0601**), Multiface (**0.0441**), and NeRSemble (**0.0402**).

The complete workflow took **12.8 seconds** on an NVIDIA RTX PRO 6000 for the timed WildCap capture, from selecting views in the input video to exporting the mesh and 4K maps.

## Supplementary Video

<video controls playsinline preload="metadata" src="/videos/SwiftCap/supplementary.mp4" style="width:100%">
  Your browser does not support direct video playback.
</video>
