---
layout: post
title: "[Unreal] Tonemap to DaVinci"
author: "Tianyi Song"
categories: worklog
tags: [Unreal, Rendering, Postprocess, DaVinci, Tonemap]
image: GameDev/2025-06-13-UE-tonemap-to-DaVinci/UE2DaVinci_01.png
---

<br>
## 背景  

希望能够用更专业的调色工具来调整ue在tonemap之前的hdr图像，然后再使用ue的tonemap  


<br>
## .Cube LUT格式

adobe:  

https://web.archive.org/web/20220215173646/https://wwwimages2.adobe.com/content/dam/acom/en/products/speedgrade/cc/pdfs/cube-lut-specification-1.0.pdf  

建议写法：  
```
// 3d template
// 一个65x65x65，输入范围在0-2的lut

TITLE "3dTestLUT"
LUT_3D_SIZE 65
DOMAIN_MIN 0 0 0
DOMAIN_MAX 2 2 2
```



在DaVinci里使用的lut，实际测试下来，包括看内置的lut，输入范围用的是LUT_3D_INPUT_RANGE  
```
// 3d template
// 一个65x65x65，输入范围在0-2的lut

TITLE "3dTestLUT"
LUT_3D_SIZE 65
LUT_3D_INPUT_RANGE 0 2
```



eg: 写一个对颜色没有修改的cube文件  
```
with open(file_path, "w") as f:
    f.write("LUT_3D_SIZE {}\n".format(LUT_SIZE))
    f.write("LUT_3D_INPUT_RANGE {} {}\n".format(0, LUT_3D_INPUT_RANGE))
    for b in range(LUT_SIZE):
        for g in range(LUT_SIZE):
            for r in range(LUT_SIZE):
                rgb = np.array([r, g, b]) / (LUT_SIZE - 1) * LUT_3D_INPUT_RANGE
                f.write("{:.10f} {:.10f} {:.10f}\n".format(*rbg))
```

<br>
## UE Tonemap过程

ue tonemap分为两步，先将tonemap写入到lut(CombineLUTS)，然后采样lut来做tonemap(Tonemap)  
<img src="{{ site.url }}/assets/img/GameDev/2025-06-13-UE-tonemap-to-DaVinci/UE2DaVinci_00.png" width="400" 
style="display:block; margin:auto;">  

<br>
### 1 - CombineLUTS  

写一个3d LUT  

输入UV和LayerIndex  

uv.xy 从(0,1)偏移到(-0.5/32, 1.05/32)，z还是(0,1)  

作为log空间颜色，从log转到linear  

转到AP1颜色空间  

在FilmTonemap之前会做ExpandGamut, ColorCorrect, BlueCorrect  

之后做BlueCorrect，转到WorkingColorSpace  

后续ColorCorrect，ColorGrade，LinearToSrgb  

最后结果/1.05  

