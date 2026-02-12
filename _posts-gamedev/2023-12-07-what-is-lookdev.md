---
layout: post
title: "What is Lookdev"
author: "Tianyi Song"
categories: worklog
tags: [Gamedev]
---

关于lookdev这部分的工作，个人感觉是很去定义的，因为不同的行业，游戏和影视，不同的工作室都有所差异。比如迪士尼对于[Look Devlopment外观开发](https://disneyanimation.com/process/look-development/)主要负责画面中的场景，角色的表现，工作内容则是贴图和材质的制作。比如视效从业者分享的[视效流程中的lookdev](http://www.andrew-whitehurst.net/pipeline.html)，lookdev部分和贴图，模型部门共同来完成外观开发，依赖与现场拍摄的hdr环境贴图作为光照，以及实物reference作为参考，如果没有实物，以supervisor和director决定表现是否正确  
<img src="{{ site.url }}/assets/img/GameDev/2023-12-07-what-is-lookdev/WIL_01.png" width="1000" 
style="display:block; margin:auto;">  

<br>


这篇log主要是记录自己对于游戏开发的lookdev环境疑问的解答  
首先会根据流程的不同将lookdev环境分为两种**真实lookdev环境**和**资产统一审查预览环境**  
先直接抛出结论吧，后面补充两个流程的实践案例，有兴趣可以详细了解  

**流程**  
- 真实lookdev环境  
采集真实数据：Calibration, Lightprobe, Ground Truth  
hdr 制作，高动态数据处理，色偏矫正  
引擎中重建，和实拍灰球铬球对齐  
- 资产统一审查预览环境  
网络hdr资源  
在引擎中调整直射光和天光达到合适的曝光  

<br>
**相同**  
- 曝光：都有合适的曝光  
- 色彩：通过色板矫正色偏；网络hdr 去色  
- 都可以作为统一标准的光环境用于资产检查和制作  

<br>
**不同**  
- 要制作电影品质的真实效果，真实lookdev 是必不可少的  
- 接近真实材质的生产依赖正确的物理光照数据，和对应真实的参考  

<br>



## 真实lookdev环境的工作流实践  

Reference: [Enabling a Look Development Workflow for UE4: From Shoot to Final LookDev Scene](https://www.unrealengine.com/en-US/events/unreal-fest-europe-2019/enabling-a-look-development-workflow-for-ue4-from-shoot-to-final-lookdev-scene)  

### 1 - 拍摄The Shoot  

#### OverView  
总共拍摄三个内容，Calibration(色卡，灰球，铬球)，Lightprobe(用于IBL的hdr)，Ground Truth(需要对齐的资产)
<img src="{{ site.url }}/assets/img/GameDev/2023-12-07-what-is-lookdev/WIL_02.png" width="1000" 
style="display:block; margin:auto;">  

<br>
#### Layout  
确定相机位置，中心和八方向点位  
注意拍摄起始点灰球的光影表现  
<img src="{{ site.url }}/assets/img/GameDev/2023-12-07-what-is-lookdev/WIL_03.png" width="1000" 
style="display:block; margin:auto;">  
<img src="{{ site.url }}/assets/img/GameDev/2023-12-07-what-is-lookdev/WIL_04.png" width="1000" 
style="display:block; margin:auto;">  

<br>
#### Camera Setting  
BaseEV通过中灰色卡来确定  
一组合适的光圈，快门速度，ISO 组合  
<img src="{{ site.url }}/assets/img/GameDev/2023-12-07-what-is-lookdev/WIL_05.png" width="1000" 
style="display:block; margin:auto;">  
确保相机的快门速度在AEB 拍摄的范围内  
假定设置的base 快门速度为1/125，最后得到的7 张照片的快门速度分别为  
1/8000，1/2000，1/500，1/125，1/30，1/8，0.5''  
使用固定色温  
<img src="{{ site.url }}/assets/img/GameDev/2023-12-07-what-is-lookdev/WIL_06.png" width="1000" 
style="display:block; margin:auto;">  

<br>
#### Calibration  
用于修复色偏  
<img src="{{ site.url }}/assets/img/GameDev/2023-12-07-what-is-lookdev/WIL_07.png" width="1000" 
style="display:block; margin:auto;">  
用于引擎中重建的参考  
<img src="{{ site.url }}/assets/img/GameDev/2023-12-07-what-is-lookdev/WIL_08.png" width="1000" 
style="display:block; margin:auto;">  
对于灰球铬球，拍摄1，5两个位置即可  
<img src="{{ site.url }}/assets/img/GameDev/2023-12-07-what-is-lookdev/WIL_09.png" width="1000" 
style="display:block; margin:auto;">  

<br>
#### Lightprobe  
<img src="{{ site.url }}/assets/img/GameDev/2023-12-07-what-is-lookdev/WIL_10.png" width="1000" 
style="display:block; margin:auto;">  
拍摄的准确  
向下拍摄镜头光学中心对齐轴心  
水平拍摄镜头光学中心在拍摄原点  
<img src="{{ site.url }}/assets/img/GameDev/2023-12-07-what-is-lookdev/WIL_11.png" width="1000" 
style="display:block; margin:auto;">  

<br>
#### Ground Truth  
一些拍摄建议：  
High Value Asset: 相机在1-8 的位置拍摄物体1-8 的朝向  
Mid Value Asset: 1 位置拍摄物体1-8 朝向，和物体1 朝向相机1-8 方向  
Lower Value Asset: 资产相机在1 和5 的位置拍摄物件1-8 的朝向  
<img src="{{ site.url }}/assets/img/GameDev/2023-12-07-what-is-lookdev/WIL_12.png" width="1000" 
style="display:block; margin:auto;">  

<br>
### 2 - 图像处理Image Ingestion  
1. 通过color checker shot 获得修复色偏的变换矩阵  
2. 一组包围曝光合成hdr  
3. 在PtGUI 中合成全景图  

<img src="{{ site.url }}/assets/img/GameDev/2023-12-07-what-is-lookdev/WIL_13.png" width="1000" 
style="display:block; margin:auto;">  
<img src="{{ site.url }}/assets/img/GameDev/2023-12-07-what-is-lookdev/WIL_14.png" width="1000" 
style="display:block; margin:auto;">  
这里是先使用TIFF 在PtGUI 中拼接，再合成7 张全景图为hdr  
而不是先将每一个镜头合成hdr，在PtGUI 中拼接hdr  
原因：PtGUI 的exr 计算可能和他们的不一致  
<img src="{{ site.url }}/assets/img/GameDev/2023-12-07-what-is-lookdev/WIL_15.png" width="1000" 
style="display:block; margin:auto;">  

<br>
### 3 - 引擎重建The LookDev Scene  
reference 照片Unlit 材质在场景中作为参考  
反复调节hdri 强度和灯光来达到一致  
<img src="{{ site.url }}/assets/img/GameDev/2023-12-07-what-is-lookdev/WIL_16.png" width="1000" 
style="display:block; margin:auto;">  

<br>
### 4 - LookDev有哪些帮助  
有了物理的灯光，更好的去制作基于物理的材质，接近真实的表现；解耦材质问题和灯光问题；作为打光数值的参考  
<img src="{{ site.url }}/assets/img/GameDev/2023-12-07-what-is-lookdev/WIL_17.png" width="1000" 
style="display:block; margin:auto;">  


<br>
## 查看环境构建实践  
Reference: [关于建立资产统一审查预览环境的一些说明](https://www.unrealengine.com/zh-CN/tech-blog/a-few-tips-for-building-unified-assets-reviewing-enviroment)  

面临的问题  
- 比如材质照不亮，过曝，材质多，不同人或设备看到差异化，反复迭代，品质不统一等等  

拆解成需求  
- 需要一个固定统一环境制作检查资产，能精确地表现美术师初衷效果（亮度，色彩，对比，质感）。资产，主要是贴图材质在任何灯光环境下都表现良好  
- 在不同的机器上表现一致；比如美术总监或Lead 查看时和美术师（内部或外包）看到一致  

<img src="{{ site.url }}/assets/img/GameDev/2023-12-07-what-is-lookdev/WIL_19.png" width="1000" 
style="display:block; margin:auto;">  


实现的方案  
- 准确曝光的环境  
- 涵盖不同光比环境  
- 保存并共享预览环境  


<img src="{{ site.url }}/assets/img/GameDev/2023-12-07-what-is-lookdev/WIL_18.png" width="1000" 
style="display:block; margin:auto;">  

<br>