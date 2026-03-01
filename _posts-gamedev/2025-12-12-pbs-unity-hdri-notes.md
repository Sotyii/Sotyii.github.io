---
layout: post
title: "Pbs Unity HDRI Notes"
author: "Tianyi Song"
categories: worklog
tags: [Unreal, Rendering, Lookdev,HDRI]
image: GameDev/2025-06-13-UE-tonemap-to-DaVinci/UE2DaVinci_01.png
---

<br>
## 背景  

项目组经历了换引擎，从自研引擎到ue，当前ue的lookdev环境从自研引擎迁移而来  
突然被反馈lookdev的效果和灯光美术的一个环境效果差异很大  
第一反应是灯光美术的打光的问题，没有怀疑到lookdev的环境上  
后面才发现确实是lookdev的数据出了问题  
  
  
古早的数据很难去考证正确性了，新的环境也暂时没有条件去拍摄和制作  
分成了两步，一步去还原unity的拍摄数据，一步是和原先自研引擎的数据对齐  
涉及到项目的内容，所以记录一下unity的环境在ue中搭建  

参考的文章  
[An Artist-Friendly Workflow for Panoramic HDRI](https://blog.selfshadow.com/publications/s2016-shading-course/#course_content)  

<img src="{{ site.url }}/assets/img/GameDev/2025-12-12-pbs-unity-hdri-notes/PBSNotes_01.png" width="1000" 
style="display:block; margin:auto;">  

LUX是51000  
EV100定为 14  
ISO100 F/8 1/250S  

## white balanced
有白平衡过的资源，跳过  

## absoluted
基于“Treasure Island - white balanced.exr”这张操作  
### ps处理
先按照ps的方法的操作了一下  
大致的方法是先算Luminance，乘一个半球的遮罩，binear的方式压缩成1x1即是半球亮度  
<img src="{{ site.url }}/assets/img/GameDev/2025-12-12-pbs-unity-hdri-notes/PBSNotes_02.png" width="1000" 
style="display:block; margin:auto;">
算得半球illuminance为5.2050  

51000/5.2050 = 9798.270893  

ps中相乘最大得值为20  
9798.270893 = 20 x 20 x 20 x 1.224783862  


### 脚本计算
参考给出的计算  
<img src="{{ site.url }}/assets/img/GameDev/2025-12-12-pbs-unity-hdri-notes/PBSNotes_03.png" width="1000" 
style="display:block; margin:auto;">

有太阳的hdr是5.21（和ps的结果差不多）  
没有太阳的hdr是1.993  
<img src="{{ site.url }}/assets/img/GameDev/2025-12-12-pbs-unity-hdri-notes/PBSNotes_04.png" width="1000" 
style="display:block; margin:auto;">

## 太阳位置
拍摄地点是sf 的 treasure island  
大致坐标：37.82702182429787, -122.37072477991983  
拍摄时间是 2016.03.13 10:15  
先在网站上查了一下：https://www.sunearthtools.com/dp/tools/pos_sun.php?lang=cn#top  
后面考虑自己算一下  
高度角为 40.45°  
<img src="{{ site.url }}/assets/img/GameDev/2025-12-12-pbs-unity-hdri-notes/PBSNotes_05.png" width="1000" 
style="display:block; margin:auto;">

拿着这个高度角在引擎里和有太阳的hdri是对不上的  
手动调整了一下到34°，和hdri的反射能够对上  
<img src="{{ site.url }}/assets/img/GameDev/2025-12-12-pbs-unity-hdri-notes/PBSNotes_06.png" width="1000" 
style="display:block; margin:auto;">

- 25.12.17  
高度角方位角直接通过hdr可以算出来  
lat-long的全景图，https://en.wikipedia.org/wiki/Equirectangular_projection  
ps像素从左到右，从上到下 太阳大致在5557, 1297  
360° x (5557+0.5)/8192 - 180°  = 64.22607422  
90° - 180° * (1297+0.5)/4096 = 32.98095703  
<img src="{{ site.url }}/assets/img/GameDev/2025-12-12-pbs-unity-hdri-notes/PBSNotes_07.png" width="1000" 
style="display:block; margin:auto;">
高度角就是32，ue设置为-32  
方位角比较麻烦，下图都是从上往下看  
左图hdri从左到右 -180到180，64度差不多是太阳的位置  
右图是ue的坐标，cubemap默认是从y轴正对的方向开始采样  
太阳的方向是朝x轴，所以最终是90+64  
<img src="{{ site.url }}/assets/img/GameDev/2025-12-12-pbs-unity-hdri-notes/PBSNotes_08.png" width="1000" 
style="display:block; margin:auto;">


## 引擎配置
unity的数据  
<img src="{{ site.url }}/assets/img/GameDev/2025-12-12-pbs-unity-hdri-notes/PBSNotes_09.png" width="1000" 
style="display:block; margin:auto;">

### 自己算的absolute
60%白板测出来luminance 0.374  
0.374 x 16000 = 5984  
<img src="{{ site.url }}/assets/img/GameDev/2025-12-12-pbs-unity-hdri-notes/PBSNotes_10.png" width="1000" 
style="display:block; margin:auto;">
<img src="{{ site.url }}/assets/img/GameDev/2025-12-12-pbs-unity-hdri-notes/PBSNotes_11.png" width="1000" 
style="display:block; margin:auto;">


~~贴图在ps中应该被clamp掉了部分数据，结果是不对的~~  
贴图在引擎里被clamp了，压缩设置是16位的  
<img src="{{ site.url }}/assets/img/GameDev/2025-12-12-pbs-unity-hdri-notes/PBSNotes_12.png" width="1000" 
style="display:block; margin:auto;">


推荐归一之后在引擎里乘系数  
<img src="{{ site.url }}/assets/img/GameDev/2025-12-12-pbs-unity-hdri-notes/PBSNotes_13.png" width="1000" 
style="display:block; margin:auto;">

贴图设置为32位后，结果正确了  
60%白板测出来luminance 0.593  
0.593 x 16000 = 9488  
<img src="{{ site.url }}/assets/img/GameDev/2025-12-12-pbs-unity-hdri-notes/PBSNotes_14.png" width="1000" 
style="display:block; margin:auto;">
<img src="{{ site.url }}/assets/img/GameDev/2025-12-12-pbs-unity-hdri-notes/PBSNotes_15.png" width="1000" 
style="display:block; margin:auto;">



### 用提供的absolute
60%白板测出来luminance 0.593  
0.593 x 16000 = 9488  
<img src="{{ site.url }}/assets/img/GameDev/2025-12-12-pbs-unity-hdri-notes/PBSNotes_16.png" width="1000" 
style="display:block; margin:auto;">
<img src="{{ site.url }}/assets/img/GameDev/2025-12-12-pbs-unity-hdri-notes/PBSNotes_17.png" width="1000" 
style="display:block; margin:auto;">


### 用Normalized
51000/(5.2050x2^14) = 0.598  
ps里hdr乘以0.598  

60%白板测出来luminance 0.577  
0.577 x 2^14 = 9453  
<img src="{{ site.url }}/assets/img/GameDev/2025-12-12-pbs-unity-hdri-notes/PBSNotes_18.png" width="1000" 
style="display:block; margin:auto;">
<img src="{{ site.url }}/assets/img/GameDev/2025-12-12-pbs-unity-hdri-notes/PBSNotes_19.png" width="1000" 
style="display:block; margin:auto;">


### no sun absolute
51000/5.21 = 9788.87  
直射光在平面的照度：51000 - 1.993x51000/5.21 = 31490.79  
直射光的照度：31490.79/sin(高度角) = 31490.79/sin(34) = 56314.71681  

60%白板测出来luminance 0.593  
0.593 x 16000 = 9488  
<img src="{{ site.url }}/assets/img/GameDev/2025-12-12-pbs-unity-hdri-notes/PBSNotes_20.png" width="1000" 
style="display:block; margin:auto;">

- 25.12.17更新  
在使用没有太阳的hdri时会发现光照结果和纯ibl在色相上有差别  
猜测是因为白平衡是在有太阳的情况做的，hdri去掉太阳之后的ibl照明则会出现偏色  
方法1：  
修改方向光的颜色来调整  
问题：阴影还是会偏色  
方法2：  
对去掉太阳的hdri重新白平衡  
问题：hdri颜色会不同于原先白平衡结果  
方法3：  
对去掉太阳的hdri去色  
- 25.12.18方向光照度  
鲍大佬提醒方向光的照度不应该直接用垂直于向上照度相减后除sin高度角，照度是单位面积全部方位角积分，不同朝向的积分是有差异的  
修改为两张hdri中计算朝向太阳的照度的差值  
(7.573284550547418 - 1.6446900322108455) x 51000/5.21 = 58032.80268  
<img src="{{ site.url }}/assets/img/GameDev/2025-12-12-pbs-unity-hdri-notes/PBSNotes_21.png" width="1000" 
style="display:block; margin:auto;">
- 25.12.18 方向光颜色  
选择用方法1修改方向光颜色  
测得白平衡后的太阳颜色：0.054, 0.0542, 0.0525  
缩放一下：0.9963099631, 1, 0.9686346863  
亮度：0.9961  
方向光强度：58032.80268/0.9961 = 58260.01675  
修改之后上图是纯ibl的结果，中图是替换方向光，修改方向光颜色，下图是替换方向光，使用白色方向光  
正对太阳的测试是更接近，垂直向上的测试结果更亮了  
60白板向上  
<img src="{{ site.url }}/assets/img/GameDev/2025-12-12-pbs-unity-hdri-notes/PBSNotes_22.png" width="1000" 
style="display:block; margin:auto;">
<img src="{{ site.url }}/assets/img/GameDev/2025-12-12-pbs-unity-hdri-notes/PBSNotes_23.png" width="1000" 
style="display:block; margin:auto;">
<img src="{{ site.url }}/assets/img/GameDev/2025-12-12-pbs-unity-hdri-notes/PBSNotes_24.png" width="1000" 
style="display:block; margin:auto;">
18灰板正对太阳  
<img src="{{ site.url }}/assets/img/GameDev/2025-12-12-pbs-unity-hdri-notes/PBSNotes_25.png" width="1000" 
style="display:block; margin:auto;">
<img src="{{ site.url }}/assets/img/GameDev/2025-12-12-pbs-unity-hdri-notes/PBSNotes_26.png" width="1000" 
style="display:block; margin:auto;">
<img src="{{ site.url }}/assets/img/GameDev/2025-12-12-pbs-unity-hdri-notes/PBSNotes_27.png" width="1000" 
style="display:block; margin:auto;">

- 26.01.17 方向光颜色  
从ps里采的是线性的颜色  
需要转到srgb  
0.9983761734, 1.0, 0.9860843918  
