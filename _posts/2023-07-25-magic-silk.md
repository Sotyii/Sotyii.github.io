---
layout: post
title: "[Unreal] Magic Silk"
author: "Tianyi Song"
categories: worklog
tags: [unreal, material, fabric]
image: 2023-07-25-magic-silk\MS_01.png
---


<br>
设计是魔法师的丝绸长袍  
动态刺绣  
亮片和流星  
丝绸修改brdf支持自定义specular color  
``` hlsl
if ( GBuffer.ShadingModelID == SHADINGMODELID_DEFAULT_LIT && GBuffer.CustomData.a > .0)
{			
	GBuffer.SpecularColor = ComputeF0(GBuffer.Specular, GBuffer.CustomData.rgb, GBuffer.CustomData.a);
}
```

<img src="{{ site.url }}/assets\img\2023-07-25-magic-silk\MS_02.gif" width="1000" 
style="display:block; margin:auto;">  