```
#line 149 "/Engine/Private/PostProcessCombineLUTs.usf"
float4 CombineLUTsCommon(float2 InUV, uint InLayerIndex)
{

	// Neutral是uv 0-1的线性值
    // 也是LUTEncodedColor
    // LinearColor通过LogToLin从LUTEncodedColor转化
	float4 Neutral;
	{
		float2 UV = InUV - float2(0.5f / LUTSize, 0.5f / LUTSize);

		Neutral = float4(UV * LUTSize / (LUTSize - 1), InLayerIndex / (LUTSize - 1), 0);
	}
#line 178 "/Engine/Private/PostProcessCombineLUTs.usf"


	float4 OutColor = 0;
	
	const float3x3 AP1_2_sRGB = mul( XYZ_2_sRGB_MAT, mul( D60_2_D65_CAT, AP1_2_XYZ_MAT ) );
	const float3x3 AP0_2_AP1 = mul( XYZ_2_AP1_MAT, AP0_2_XYZ_MAT );
	const float3x3 AP1_2_AP0 = mul( XYZ_2_AP0_MAT, AP1_2_XYZ_MAT );

	const float3x3 AP1_2_Output  = OuputGamutMappingMatrix( OutputGamut );

	float3 LUTEncodedColor = Neutral.rgb;
	float3 LinearColor;
	
	if (GetOutputDevice() >= 3)
	{
		LinearColor = ST2084ToLinear(LUTEncodedColor) * LinearToNitsScaleInverse;
	}
	
	else
	{
		LinearColor = LogToLin(LUTEncodedColor) - LogToLin(0);
	}


	float3 BalancedColor = LinearColor;


	float3 ColorAP1 = mul( (float3x3)WorkingColorSpace.ToAP1, BalancedColor );

	
 	float  LumaAP1 = dot( ColorAP1, AP1_RGB2Y );
	float3 ChromaAP1 = ColorAP1 / LumaAP1;

	float ChromaDistSqr = dot( ChromaAP1 - 1, ChromaAP1 - 1 );
	float ExpandAmount = ( 1 - exp2( -4 * ChromaDistSqr ) ) * ( 1 - exp2( -4 * ExpandGamut * LumaAP1*LumaAP1 ) );
	
	
	const float3x3 Wide_2_XYZ_MAT = 
	{
		0.5441691,  0.2395926,  0.1666943,
		0.2394656,  0.7021530,  0.0583814,
		-0.0023439,  0.0361834,  1.0552183,
	};

	const float3x3 Wide_2_AP1 = mul( XYZ_2_AP1_MAT, Wide_2_XYZ_MAT );
	const float3x3 ExpandMat = mul( Wide_2_AP1, AP1_2_sRGB );

    // ColorExpand
	float3 ColorExpand = mul( ExpandMat, ColorAP1 );
	ColorAP1 = lerp( ColorAP1, ColorExpand, ExpandAmount );
    
    // ColorCorrect
	ColorAP1 = ColorCorrectAll( ColorAP1 );

	
	float3 GradedColor = mul( (float3x3)WorkingColorSpace.FromAP1, ColorAP1 );

	const float3x3 BlueCorrect =
	{
		0.9404372683, -0.0183068787, 0.0778696104,
		0.0083786969,  0.8286599939, 0.1629613092,
		0.0005471261, -0.0008833746, 1.0003362486
	};
	const float3x3 BlueCorrectInv =
	{
		1.06318,     0.0233956, -0.0865726,
		-0.0106337,   1.20632,   -0.19569,
		-0.000590887, 0.00105248, 0.999538
	};
	const float3x3 BlueCorrectAP1    = mul( AP0_2_AP1, mul( BlueCorrect,    AP1_2_AP0 ) );
	const float3x3 BlueCorrectInvAP1 = mul( AP0_2_AP1, mul( BlueCorrectInv, AP1_2_AP0 ) );

	// BlueCorrection = 0.6
	ColorAP1 = lerp( ColorAP1, mul( BlueCorrectAP1, ColorAP1 ), BlueCorrection );

	// Tonemap
	float3 ToneMappedColorAP1 = FilmToneMap( ColorAP1 );
	// ToneCurveAmount = 1
	ColorAP1 = lerp(ColorAP1, ToneMappedColorAP1, ToneCurveAmount);
	
	// BlueCorrection = 0.6
	ColorAP1 = lerp( ColorAP1, mul( BlueCorrectInvAP1, ColorAP1 ), BlueCorrection );

	// from ColorAP1 to workingspace
	float3 FilmColor = max(0, mul( (float3x3)WorkingColorSpace.FromAP1, ColorAP1 ));

#line 304 "/Engine/Private/PostProcessCombineLUTs.usf"
	
	// ColorCorrection
	FilmColor = ColorCorrection( FilmColor );

	// ColorScale = 1,1,1
	// OverlayColor = 0,0,0,0
	// return FilmColor
	float3 FilmColorNoGamma = lerp( FilmColor * ColorScale, OverlayColor.rgb, OverlayColor.a );
	
	GradedColor = lerp(GradedColor * ColorScale, OverlayColor.rgb, OverlayColor.a);

	// InverseGamma = 0.45455, 1.00, 1.00
	FilmColor = pow( max(0, FilmColorNoGamma), InverseGamma.y );
	
	
	float3 OutDeviceColor = 0;
	[branch]

	
	if( GetOutputDevice() == 0)
	{		
	    // WorkingColorSpace.bIsSRGB = 1, return FilmColor
		float3 OutputGamutColor = WorkingColorSpace.bIsSRGB ? FilmColor : mul( AP1_2_Output, mul( (float3x3)WorkingColorSpace.ToAP1, FilmColor ) );

		// LinearToSrgb?
		OutDeviceColor = LinearToSrgb( OutputGamutColor );
	}

	
	OutColor.rgb = OutDeviceColor / 1.05;
	OutColor.a = 0;

	return OutColor;
}

#line 423 "/Engine/Private/PostProcessCombineLUTs.usf"
void MainPS(FWriteToSliceGeometryOutput Input, out float4 OutColor : SV_Target0)
{
	OutColor = CombineLUTsCommon(Input.Vertex.UV, Input.LayerIndex);
}
```

