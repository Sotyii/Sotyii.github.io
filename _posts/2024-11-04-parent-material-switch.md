---
layout: post
title: "[Unreal] parent material switch"
author: "Tianyi Song"
categories: worklog
tags: [unreal,material,python,batch,tool]
---

## 需求  
一些历史原因，武器的贴图通道和通用材质贴图通道不一样  
在启用vt之后，决定将武器材质贴图统一  

左图为原武器贴图通道，右图为通用材质贴图通道  
多了一个specular level，塞到了ORM的alpha中  
<img src="{{ site.url }}/assets\img\2024-11-04-parent-material-switch\PMS_01.png" width="1000" 
style="display:block; margin:auto;">  

<br>
## 步骤  
大致的思路是先备份出来当前版本所有子材质的信息，包括贴图和参数  
然后将贴图的通道格式转换生成新的一套贴图  
最后是把新贴图导回到引擎替换掉旧的贴图，并修改为正确的命名  
把武器材质的母材质都替换成统一材质，并重新给到正确的贴图和参数  

脚本都是使用unreal python，贴图换通道用的python pillow module  
因为没打算做成工具，写的有点随意，仅供参考，附在最后  
[unreal python api](https://dev.epicgames.com/documentation/en-us/unreal-engine/python-api/?application_version=5.3)  

1. [导出‘M_WeaponStandard’全部子材质，引用的贴图，材质实例参数](#1---导出m_weaponstandard全部子材质引用的贴图材质实例参数)
2. [处理贴图通道](#2---处理贴图通道)
3. [导回贴图到引擎并重命名](#3---导回贴图到引擎并重命名)
4. [设置母材质并设置对应参数](#4---设置母材质并设置对应参数)
5. [检查](#5---检查)

<br>

### 1 - 导出‘M_WeaponStandard’全部子材质，引用的贴图，材质实例参数  

**获取全部子材质**  
获取母材质的全部引用，从中筛选出是"MaterialInstanceConstant"类的子材质，以防子材质有子材质，可以多查几轮  
(后面发现unreal.MaterialEditingLibrary里也有获取子材质的方法，但是没试过)  

``` python
referenceList = unreal.AssetRegistryHelpers.get_asset_registry().get_referencers(packageName,unreal.AssetRegistryDependencyOptions())
```  

<br>
**导出贴图**  
同上通过引用找出子材质引用的贴图，通过后缀判断贴图用途  
(后面发现有更好的写法，就是通过材质参数来找使用的贴图)  
用unreal.AssetExportTask()将引用的贴图导出  

``` python
# export texture
lTexturePath = os.path.join(lTextureDir, textureExportName + '.tga')              
task = unreal.AssetExportTask()
task.set_editor_property('automated', True)
task.set_editor_property('prompt', False)
task.set_editor_property('exporter',unreal.TextureExporterTGA())
task.set_editor_property('filename', lTexturePath)
task.set_editor_property('object', textureAsset)                    

print('exporting texture:',lTexturePath)

try:
	check = unreal.Exporter.run_asset_export_task(task)
except:
	check = False

if check:
	pass
else:
	print('Export texture failed:',lTexturePath)
```

<br>
**导出参数**  
unreal.MaterialInstanceConstant类可以查询scalar, vector, texture类型的参数，但是bool类型要通过unreal.MaterialEditingLibrary来查询  

``` python
tintColor = materialAsset.get_vector_parameter_value('Albedo Tint')
saturation = materialAsset.get_scalar_parameter_value('Saturation')
brightness = materialAsset.get_scalar_parameter_value('Brightness')
contrast = materialAsset.get_scalar_parameter_value('Contrast')
hue = materialAsset.get_scalar_parameter_value('Hue')
normalIntensity = materialAsset.get_scalar_parameter_value('Normal Intensity')
minRoughness = materialAsset.get_scalar_parameter_value('Min Roughness')
maxRoughness = materialAsset.get_scalar_parameter_value('Max Roughness')
contrastRoughness = materialAsset.get_scalar_parameter_value('Contrast Roughness')
localWetness = materialAsset.get_scalar_parameter_value('LocalWetness')
wetMaskAsset = materialAsset.get_texture_parameter_value('Wet Mask')
wetMask = wetMaskAsset.get_path_name()
# useEmissiveColor = materialAsset.get_scalar_parameter_value('UseEmissiveColor')
# use mel to get bool parameter value
useEmissiveColor = MEL.get_material_instance_static_switch_parameter_value(materialAsset, 'UseEmissiveColor')
```  

<br>
**一些坑**  
- 贴图如果导入时是16位的，无法导出成tga，自动检查出16的贴(pillow可以判断)，手动处理  
- 可以通过贴图参数来找引用，虽然我是找全部引用的贴图，通过后缀来判断用途(不推荐)  

<br>

### 2 - 处理贴图通道  

**通道交换**  

``` python
D_r, D_g, D_b, D_a = inputTextureD_img.split()
M_r, M_g, M_b = inputTextureM_img.split()
N_r, N_g, N_b = inputTextureN_img.split()

# create output textures
outputTextureD = Image.merge("RGBA", (D_r, D_g, D_b, Image.new('L', D_r.size, 255)))
outputTextureORM = Image.merge("RGBA", (D_a, N_b, M_r, M_g))
outputTextureN = Image.merge("RGB", (N_r, N_g, Image.new('L', M_r.size, 255)))
```  

<br>
**一些坑**  
- 确认一套贴图尺寸一致，不然换通道麻烦了  
- convert之后还是保持同样的贴图名。。。ue重新导入的时候方便一些(因为没找到ue怎么reimport with new file，导入的时候名字不一样就不会替换)  


<br>

### 3 - 导回贴图到引擎并重命名  

**导回引擎**  
没找到怎么reimport with new file，只能原名导入了  

``` python
task = unreal.AssetImportTask()

task.filename = newfilePath
task.set_editor_property('destination_path', os.path.dirname(textureAsset.get_path_name()))
task.replace_existing = True
task.save = True
# print('task.destination_name:', textureAsset.get_name())
# task.destination_name = str(textureAsset.get_name())

assetTools = unreal.AssetToolsHelpers.get_asset_tools()
print('start to reimport:', newfilePath)
assetTools.import_asset_tasks([task])
```  

<br>
**重命名**

unreal.EditorAssetLibrary重命名  
<img src="{{ site.url }}/assets\img\2024-11-04-parent-material-switch\PMS_02.png" width="1000" 
style="display:block; margin:auto;">  

``` python
reimport_texture_with_new_file(textureMAsset, convertORMPath)
textureMAsset.compression_settings = unreal.TextureCompressionSettings.TC_MASKS

newTextureMAssetData = EAL.find_asset_data(textureM)
newTextureMPackageName = str(newTextureMAssetData.package_name)
newTextureMPath = newTextureMPackageName
if not textureMName.upper().endswith('_ORM'):            
	if textureMName.upper().endswith('_M'):
		editTextureMPackageName = newTextureMPackageName[:-2] + '_ORM'
		print('rename_asset:', newTextureMPackageName, editTextureMPackageName)
		EAL.rename_asset(newTextureMPackageName, editTextureMPackageName)
	else:
		print('textureMName:', textureMName + ' is not end with _ORM')
```  

<br>
**一些坑**  
- 没找到reimport的接口，就原名导入  
- 同时可以设置一下贴图的各种配置（目前设了一下压缩格式  
- 一个名字错误的贴图被多个材质引用，第一次改名后生成redirector，无法重新导入，检测同资产跳过  
- asset相关改动，先确认一下是否被别人check out  

<br>

### 4 - 设置母材质并设置对应参数  

**设置**  
都可以通过unreal.MaterialEditingLibrary中的方法来设置  
<img src="{{ site.url }}/assets\img\2024-11-04-parent-material-switch\PMS_03.png" width="1000" 
style="display:block; margin:auto;">  
<img src="{{ site.url }}/assets\img\2024-11-04-parent-material-switch\PMS_04.png" width="1000" 
style="display:block; margin:auto;">  

<br>
**一些坑**  
- asset相关改动，先确认一下是否被别人check out  
- 注意材质参数含义（同样命名计算不同  

<br>

### 5 - 检查  
一开始是没有检查的，心里也很虚，果然出问题了，美术反馈效果不对，查了一下部分贴图的压缩方式没有被正确设置（原因未知  
这里是再次检查子材质的贴图引用的各种压缩方式是否正确，把有问题的都修改了  

可以再完善一下检查这一步  



<br>