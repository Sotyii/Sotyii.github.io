---
layout: post
title: "[PivotPainter] Fracture"
author: "Tianyi Song"
categories: practice
tags: [unreal, pivot painter, fracture, houdini]
image: GameDev/2024-06-15-pivot-paint-fracture/PPF_01.gif
---

在寻找ue破坏chaos的替代方案时发现的pivot painter相关的实现  
pivotpainter是将做顶点动画的信息存在顶点色，uv中  
每一块的x轴向在顶点色，pivot坐标在uv2和uv3  
有了这些信息可以对每一快进行来附加顶点动画  
