---
layout: post
title: "[Unreal] Bottle Water - Part 1"
author: "Tianyi Song"
categories: worklog
tags: [unreal,simulation]
image: 2024-09-26-bottle-water-part-1\BWP1_01.gif
---

Part1主要记录如何在瓶内绘制液体，以及液体的物理表现实现。Part2会补充一些细节表现的实现，液体的张力，液面和水体中的气泡，随机的液体微动等等。  

需求来源于项目想要做一个类似The Finals中一个火焰燃烧弹的效果  
[The Finals Pyro Grenade](https://www.youtube.com/shorts/8uAxd1UKoww)  
<iframe width="100%"  height="400" src="https://www.youtube.com/shorts/8uAxd1UKoww" frameborder="0" allowfullscreen style="display:block; margin:auto;"></iframe>  

<br>
开始之前还是先看看已有的案例:  
01. [Raymarching的实现](https://entertainmentengineers.net/?p=51653)  
02. [液体单独的模型，将顶点拍平在水平面，compute shader中实现简单的物理](https://80.lv/articles/simulating-liquids-in-a-bottle-with-a-shader/)  
03. [UE5 Substrate实现](https://github.com/UniversalToolCompiler/UTC_LiquidShader/wiki/UE5-Dynamic-Liquid-Shader:-Technical-Breakdown)  
04. [UE材质实现](https://www.artstation.com/artwork/rAeqOO)  
05. 。。。  

看下来Raymarching不适合复杂的模型，Substrate有性能问题，04方案相比02更巧妙，最终是基于04案例来优化  
最终效果如下：  
<img src="{{ site.url }}/assets\img\2024-09-26-bottle-water-part-1\BWP1_01.gif" width="1000" 
style="display:block; margin:auto;">  

<br>
## 1 - 液面高度  
容器目前是一个旋转体，即截图转360度能形成的体积  
当只装了一半的水，最简单的情况，无论怎么旋转，水平线会一直停留在中间  
<img src="{{ site.url }}/assets\img\2024-09-26-bottle-water-part-1\BWP1_02.png" width="1000" 
style="display:block; margin:auto;">  
如果是水装的比一半多一些呢就不会距离中心一个高度了，理论上是可以通过数学计算出来，但是为了能够更多的应用在不同的形体上，额外引用了曲线资产来定义在不同旋转情况下液面的高度  
```HLSL
// CenterPos为液面的中心点
// LiquidDirection为(0,0,1)
CenterPos.z += HeightCurve  
LiquidMask = dot(normalize(WorldPos.xyz - CenterPos.xyz), LiquidDirection)  
```
<img src="{{ site.url }}/assets\img\2024-09-26-bottle-water-part-1\BWP1_03.png" width="1000" 
style="display:block; margin:auto;">  

<br>
## 2 - 液面物理  
真实的情况容器在运动下水面会有很多细微的起伏，这里会将整个液面当作一个整体来模拟  
通过液面的高低，以及倾斜来表达物理  
<img src="{{ site.url }}/assets\img\2024-09-26-bottle-water-part-1\BWP1_04.png" width="1000" 
style="display:block; margin:auto;">  
```HLSL
// HeightOffset和LiquidDirection会通过蓝图或者其他计算来输入
CenterPos.z += HeightCurve + HeightOffset  
LiquidMask = dot(normalize(WorldPos.xyz - CenterPos.xyz), LiquidDirection)  
```
这里用到来两个弹簧结果来模拟  
<img src="{{ site.url }}/assets\img\2024-09-26-bottle-water-part-1\BWP1_05.png" width="200" 
style="display:block; margin:auto;">  
<img src="{{ site.url }}/assets\img\2024-09-26-bottle-water-part-1\BWP1_06.png" width="200" 
style="display:block; margin:auto;">  

<br>
## 3 - 液面  
通过几何关系能够找到视线方向投射到液面中心平面的TestPos，进而判断TestPos是不是液面中心的半径范围内来绘制液面  
$$(EyePos – LiquidSurfaceCenter) · LiquidDireciton = (EyePos - TestPos) · LiquidDirection
$$
<img src="{{ site.url }}/assets\img\2024-09-26-bottle-water-part-1\BWP1_07.png" width="1000" 
style="display:block; margin:auto;">  

<br>
