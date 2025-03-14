---
layout: post
title: "Tri Planar Normal Fix"
author: "Tianyi Song"
categories: worklog
tags: [unreal,material]
image: 2024-11-11-tri-planar-normal-fix\TPNF_01.png
---

<img src="{{ site.url }}/assets\img\2024-11-11-tri-planar-normal-fix\TPNF_02.png" width="1000" 
style="display:block; margin:auto;">  
常规triplanar实现，在xy, yz, xz三个平面用世界坐标分别采样，通过xyz分量来做混合  
问题在模型旋转之后法线会表现出错  
如上图，top view，右模型z轴旋转180度  

<br>
- 原因  
因为我们通过世界坐标采样的是tangent space normal  
在模型旋转之后，mesh tangent/binormal也随之旋转，导致最终的结果在tangent/binormal的方向上反了  
<img src="{{ site.url }}/assets\img\2024-11-11-tri-planar-normal-fix\TPNF_03.png" width="1000" 
style="display:block; margin:auto;">  
- 修改
reference中已有一些现成的方案，在unreal的默认材质函数中找到了**WorldAlignedNormal**和**WorldAlignedNormal_HighQuality**  
默认是使用**WorldAlignedNormal_HighQuality**，reference中的Cross Product Tangent Reconstruction方案  
大致的思路就是自己构建三方向的tangent坐标系，将采样的结果转换到世界空间，再转回切线空间  
<img src="{{ site.url }}/assets\img\2024-11-11-tri-planar-normal-fix\TPNF_04.png" width="1000" 
style="display:block; margin:auto;">  


<br>
这篇blog对三维投射有很详细的介绍：  
reference: https://bgolus.medium.com/normal-mapping-for-a-triplanar-shader-10bf39dca05a#048e