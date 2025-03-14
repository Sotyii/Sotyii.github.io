---
layout: post
title: "[Unreal] Forest Water"
author: "Tianyi Song"
categories: worklog
tags: [unreal, material, water]
image: 2024-09-30-forest-water\FW_02.png
---

<br>
## 效果  
希望能够在原本干净的森林水面上增加一些细节来增加氛围  
水面上会成团的聚集一些藻类，水草或者落叶等等  
分布范围主要是岸边，水中的短木周边，以及随机的分布  
<img src="{{ site.url }}/assets\img\2024-09-30-forest-water\FW_03.png" width="1000" 
style="display:block; margin:auto;">  

<br>
## 混合

<img src="{{ site.url }}/assets\img\2024-09-30-forest-water\FW_01.png" width="1000" 
style="display:block; margin:auto;">  

通过scenedepth和scenedepth without water来获得一个大致的浅水，以及靠近物体的区域  
<img src="{{ site.url }}/assets\img\2024-09-30-forest-water\FW_04.png" width="1000" 
style="display:block; margin:auto;">  

通过距离场需要对近距离和物件接触的位置做一定的过渡，生硬的接触会很难看  
<img src="{{ site.url }}/assets\img\2024-09-30-forest-water\FW_05.png" width="1000" 
style="display:block; margin:auto;">  

先做一层混合，大面积的水草和水藻的区域，边缘通过世界空间采样的noise来更好地过度  
<img src="{{ site.url }}/assets\img\2024-09-30-forest-water\FW_06.png" width="1000" 
style="display:block; margin:auto;">  

最后是混合更小区域地落叶，更靠近中心的部分  
<img src="{{ site.url }}/assets\img\2024-09-30-forest-water\FW_07.png" width="1000" 
style="display:block; margin:auto;">  