<br>
## 2 - Tonemap  

TonemapCommonPS:  

进LUT之前，做了SceneColorTint, Exposure, Vignette, LocalExposure, Bloom  

LUT之后，加上FilmGrain  

ColorLookupTable:  

LinearColor转到Log，偏移(0,1)到(0.5/32, 31.5/32)，采样texture3d，采样结果*1.05  

```
#line 266 "/Engine/Private/PostProcessTonemap.usf"
float4 TonemapCommonPS(
	float2 UV,
	float2 Vignette,
	float4 GrainUV,
	float2 ScreenPos, 
	float2 FullViewUV,
	float4 SvPosition,
	out float OutLuminance
	)
{
	const float OneOverPreExposure = View.OneOverPreExposure;
	const float2 EyeAdaptationData = EyeAdaptationBuffer[0].xw;
	const float GlobalExposure = EyeAdaptationData.x;

	float2 SceneUV = UV.xy;

	
	float4 SceneColor;
	{
		
#line 308 "/Engine/Private/PostProcessTonemap.usf"

			SceneColor = SampleSceneColor(SceneUV);
	}

		const float MobileVignette = 1.0;
	
	
	float3 Bloom;
	{
		float2 BloomUV;

		{
			BloomUV = ApplyScreenTransform(UV, ColorToBloom);
			BloomUV = clamp(BloomUV, BloomUVViewportBilinearMin, BloomUVViewportBilinearMax);
		}
		

		float4 RawBloom = Texture2DSample(BloomTexture, BloomSampler, BloomUV);

		{
			float2 DirtLensUV = ConvertScreenViewportSpaceToLensViewportSpace(ScreenPos) * float2(1.0f, -1.0f);

			float3 RawBloomDirt = Texture2DSample(BloomDirtMaskTexture, BloomDirtMaskSampler, DirtLensUV * .5f + .5f).rgb;

			Bloom = RawBloom.rgb * (1.0 + RawBloomDirt * BloomDirtMaskTint.rgb);
		}
		
	}
	
		const float VignetteMask = MobileVignette * ComputeVignetteMask(Vignette, TonemapperParams.x);

		// ColorScale0 = 1,1,1,1
		
		const float3 SceneColorTint = ColorScale0 * SceneColorApplyParamaters[0].xyz;
	
	
#line 391 "/Engine/Private/PostProcessTonemap.usf"

	const float LocalExposure = 1.0;
	

	
	
#line 443 "/Engine/Private/PostProcessTonemap.usf"


	float4 OutColor = 0;
#line 462 "/Engine/Private/PostProcessTonemap.usf"

	{
		// OneOverPreExposure = 12406.74414
		// SceneColorTint = 
		// GlobalExposure = EyeAdaptationBuffer[0].x
		float3 FinalLinearColor = SceneColor.rgb * SceneColorTint * (OneOverPreExposure * GlobalExposure * VignetteMask * LocalExposure);
		
		
		
		{
			FinalLinearColor += Bloom * (OneOverPreExposure * GlobalExposure * VignetteMask);
		}
		
		
		
		
#line 488 "/Engine/Private/PostProcessTonemap.usf"

		
		float3 OutDeviceColor = ColorLookupTable(FinalLinearColor);

		float LuminanceForPostProcessAA = dot(OutDeviceColor, float3(0.299f, 0.587f, 0.114f));
		OutLuminance = LuminanceForPostProcessAA;

		
		OutColor.rgb = OutDeviceColor;
}

float3 ColorLookupTable( float3 LinearColor )
{
	float3 LUTEncodedColor;

        // linear to log
		LUTEncodedColor = LinToLog( LinearColor + LogToLin( 0 ) );

	

	float3 UVW = LUTEncodedColor * LUTScale + LUTOffset;


	float3 OutDeviceColor = Texture3DSample( ColorGradingLUT, ColorGradingLUTSampler, UVW ).rgb;



	
	return OutDeviceColor * 1.05;
}
```
<br>
### 3 - 整理

