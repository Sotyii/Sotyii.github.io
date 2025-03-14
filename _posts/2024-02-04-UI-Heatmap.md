---
layout: post
title: "[UI] heatmap material"
author: "Tianyi Song"
categories: worklog
tags: [unreal, ui]
image: 2024-02-04-ui-heatmap\UH_01.gif
---



尝试在地图上以一种模糊的形式来表现即将遇到怪物的强度  

<br>
## 与gameplay之间的数据  
怪物的信息会存在一张rt中，第一个像素记录当前地图中怪物数量，后续逐个像素记录怪物的位置，强度，以及状态（是否死亡）  
``` hlsl
// NOTE point count is divided by 255 in cpp
int PointCount = int(positionBuffer.Load(int3(0, 0, 0)).x * 255);
for (int i = 0; i < PointCount; i++)
{
    float4 ptData = positionBuffer.Load(int3(i+1, 0, 0));
}
```

<br>
## 生成每个怪物的热力图  
生成根据距离怪物位置的权重，越近权重越大  
在怪物中心位置生成随机的移动的亮点权重  
根据权重信息去采样颜色曲线得到最终的结果  

uv的扰动形成边缘的变化  
怪物位置随时间的移动  
半径随时间变化  
亮点的移动和呼吸  

<br>
<img src="{{ site.url }}/assets\img\2024-02-04-ui-heatmap\UH_02.png" width="1000" 
style="display:block; margin:auto;">  
<img src="{{ site.url }}/assets\img\2024-02-04-ui-heatmap\UH_03.png" width="1000" 
style="display:block; margin:auto;">  
<img src="{{ site.url }}/assets\img\2024-02-04-ui-heatmap\UH_04.png" width="1000" 
style="display:block; margin:auto;">  

