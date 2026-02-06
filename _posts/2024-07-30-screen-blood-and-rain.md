---
layout: post
title: "[PP] Screen Blood and Rain"
author: "Tianyi Song"
categories: worklog
tags: [unreal, postprocess]
image: GameDev/2024-07-30-screen-blood-and-rain/SBR_03.gif
---

<br>
## 血迹下流
思路是在一个rt上绘制血迹初始的范围（一张好看的血点），随时间向下采样，和上一帧的rt减弱后叠加  
每次喷溅的初始范围可以在每次采样贴图时做一些随机的旋转缩放位移  
<img src="{{ site.url }}/assets/img/GameDev/2024-07-30-screen-blood-and-rain/SBR_03.gif" width="1000" 
style="display:block; margin:auto;">  
<img src="{{ site.url }}/assets/img/GameDev/2024-07-30-screen-blood-and-rain/SBR_01.gif" width="1000" 
style="display:block; margin:auto;">  

<br>
## 雨滴增加
通过对noise一定区间亮度值的选取来形成自然的扩散效果  
<img src="{{ site.url }}/assets/img/GameDev/2024-07-30-screen-blood-and-rain/SBR_04.gif" width="1000" 
style="display:block; margin:auto;">  
<img src="{{ site.url }}/assets/img/GameDev/2024-07-30-screen-blood-and-rain/SBR_02.gif" width="1000" 
style="display:block; margin:auto;">  