从SceneColor到最后输出：  

SceneColor, SceneColorTint, Exposure, Vignette, LocalExposure, Bloom, LUT, FilmGrain  

LUT中包含：  

ExpandGamut, ColorCorrect, BlueCorrect, FilmTonemap, BlueCorrect, ColorCorrect，ColorGrade，LinearToSrgb  



其他：  

后处理ToneCurveAmount改为0，和MRQ中勾选disable tone curve的结果一致  

<br>
## DaVinci LUT制作

输入是未作tonemap的线性颜色，用3d的lut精度不够，先用1d的4096 lut转到log，在log空间用3d lut做tonemap


<br>
### 1 - LinearToLog
```
def LinToLog( LinearColor ):
	LinearRange = 14
	LinearGrey = 0.18
	ExposureGrey = 444
	
	LogColor = np.log2(LinearColor) / LinearRange - np.log2(LinearGrey) / LinearRange + ExposureGrey / 1023.0

	LogColor = np.clip( LogColor, .0, 1.0 )

	return LogColor

def LogToLin( LogColor ):
    LinearRange = 14
    LinearGrey = 0.18
    ExposureGrey = 444

    LinearColor = pow(2, (LogColor - ExposureGrey / 1023.0) * LinearRange)* LinearGrey

    return LinearColor

with open(file_path, "w") as f:
    f.write("TILE {}\n".format('"'+LUT_Title+'"'))
    f.write("LUT_1D_SIZE {}\n".format(LUT_SIZE))
    f.write("LUT_1D_INPUT_RANGE {} {}\n".format(LUT_1D_INPUT_RANGE_Min, LUT_1D_INPUT_RANGE_Max))
    for i in range(LUT_SIZE):
        LinearColor = (i / (LUT_SIZE - 1)) * (LUT_1D_INPUT_RANGE_Max - LUT_1D_INPUT_RANGE_Min) + LUT_1D_INPUT_RANGE_Min
        LogColor = LinToLog(LinearColor + LogToLin(0))
        LogColor = LogColor * 0.96875 + 0.01563
        f.write("{:.10f} {:.10f} {:.10f}\n".format(*(LogColor, LogColor, LogColor)))
```
<br>
### 2 - FilmTonemap
```
FilmSlope = 0.88
FilmToe = 0.55
FilmShoulder = 0.26
FilmBlackClip = 0.0
FilmWhiteClip = 0.04

def LinToLog( LinearColor ):
	LinearRange = 14
	LinearGrey = 0.18
	ExposureGrey = 444
	
	LogColor = np.log2(LinearColor) / LinearRange - np.log2(LinearGrey) / LinearRange + ExposureGrey / 1023.0

	LogColor = np.clip( LogColor, .0, 1.0 )

	return LogColor


def LogToLin( LogColor ):
    LinearRange = 14
    LinearGrey = 0.18
    ExposureGrey = 444

    LinearColor = pow(2, (LogColor - ExposureGrey / 1023.0) * LinearRange)* LinearGrey

    return LinearColor

XYZ_2_AP0_MAT = np.array([
    [1.0498110175, 0.0000000000, -0.0000974845],
    [-0.4959030231, 1.3733130458, 0.0982400361],
    [0.0000000000, 0.0000000000, 0.9912520182]
])

AP1_2_XYZ_MAT = np.array([
    [0.6624541811, 0.1340042065, 0.1561876870],
    [0.2722287168, 0.6740817658, 0.0536895174],
    [-0.0055746495, 0.0040607335, 1.0103391003]
])

AP0_2_AP1_MAT = np.array([
    [1.4514393161, -0.2365107469, -0.2149285693],
    [-0.0765537734, 1.1762296998, -0.0996759264],
    [0.0083161484, -0.0060324498, 0.9977163014]
])


WorkingColorSpace_ToAP1 = np.array([
    [0.6131, 0.33952, 0.04738, 0.00],
    [0.07019, 0.91635, 0.01345, 0.00],
    [0.02062, 0.10957, 0.86981, 0.00],
    [0.00, 0.00, 0.00, 1.00]
])

WorkingColorSpace_ToAP1_3x3 = WorkingColorSpace_ToAP1[:3, :3]


WorkingColorSpace_FromAP1 = np.array([
    [1.70505, -0.62179, -0.08326, 0.00],
    [-0.13026, 1.1408, -0.01055, 0.00],
    [-0.024, -0.12897, 1.15297, 0.00],
    [0.00, 0.00, 0.00, 1.00]
])

WorkingColorSpace_FromAP1_3x3 = WorkingColorSpace_FromAP1[:3, :3]


AP1_RGB2Y = np.array([
    0.2722287168,  # AP1_2_XYZ_MAT[0][1]
    0.6740817658,  # AP1_2_XYZ_MAT[1][1]
    0.0536895174   # AP1_2_XYZ_MAT[2][1]
])

def rgb_2_saturation(rgb): 
    minrgb = min(min(rgb[0],rgb[1]), rgb[2])
    maxrgb = max(max(rgb[0],rgb[1]), rgb[2])
    return (max(maxrgb, 1e-10) - max(minrgb, 1e-10)) / max(maxrgb, 1e-2)

def rbg_2_yc(rgb):
    r = rgb[0]
    g = rgb[1]
    b = rgb[2]
    chroma = math.sqrt(b * (b - g) + g * (g - r) + r * (r - b))

    return (b + g + r + 1.75 * chroma) / 3.

def sigmoid_shaper(x):

    t = max(1 - abs(0.5 * x), 0)
    y = 1 + np.sign(x) * (1 - t * t)
    return 0.5 * y

def glow_fwd(ycIn, glowGainIn, glowMid):
    glowGainOut = 0.0

    if (ycIn <= 2. / 3. * glowMid):
        glowGainOut = glowGainIn

    elif (ycIn >= 2 * glowMid):
        glowGainOut = 0

    else:
        glowGainOut = glowGainIn * (glowMid / ycIn - 0.5)

    return glowGainOut

def clamp(value, min_value, max_value):
    return max(min(value, max_value), min_value)

def rgb_2_hue(rgb):

    hue = .0
    if (rgb[0] == rgb[1] and rgb[1] == rgb[2]):
        
        hue = 0
    else:
        hue = (180. / math.pi) * math.atan2(math.sqrt(3.0) * (rgb[1] - rgb[2]), 2 * rgb[0] - rgb[1] - rgb[2])

    if (hue < 0.):
        hue = hue + 360

    return clamp(hue, 0, 360)


def center_hue( hue,  centerH):

    hueCentered = hue - centerH
    if (hueCentered < -180.):
        hueCentered += 360
    elif (hueCentered > 180.):
        hueCentered -= 360
    return hueCentered

def Square(x):
    return x * x

def smoothstep(edge0, edge1, x):
    x = clamp((x - edge0) / (edge1 - edge0), 0.0, 1.0)
    return x * x * (3 - 2 * x)


def FilmToneMap( LinearColor ):

    AP1_2_AP0 = np.matmul(XYZ_2_AP0_MAT, AP1_2_XYZ_MAT)

    ColorAP1 = LinearColor
    ColorAP0 = np.matmul(AP1_2_AP0, ColorAP1)


    RRT_GLOW_GAIN = 0.05
    RRT_GLOW_MID = 0.08

    saturation = rgb_2_saturation(ColorAP0)
    ycIn = rbg_2_yc(ColorAP0)
    s = sigmoid_shaper((saturation - 0.4)/0.2)
    addedGlow = 1 + glow_fwd( ycIn, RRT_GLOW_GAIN * s, RRT_GLOW_MID)
    ColorAP0 *= addedGlow


    RRT_RED_SCALE = 0.82
    RRT_RED_PIVOT = 0.03
    RRT_RED_HUE = 0
    RRT_RED_WIDTH = 135

    hue = rgb_2_hue( ColorAP0 )
    centeredHue = center_hue( hue, RRT_RED_HUE )
    hueWeight = Square( smoothstep( 0, 1, 1 - abs( 2 * centeredHue / RRT_RED_WIDTH ) ) )
        
    ColorAP0[0] += hueWeight * saturation * (RRT_RED_PIVOT - ColorAP0[0]) * (1. - RRT_RED_SCALE)


    WorkingColor = np.matmul(AP0_2_AP1_MAT, ColorAP0)
    WorkingColor = np.maximum(WorkingColor, 0.0)

    # Pre desaturate
    WorkingColor = lerp( np.dot(WorkingColor, AP1_RGB2Y), WorkingColor, 0.96 )

    ToeScale = 1 + FilmBlackClip - FilmToe
    ShoulderScale = 1 + FilmWhiteClip - FilmShoulder

    InMatch = 0.18
    OutMatch = 0.18

    ToeMatch = 0.0
    if FilmToe > 0.8:
        # 0.18 will be on straight segment
        ToeMatch = ( 1 - FilmToe  - OutMatch ) / FilmSlope + np.log10( InMatch )
    else:
        # 0.18 will be on toe segment
        # Solve for ToeMatch such that input of InMatch gives output of OutMatch.
        bt = ( OutMatch + FilmBlackClip ) / ToeScale - 1
        ToeMatch = np.log10( InMatch ) - 0.5 * np.log( (1+bt)/(1-bt) ) * (ToeScale / FilmSlope)

    StraightMatch = ( 1 - FilmToe ) / FilmSlope - ToeMatch
    ShoulderMatch = FilmShoulder / FilmSlope - StraightMatch

    LogColor = np.log10( WorkingColor )
    StraightColor = FilmSlope * ( LogColor + StraightMatch )

    ToeColor		= (    -FilmBlackClip ) + (2 *      ToeScale) / ( 1 + np.exp( (-2 * FilmSlope /      ToeScale) * ( LogColor -      ToeMatch ) ) )
    ShoulderColor	= ( 1 + FilmWhiteClip ) - (2 * ShoulderScale) / ( 1 + np.exp( ( 2 * FilmSlope / ShoulderScale) * ( LogColor - ShoulderMatch ) ) )

    ToeColor = np.where(LogColor < ToeMatch, ToeColor, StraightColor)
    ShoulderColor = np.where(LogColor > ShoulderMatch, ShoulderColor, ToeColor)

    t = np.clip( ( LogColor - ToeMatch ) / ( ShoulderMatch - ToeMatch ), .0 , 1.0 )

    if ShoulderMatch < ToeMatch:
        t = 1 - t
    t = (3-2*t)*t*t

    ToneColor = lerp( ToeColor, ShoulderColor, t )

    # Post desaturate
    ToneColor = lerp( np.dot( ToneColor, AP1_RGB2Y ), ToneColor, 0.93 )


    return np.maximum(0, ToneColor)

def LinearToSrgb( Lin):
    return LinearToSrgbBranching(Lin)

def LinearToSrgbBranching(Lin):
    return np.array([
        LinearToSrgbChannel(Lin[0]),
        LinearToSrgbChannel(Lin[1]),
        LinearToSrgbChannel(Lin[2])])

def LinearToSrgbChannel(Lin):
    if Lin <= 0.0031308:
        return Lin * 12.92
    else:
        return 1.055 * np.power(Lin, 1.0 / 2.4) - 0.055

def LinearToST2084( Lin ):
    m1 = 0.1593017578125
    m2 = 78.84375
    c1 = 0.8359375
    c2 = 18.8515625
    c3 = 18.6875
    C = 10000

    L = Lin/C
    Lm = pow(L, m1)
    N1 = ( c1 + c2 * Lm )
    N2 = ( 1.0 + c3 * Lm )
    # N = N1 * rcp(N2)
    N = N1 / N2
    P = pow( N, m2 )

    return P

def CombineLUTs( LutEncodedColor ):

    LutEncodedColor -= np.array([0.5/32.0, 0.5/32.0, 0.0])
    LutEncodedColor *= np.array([32.0/31.0, 32.0/31.0, 1.0])

    LinearColor = LogToLin(LutEncodedColor) - LogToLin(0)
    BalancedColor = LinearColor

    # ColorAP1 = mul( (float3x3)WorkingColorSpace.ToAP1, BalancedColor )
    ColorAP1 = np.matmul(WorkingColorSpace_ToAP1_3x3, BalancedColor)

    GradedColor = np.matmul(WorkingColorSpace_FromAP1_3x3, ColorAP1)

    ToneMappedColorAP1 = FilmToneMap(ColorAP1)
    ColorAP1 = lerp(ColorAP1, ToneMappedColorAP1, 1.0)

    # FilmColor = max(0, mul( (float3x3)WorkingColorSpace.FromAP1, ColorAP1 ))
    FilmColor = np.clip(np.matmul(WorkingColorSpace_FromAP1_3x3, ColorAP1), .0, 1.0)

    # FilmColorNoGamma = lerp( FilmColor * ColorScale, OverlayColor.rgb, OverlayColor.a )
    FilmColorNoGamma = FilmColor

    # FilmColor = pow( max(0, FilmColorNoGamma), InverseGamma.y )
    FilmColor = np.power(np.maximum(0, FilmColorNoGamma), 1.0)

    # Tonemapper Output sRGB
    OutputGamutColor = FilmColor
    OutputDeviceColor = LinearToSrgb(OutputGamutColor)

    # Tonemapper Output LinearEXR
    # OutputGamutColor = np.matmul(WorkingColorSpace_ToAP1_3x3, GradedColor)
    # OutputDeviceColor = LinearToST2084(OutputGamutColor)

    OutColor = OutputDeviceColor/1.05

    return OutColor

def lerp(a, b, t):
    return a + t * (b - a)

lut_folder = os.path.join(file_folder, "LUT")
if not os.path.exists(lut_folder):
    os.makedirs(lut_folder)

file_path = os.path.join(lut_folder, "UE_Filmic_01.cube")

with open(file_path, "w") as f:
    f.write("LUT_3D_SIZE {}\n".format(LUT_SIZE))
    f.write("LUT_3D_INPUT_RANGE {} {}\n".format(0, LUT_3D_INPUT_RANGE))
    for b in range(LUT_SIZE):
        for g in range(LUT_SIZE):
            for r in range(LUT_SIZE):
                LUTEncodedColor = np.array([r, g, b]) / (LUT_SIZE - 1) * LUT_3D_INPUT_RANGE
                tonemapped = CombineLUTs( LUTEncodedColor )*1.05
                f.write("{:.10f} {:.10f} {:.10f}\n".format(*tonemapped))
```
<br>
### 3 - 结果对比