``` hlsl
// input
// uv
// positionBuffer // fisrt point count, then point data, ptData = float4(x,y,intensity,state)
// defaultImageRes
// imageRes
// time
// posNoise
// moveRange
// moveSpeed
// baseRadius
// radiusRange
// intensityRange
// xyRatio
// gradientMap

// intensityLimit
// backgroundRotateSpeed
// backgroundCenterDistance
// backgroundCenterDistanceRange
// backgroundIntensity
// backgroundIntensityRange
// backgrondMoveRange
// backgroundBaseRadius
// backgroundRadiusRange

// remap intensity
// bgIntensityMin 0.0
// bgIntensityMax 1.0

// bgNumMin 4
// bgNumMax 5

// shineIntensityMin
// shineIntensityMax

// shinCenterDistanceMin
// shinCenterDistanceMax

// shineNoiseMul
// shinePeriod
// shinePeriodSpace
// shineFadeTimeRatio
// shineCenterDistance
// shineIntensity
// shineMoveRange
// shineBaseRadius
// shineRadiusRange
// shineIntensityRange

// output
// finalColor
// finalOpacity
// finalWeight
// finalState


struct Functions
{
    // float2 to float noise
    float randomNoise(float2 c)
    {
        return frac(sin(dot(c.xy, float2(12.9898, 78.233))) * 43758.5453);
    }

    // heatmap
    float2 heatmap(float2 uv, float4 ptData, float time, int iteration, float xyRatio, float moveRange, float moveSpeed, float baseRadius, float radiusRange, float intensityRange)
    {
        // for different iteration get different move speed
        float moveTime = 0.5f * time * ( 1 + randomNoise(float2(iteration,iteration+0.2315451))) * moveSpeed;
        float2 move = float2(sin(0.9 * moveTime + iteration), cos(1.1 * moveTime+iteration*2)*xyRatio);
        float2 pointUV = ptData.xy + move*moveRange;

        float2 diff = uv - pointUV;
        diff.y *= xyRatio;
        float dist = length(diff);
        float radius = baseRadius *(1 + sin(time+iteration*3)*radiusRange);
        float intensity = max(ptData.z * (1 + sin(time + iteration*100)*intensityRange), .0f);
        float falloff = 0.1*randomNoise(float2(iteration+0.534124,iteration+0.2315451)) + 1.0;
        float weight = (1.0f - pow(dist /radius, 1.0f)) * intensity;
        weight = pow(max(.0,weight),2.2);

        float distDiff = dist - radius;
        float state = distDiff > 0.0f ? 0.0f : ptData.w;

        return float2(weight,state);
    }
};

Functions f;


// NOTE point count is divided by 255 in cpp
int PointCount = int(positionBuffer.Load(int3(0, 0, 0)).x * 255);
float3 finalColor = float3(.0f, .0f, .0f);
float weightSum = .0f;
float Pi = 3.1415926f;
float xyRatio = imageRes.y / imageRes.x;

float2 noiseUV = uv + posNoise;

backgroundCenterDistance = backgroundCenterDistance * defaultImageRes.x / imageRes.x;
moveRange = moveRange * defaultImageRes.x / imageRes.x;
baseRadius = baseRadius * defaultImageRes.x / imageRes.x;
// shineCenterDistance = shineCenterDistance * defaultImageRes.x / imageRes.x;
shineCenterDistanceMin = shineCenterDistanceMin * defaultImageRes.x / imageRes.x;
shineCenterDistanceMax = shineCenterDistanceMax * defaultImageRes.x / imageRes.x;


for (int i = 0; i < PointCount; i++)
{
    float4 ptData = positionBuffer.Load(int3(i+1, 0, 0));

    // dynamic high light
    if (ptData.z > intensityLimit)
    {
        float2 pointUV = ptData.xy;

        // background
        // 0.8-0.9 8  0.9-1.0 9
        float highLightRange = max(0.0001f, ptData.z - intensityLimit);
        int bgNum = bgNumMin + floor((ptData.z - intensityLimit) * (bgNumMax - bgNumMin + 0.999) / highLightRange);

        // rotate pos
        float rotateTime = time * backgroundRotateSpeed;        
        
        for (int j = 0; j < bgNum; j++)
        {
            // float childOffset = 
            float2 childPos = float2(sin(rotateTime + j*2.0f*Pi/bgNum), cos(rotateTime + j*2.0f*Pi/bgNum)/xyRatio);
            float2 pos = childPos*backgroundCenterDistance*(1+sin(time * f.randomNoise(float2(j+i*50+0.131,j+i*50+0.465468)))*backgroundCenterDistanceRange) + pointUV;
            float localIntensity = lerp(bgIntensityMin, bgIntensityMax, ptData.z) * backgroundIntensity *(f.randomNoise(float2(j+i*50+0.131,j+i*50+0.465468))*backgroundIntensityRange + 1.0f);
            float timeRand = time * (1 + f.randomNoise(float2(j+i*50+0.131,j+i*50+0.465468))*1.0f);
            float2 heatmap = f.heatmap(noiseUV, float4(pos, localIntensity, ptData.w), timeRand, i+j*15, xyRatio, moveRange*backgrondMoveRange, moveSpeed, baseRadius*backgroundBaseRadius, radiusRange*backgroundRadiusRange, intensityRange*0.02f);

            finalState = max(finalState, heatmap.y);
            weightSum += heatmap.x;
        }

        // shining effect
        float2 shineUV = uv + posNoise*shineNoiseMul;

        int shineNum = 5;
        float2 shinePos[] = {
            float2 (0.12f, 0.68f/xyRatio),
            float2 (0.55f, 0.33f/xyRatio),
            float2 (-0.43f, 0.16f/xyRatio),
            float2 (0.456f, -0.345f/xyRatio),
            float2 (-0.165f, -0.444f/xyRatio)
        };

        float singlePeriod = shinePeriod*(1+ f.randomNoise(float2(i+0.5438423,i+0.2315451))*0.1);
        float periodSpace = shinePeriodSpace*(1+ f.randomNoise(float2(i+0.456324,i+0.654451))*0.1);
        float periodTime = singlePeriod + periodSpace;

        // for all shinNum as a period
        int count = floor(time /periodTime) + i*10;
        int arrayCount = count / shineNum;
        int shineIndex = count % shineNum;

        // for a certain array count create a certain random array
        int shineArray[] = {0,0,0,0,4};
        for (int arrayIndex = 0; arrayIndex < shineNum-2; arrayIndex++)
        {
            int randomIndex = floor(f.randomNoise(float2(arrayCount+0.5438423,arrayCount+0.2315451))* (shineNum - 2.0f - 0.001f)) + 1;
            if (randomIndex != shineArray[arrayIndex])
            {
                shineArray[arrayIndex+1] = randomIndex;
            }
            else
            {
                shineArray[arrayIndex+1] = ((shineArray[arrayIndex] - 1) % (shineNum-2) + 1)% (shineNum-2) + 1;
            }
        }

        int randomShine = shineArray[shineIndex];
        float localTime = frac(time/periodTime)*periodTime;
        float2 pos = shinePos[randomShine]*lerp(shineCenterDistanceMin,shineCenterDistanceMax,ptData.z) + pointUV;
        float intensityTime = clamp(localTime, 0.0f, singlePeriod);
        float firstPhase = singlePeriod / (1.0f + shineFadeTimeRatio);
        float intensity = 0.0f;
        if (intensityTime < firstPhase)
        {
            intensity = lerp(shineIntensityMin,shineIntensityMax,ptData.z) * sin(0.5f * Pi * intensityTime/firstPhase)*shineIntensity;
        }
        else
        {
            intensity = lerp(shineIntensityMin,shineIntensityMax,ptData.z) * sin(0.5f * Pi * (intensityTime - firstPhase)/(singlePeriod - firstPhase) + 0.5f * Pi)*shineIntensity;
        }
        // float intensity = ptData.z * sin(Pi*intensityTime/singlePeriod)*shineIntensity;
        float timeRand = time * (1 + f.randomNoise(float2(i*100 + randomShine + 0.5438423,1*100 + randomShine + 0.2315451))*1.0f);
        float2 heatmap = f.heatmap(shineUV, float4(pos, intensity,ptData.w), timeRand, i*100 + randomShine, xyRatio, moveRange*shineMoveRange, moveSpeed, baseRadius*shineBaseRadius, radiusRange*shineRadiusRange, intensityRange*shineIntensityRange);

        finalState = max(finalState, heatmap.y);
        weightSum += heatmap.x;
    }
    else // basic heatmap background
    {
        //// random move
        float moveTime = 0.5f * time * ( 1 + f.randomNoise(float2(i,i+0.2315451))) * moveSpeed;
        float2 move = float2(sin(0.9 * moveTime + i), cos(1.1 * moveTime+i*2)*xyRatio);
        float2 pointUV = ptData.xy + move*moveRange;
        
        float2 diff = noiseUV - pointUV;
        diff.y *= xyRatio;
        float dist = length(diff);

        //// random radius
        float radius = baseRadius *(1 + sin(time+i*3)*radiusRange);
        //// random intensity
        float intensity = max(lerp(bgIntensityMin,bgIntensityMax,ptData.z) * (1 + sin(time + i*100)*intensityRange), .0f);

        //// random falloff
        float falloff = 0.1*f.randomNoise(float2(i+0.534124,i+0.2315451)) + 1.0;
        float weight = (1.0f - pow(dist /radius, 1.0f)) * intensity;

        //// get state
        float distDiff = dist - radius;
        float state = distDiff > 0.0f ? 0.0f : ptData.w;
        // finalState += state;

        /// end
        weightSum += pow(max(.0,weight),2.2)*state;
        finalState = max(finalState, state);
        // weightSum += .0f;
    }
}



float4 gradient = Texture2DSample(gradientMap, gradientMapSampler, float2(1.0f - weightSum, 0.5f));
finalColor = pow(gradient.rgb, 2.2);
finalOpacity = gradient.a * finalState;
// finalOpacity = gradient.a;
finalWeight = weightSum;

return finalColor;


```