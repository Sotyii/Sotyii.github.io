---
layout: post
title: "[Unreal] Slime Monster(WIP)"
author: "Tianyi Song"
categories: worklog
tags: [unreal, simulation, sph, niagara, fluid, rendering]
image: GameDev/2025-02-25-slime-monster/SM_00.gif
---

<br>
类似史莱姆质感的表现，已有一些实现可供参考：  
<br>
## Raymarching  
reference:  
<iframe width="100%"  height="400" src="https://www.youtube.com/embed/grmZ0I5-CgA" frameborder="0" allowfullscreen style="display:block; margin:auto;"></iframe>  
raymarching能够很好的实现和史莱姆和碰撞（距离场）融合的特性，但是仅通过材质表现很难表现出很好的物理效果  
这里通过整体的偏移来表达惯性，但还是少了很多内部运动带来的物理细节  
<img src="{{ site.url }}/assets/img/GameDev/2025-02-25-slime-monster/SM_01.gif" width="1000" 
style="display:block; margin:auto;">  

<br>
## 约束网格  
reference:  
<iframe width="100%"  height="400" src="https://www.youtube.com/embed/6TFpEDKANZY" frameborder="0" allowfullscreen style="display:block; margin:auto;"></iframe>  
类似与布料的效果  
性能目测没有很好，b站也有类似的实现，通过proceduralmesh，同时生成约束  
<iframe width="100%"  height="400" src="https://player.bilibili.com/player.html?bvid=BV1wY411r7Bf&page=1" frameborder="0" allowfullscreen style="display:block; margin:auto;"></iframe>  

<br>
## SPH  
最终还是参考Asher的实现方案，毕竟是最厉害的史莱姆  

<iframe width="100%"  height="400" src="https://www.youtube.com/embed/-cKgZrrBJ2w" frameborder="0" allowfullscreen style="display:block; margin:auto;"></iframe>  
这个讲座讲了整个模拟到渲染核心的思想，基于neighbor grid加速查询的sph模拟，3d粒子到2d屏幕空间的流体渲染等等  
[流体渲染主要来源于这篇gdc](https://developer.download.nvidia.cn/presentations/2010/gdc/Direct3D_Effects.pdf)  
什么是SPH，[这篇blog有很好的介绍](https://www.thecodeway.com/blog/2025/02/SPH001.html)

<br>
尝试了流体渲染，开销很大，暂时使用mesh renderer替代  
后续测试粒子到sdf和raymarching的方案  