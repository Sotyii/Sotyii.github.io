---
layout: post
title: "[Unreal] interactive water"
author: "Tianyi Song"
categories: worklog
tags: [unreal,simulation]
---

在实现水体交互的过程中，发现很多不错的交互效果其实只需要一步很简单的推导，包括ue的示例工程，以及shadertoy上的一些项目。从shadertoy的评论和shader中找到了公式最早的出处[Hugo Elias](https://web.archive.org/web/20160418004149/http://freespace.virgin.net/hugo.elias/graphics/x_water.htm)，但是文章中的解释和推导很难让人信服。后续则是通过对浅水方程的近似推导出经验公式的物理正确性，看起来合理的东西背后也是有其合理的原因。最后便是ue的实现部分。  

reference:  
[Ue交互水示例](https://dev.epicgames.com/documentation/en-us/unreal-engine/creating-a-fluid-surface-with-blueprints-and-render-targets?application_version=4.27#4-endresult)  
[shadertoy示例](https://www.shadertoy.com/view/3sB3WW)  
[我的测试](https://www.shadertoy.com/view/MX3XWS)


<br>

## 目录
1. [Hugo Elias 2D Water](#hugo-elias-2d-water)
2. [From SWE(Shallow Water Equation) to Hugo Elias 2D Water](#from-sweshallow-water-equation-to-hugo-elias-2d-water)
3. [Unreal Implementation](#unreal-implementation)

<br>

## Hugo Elias 2D Water
[Hugo Elias 2D Water](https://web.archive.org/web/20160418004149/http://freespace.virgin.net/hugo.elias/graphics/x_water.htm)  
<img src="{{ site.url }}/assets\img\2024-07-05-interactive-water\IW_01.png" width="1000" 
style="display:block; margin:auto;">  
思路是通过前两帧的高度，来推出当前帧的高度  
怎么推出，以下是作者的**“解释”**  
<img src="{{ site.url }}/assets\img\2024-07-05-interactive-water\IW_02.png" width="1000" 
style="display:block; margin:auto;">  

```
//上上帧(曲线2)的数值近似上一帧(曲线1)到当前帧(曲线0)的变化
Velocity(x, y) = -Buffer2(x, y)

//采样上下左右来扩散
Smoothed(x, y) = (Buffer1(x-1, y) +
                  Buffer1(x+1, y) + 
                  Buffer1(x, y-1) +
                  Buffer1(x, y+1)) / 4 

// 乘2减弱速度的影响
NewHeight(x, y) = (Smoothed(x, y)*2 + Velocity(x, y)) * Damping
```

这个解释没什么道理。。。但确实达到了很好的结果  



<br>

## From SWE(Shallow Water Equation) to Hugo Elias 2D Water
浅水方程的前提，同一水平位置下的不同深度的水速度相同  
欧拉视角，将水离散成小的空间单位，对于每一个空间单位分析，如下图所示平面单元  
<img src="{{ site.url }}/assets\img\2024-07-05-interactive-water\IW_03.png" width="400" 
style="display:block; margin:auto;">  
在浅水方程中，通过联立两个方程，单位空间的质量守恒（流入和流出的体积变化）和动量守恒（受力和速度变化）来求解  

**质量守恒**
<img src="{{ site.url }}/assets\img\2024-07-05-interactive-water\IW_04.png" width="1000" 
style="display:block; margin:auto;">  
通过体积变化联立速度和高度的关系，简单的数学：  
速度 x 截面面积 x 时间  
这里我们定义从左向右方向为正方向，左侧流入的体积减去右侧流出的体积则为X方向上的体积变化，Y方向同理  

$$dh_{x} = \frac{(v_{左} - v_{右}) * h_{x} *dx*dt}{dx*dx}$$  

$$dh_{x} = (v_{左} - v_{右}) * h_{x} *dt / dx$$  

<br>

**动量守恒**
<img src="{{ site.url }}/assets\img\2024-07-05-interactive-water\IW_05.png" width="1000" 
style="display:block; margin:auto;">  
这里分析单元的速度变化，速度的变化来源于受力，受力来自与两侧因为高度差产生的压力，简单的数学：  
压强 x 受力面积 x 时间 / 质量  
已知 $$P=\rho * g * h$$  

$$dv_{x}=\frac{(\rho *g*(h_{x-0.5} - h) - \rho *g*(h_{x+0.5} - h))*s}{s*dx*\rho}*dt$$  

$$dv_{x} = (h_{x-0.5} - h_{x+0.5}) * g *dt / dx$$  

这样我们得到的并不是在质量守恒中所需的单元间的速度，而是单元的速度  
有个优雅的方式来得到单元间的速度，就是将$$x=x-0.5$$带入上一个公式  
也很好理解，将原先的离散网格整体偏移0.5，偏移的单元中心便是未偏移的单元边缘  
$$dv_{x-0.5}=(h_{x-1} - h_{x})*g*dt/dx$$  

为了方便，避免用0.5，可以为速度构建另一套grid，也就是所谓的mac网格（高度记录在中心，速度记录在红点的网格线）  
在这套网格下等式可以更新为：  
$$dv_{x}=(h_{x-1} - h_{x})*g*dt/dx$$  

<img src="{{ site.url }}/assets\img\2024-07-05-interactive-water\IW_06.png" width="300" 
style="display:block; margin:auto;">  


<br>

**联立求解**

写一下上面两步得到等式：  
等式1：$$dv_{(x,t)} = (h_{(x-1,t)} - h_{(x,t)})*g*dt/dx$$  
等式2：$$dh_{(x,t)} = (v_{(x,t)} - v_{(x+1,t)})*h_{(x,t)}*dt/dx$$  
<br>
记x处当前速度为v(x,t)， 高度为h(x,t)，上一帧为v(x,t-1) h(x,t-1)  
将等式2中的速度改写 $$v_{(x,t)}=v_{(x,t-1)}+dv_{(x,t-1)}$$，替换掉v：
$$dh_{(x,t)} = (v_{(x,t-1)}+dv_{(x,t-1)} - v_{(x+1,t-1)} - dv_{(x+1,t-1)})*h_{(x,t)}*dt/dx$$  
$$dh_{(x,t)} = (v_{(x,t-1)}- v_{(x+1,t-1)})*h_{(x,t)}*dt/dx  +(dv_{(x,t-1)}  - dv_{(x+1,t-1)})*h_{(x,t)}*dt/dx$$  
<br>
再次将 $$(v_{(x,t-1)}- v_{(x+1,t-1)})$$ 带入等式2  
$$dv_{(x,t-1)}$$ 和 $$dv_{(x+1,t-1)}$$ 带入等式1  
$$dh_{(x,t)} = (h_{(x,t)}- h_{(x,t-1)})*\frac{h_{(x,t)}}{h_{(x,t-1)}} +(h_{(x-1,t)}  + h_{(x+1,t)} - 2h_{(x,t)})*h_{(x,t)}*g*dt*dt/(dx*dx)$$  
<br>
$$dh_{(x,t)}=h_{(x,t+1)}-h_{(x,t)}$$带入  
浅水方程中 $$\frac{h_{(x,t)}}{h_{(x,t-1)}}$$ 为了简化可以近似为1  
最后的 $$h_{(x,t)}*g*dt*dt/(dx*dx)$$ 可以视为常量s  
最终便简化到了这样一个眼熟的等式，
$$h_{(x,t+1)} = 2h_{(x,t)}- h_{(x,t-1)} +(h_{(x-1,t)}  + h_{(x+1,t)} - 2h_{(x,t)})*s$$  
可以发现在s等于1时，和Elias Hugo的等式是一致的  
<img src="{{ site.url }}/assets\img\2024-07-05-interactive-water\IW_07.jpg" width="1000" 
style="display:block; margin:auto;">  

<br>

## Unreal Implementation
ue的niagara grid2d很契合2d的模拟  
- set grid2d collection
- 角色碰撞输入
- 根据角色输入，水面高度计算输入水面高度
- 模拟计算
- 通过高度计算法线并写入rt
<img src="{{ site.url }}/assets\img\2024-07-05-interactive-water\IW_07.png" width="200" 
style="display:block;margin:auto;">  
角色的输入可以通过蓝图或者其他方式给到niagara data interface  
<img src="{{ site.url }}/assets\img\2024-07-05-interactive-water\IW_08.jpg" width="1000" 
style="display:block; margin:auto;">  
<img src="{{ site.url }}/assets\img\2024-07-05-interactive-water\IW_09.jpg" width="1000" 
style="display:block; margin:auto;">  
具体解算部分shader:  
<img src="{{ site.url }}/assets\img\2024-07-05-interactive-water\IW_10.png" width="1000" 
style="display:block; margin:auto;">  

<br>
