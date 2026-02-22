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

