---
layout: post
title: "[Tool] Icon Capture"
author: "Tianyi Song"
categories: worklog
tags: [Unreal, Tool, Editor Utility Blueprint]
---

<br>
给ui同事制作的批量拍摄工具  
手工的制作需要把模型放到场景中，寻找合适的拍摄角度，再截图，再进到ps加工  
工具在设计上支持用户自定义机位，多个机位，多个模型批量输出 
同时输出alpha，不再用抠图  

<br>
下面是具体工具使用流程：  
1. [准备](#准备)
2. [选择拍摄loot](#选择拍摄loot)
3. [设置旋转预设](#设置旋转预设)
4. [拍摄](#拍摄)

<br>
### 准备  
- 在场景中添加拍摄蓝图（场景中只能有一个该蓝图）  
- 打开拍摄菜单  
ArtTools->Loot Capture Tool  
工具中Settings面板如果没有加载出来，可以尝试关掉窗口再次打开  
<img src="{{ site.url }}/assets\img\2024-12-10-icon-capture-tool\ICT_01.png" width="1000" 
style="display:block; margin:auto;">  
<img src="{{ site.url }}/assets\img\2024-12-10-icon-capture-tool\ICT_02.png" width="1000" 
style="display:block; margin:auto;">  

<br>
### 选择拍摄loot  
- 选择要拍摄的LooseLoot加入列表，列表中的所有LooseLoot都在**Batch Capture**的操作中被导出  
content中选择任意数量父类为BP_LooseLoot的蓝图，点击**Select LooseLoot BP**  
<img src="{{ site.url }}/assets\img\2024-12-10-icon-capture-tool\ICT_03.png" width="1000" 
style="display:block; margin:auto;">  
<img src="{{ site.url }}/assets\img\2024-12-10-icon-capture-tool\ICT_04.png" width="1000" 
style="display:block; margin:auto;">  
- 在列表中选择作为预览  
点击**Select**，会出现绿色显示选中bp，点击**Set**，可以设置选中LooseLoot来预览  
点击**Remove**可以从列表中移除  
<img src="{{ site.url }}/assets\img\2024-12-10-icon-capture-tool\ICT_05.png" width="1000" 
style="display:block; margin:auto;">  

<br>
### 设置旋转预设
所有列表中勾选的预设，会在**Batch Capture**的操作中被导出  
Add Current Loot Rot To Preset: 可以旋转场景的中拍摄蓝图到合适的拍摄角度，记录当前Loot的旋转到下方列表的预设中  
Save Preset: 保存当前列表的预设，本地会存一个不提交的csv，如果要共享给其他人，可以上传预设文件  
Reset List: 初始化列表，会恢复到默认  
Toggle On All: 勾选全部预设  
Toggle Off All: 取消勾选全部预设  
Remove: 去掉该预设  
Set: 将当前Loot旋转到该预设  
<img src="{{ site.url }}/assets\img\2024-12-10-icon-capture-tool\ICT_06.png" width="1000" 
style="display:block; margin:auto;">  


<br>
### 拍摄  
**Capture**：仅导出当前视角的当前Loot  
**Batch Capture**：导出所有列表中的Loot，以及所有勾选旋转旋转预设  
默认储存路径： \Client\Projects\Saved\LootIcon  
默认导出FOV：25  