截帧Final Color  
<img src="{{ site.url }}/assets/img/GameDev/2025-06-13-UE-tonemap-to-DaVinci/UE2DaVinci_01.png" width="400" 
style="display:block; margin:auto;">  

截帧Scene Color + LUT  
<img src="{{ site.url }}/assets/img/GameDev/2025-06-13-UE-tonemap-to-DaVinci/UE2DaVinci_02.png" width="400" 
style="display:block; margin:auto;">  

MRQ exr with tonemap, linear to srgb  
<img src="{{ site.url }}/assets/img/GameDev/2025-06-13-UE-tonemap-to-DaVinci/UE2DaVinci_03.png" width="400" 
style="display:block; margin:auto;">  

MRQ exr disabled tonemap, custom lut  
<img src="{{ site.url }}/assets/img/GameDev/2025-06-13-UE-tonemap-to-DaVinci/UE2DaVinci_04.png" width="400" 
style="display:block; margin:auto;">  

<br>
## 后处理设置

从scenecolor到finalcolor  

从SceneColor到最后输出：  

SceneColor, SceneColorTint, Exposure, Vignette, LocalExposure, Bloom, LUT, FilmGrain  

LUT中包含：  

ExpandGamut, ColorCorrect, BlueCorrect, FilmTonemap, BlueCorrect, ColorCorrect，ColorGrade，LinearToSrgb  



从合成顺序来看，LUT中Tonemap处于中间阶段，为了能够使Tonemap在中间，和Tonemap在最后的结果一致，所以要关掉Tonemap之后的后处理：  

FilmGrain, ColorCorrect, BlueCorrect, ColorGrade  



烘焙的Film设置：  
```
FilmSlope = 0.88
FilmToe = 0.55
FilmShoulder = 0.26
FilmBlackClip = 0.0
FilmWhiteClip = 0.04
```