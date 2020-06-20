---
title: "003-Shadow(阴影)——CatLikeCoding学习笔记"
date: 2020-6-20
categories:
  - Post
tags:
  - 渲染
  - CatLikeCoding学习笔记
---
# 阴影
当本该射射向B物体的光源的射线被A物体挡住了，于是B物体那片被挡住区域我们称之为**阴影**。  
![图一]({{ site.url }}{{ site.baseurl }}/assets/../../assets/images/003_1.png)  

在现实中，在完全点亮和完全黑暗之间有一段过渡区域，我们称之为penumbra（半阴影）。它存在的原因是因为所有光源都有体积，于是有些地方只有部分的光源才能照到。  

Unity不支持penumbra，但是Unity支持软阴影，不过这是一种阴影滤波技术，不是对penumbra的模拟。  

## 阴影映射（Shadow Mapping）
市面上有几种支持实时阴影的技术，每一种都有各自的优点和缺点。而Unity用的是现在最常见的技术，**阴影映射（Shadow Mapping）**。这表示unity将阴影信息储存在贴图中。  

### 渲染深度贴图（Depth Texture,场景摄像机的阴影映射图）
当方向阴影被启用，unity开始在渲染渲染程序中加入一次Depth Pass,目的就是为了生成一张**深度贴图**。  

**深度贴图**是由一个摄像机在它的**裁剪空间(clip space)**[^1]生成的。深度信息，也就是每个片元的Z（范围是0~1）作为颜色值储存在这张贴图上。  

![图二]({{ site.url }}{{ site.baseurl }}/assets/../../assets/images/003_2.png)  
深度贴图，如上图所示，越远颜色越亮  

[^1]:裁剪空间定义了摄像机可以看到的区域。//TODO 这里要讲得更加详细才行啊。  

### 渲染阴影映射图（Rendering to Shadow Maps）
**阴影映射图**（Shadow Maps）和深度贴图很类似，是光源在自己的光源空间，生成的一张贴图。里面也储存着深度信息，但是这个**深度信息**代表了光线在运动了**多少距离后撞击到了物体表面**。  

**PS**:unity的实现原理就是在光源处生成了一个摄像机，然后生成了一张深度贴图作为该光源的阴影映射图。不过，因为是方向光，这个摄像机也是个**正交摄像机**。  

**原来，unity并不是每个光源只渲染一张阴影映射图**，默认情况下每个光源渲染4张阴影映射图，每一张在不同的位置渲染的。原因是我们选择了four shadow cascades(四级阴影层叠).如果我们选择two cascades（二级阴影层叠）则每个光源渲染两次。如果我们选择没有阴影层叠，则每个光源渲染一次。  
![图三]({{ site.url }}{{ site.baseurl }}/assets/../../assets/images/003_3.png)  

### 屏幕空间的阴影映射图（Collecting Shadows）
我们有摄像机视角下场景的**深度信息（深度图，Depth Texture）**，我们由光源视角下场景的**深度信息(阴影映射图，Shadow Maps)** 。虽然这些数据，被储存在不同的裁剪空间下（摄像机的和光源的），但是我们知道这些空间的位置和旋转关系，我们可以从一个空间转换到另一个空间。这使得我们可以在我们想要的视角下，比较两方的深度信息。理论上来说我们由会有两个向量同时以一个点为终点，如果出现这么一个点，就代表摄像机和光源都可以照射到这个点，所以它被点亮了。如果光的向量没能到达这个点，这说明光被挡住了，也代表了这个点处于阴影之内。  

![图四]({{ site.url }}{{ site.baseurl }}/assets/../../assets/images/003_4.png)  
屏幕空间下的阴影  

Unity通过渲染一个覆盖了整个视野的四边形来创造这些贴图，它在这个pass用了
Hidden/Internal-ScreenSpaceShadows shader。每个片元从场景摄像机和光源的深度贴图采样，做对比,将最后的深度值赋给**场景空间的阴影映射图**。被点亮的像素设为1，**完全**处于阴影中的设为0。此时，unity也可以通过采样去创造软阴影。

### 阴影质量
因为场景摄像机的阴影映射图和光源的阴影映射图，因为旋转不相同且分辨率也不相同。  
#### 阴影映射图的分辨率和阴影范围
**在相同的阴影映射图分辨率下，阴影范围越近，范围内的阴影的分辨率越高。** 这个很好理解，你阴影距离越小，阴影映射图在分辨率不变的情况下，可分配给近处阴影映射图的分辨率越多。  
##### 如果让阴影映射图的分辨率和阴影范围不变小的情况下，优化阴影质量呢？答案是cascades
当cascades被打开时，多张阴影映射图被渲染到一张贴图上，每张阴影映射图负责一段特定的距离。坏处是现在我们每一帧得渲染场景更多变。（看开启了多少个cascades）  
![图五]({{ site.url }}{{ site.baseurl }}/assets/../../assets/images/003_5.png)  

cascades 范围带（band） 的形状取决于quality setting 的 Shadow Projection.  
默认情况下是**Stable Fit**:cascades的范围取决于距离摄像机的位置。  
另一种选项是**Close Fit**:cascades的范围取决于距离摄像机的深度。这产生了以摄像机视角方向的矩形的范围带。  
![图六]({{ site.url }}{{ site.baseurl }}/assets/../../assets/images/003_6.png)  
![图七]({{ site.url }}{{ site.baseurl }}/assets/../../assets/images/003_7.png)  
我们可以将scene视图的Shading Mode选为**Miscellaneous / Shadow Cascades**来看cascades的范围带。  

**Close Fit**使我们更有效率的去使用阴影贴图，也表现出了更高质量的阴影。**但是，这种阴影投射方式取决于相机的位置和旋转，造成的结果是每次相机旋转或者移动，阴影映射图也跟着改变，如果你可以看到阴影的锯齿，可以很明显的看到阴影的边缘在抖动。** 这也是为什么默认设置是**Stable Fit**的原因。  

###### 为什么Stable Fit在相机移动时就没有影响？
其实是有影响的，但是当摄像机的位置改变时，unity仍然可以对其这些阴影映射图对齐，使像素表现上去像静态的。
当然，cascade的作用范围带是移动的，所以作用范围带之间的交界处会改变。但是如果你注意不到这些作用范围带，你也不会注意到它们移动.  

### 阴影痤疮（Shadow Acne）  
当我们使用低质量的硬阴影时，我们可以看到阴影会出现一些奇怪的地方。不幸的是，不管Quality Setting如果设置，都会有这种情况发生。  

阴影映射图的每一个像素都代表了光线撞击了到了一个物体表面。但是，像素不是一个点，而是一个区域。而且它们与光的方向对齐而不是物体表面。产生的结果是，部分像素的一角会突出物体表面，我们称之为**阴影痤疮**。  
![图八]({{ site.url }}{{ site.baseurl }}/assets/../../assets/images/003_8.png)  

另一个造成阴影痤疮的原因是**数值进度限制**。但距离很小时，这些限制会造成不正确的结果。  
![图九]({{ site.url }}{{ site.baseurl }}/assets/../../assets/images/003_9.png)  

有一种避免这种问题的方法是当渲染阴影映射图的时候添加一个**深度偏移量**。这使得阴影被阴影被推导物体表面之内。  
![图十]({{ site.url }}{{ site.baseurl }}/assets/../../assets/images/003_10.png)  

**阴影偏移量（shadow bias）**是在每个光源里设置的，默认是0.05.  
注意：过大的阴影偏移量可能会产生偏移（被称之为peter panning）。  
![图十一]({{ site.url }}{{ site.baseurl }}/assets/../../assets/images/003_11.png)  

### 抗锯齿（Anti-Aliasing）  
unity的抗锯齿设置并不会对阴影映射图有影响，所以阴影还是会有锯齿。  

## 投射阴影

### 在shader中支持投射阴影  
为了支持所有相关的passes，我们要在我们的pass中再添加一个pass。并且设置lightmode为**ShadowCaster**.因为我们只关心深度信息。  

```
SubShader {

                Pass {
                        Tags {
                                "LightMode" = "ForwardBase"
                        }

                        …
                }

                Pass {
                        Tags {
                                "LightMode" = "ForwardAdd"
                        }

                        …
                }

                Pass {
                        Tags {
                                "LightMode" = "ShadowCaster"
                        }

                        CGPROGRAM

                        #pragma target 3.0

                        #pragma vertex MyShadowVertexProgram
                        #pragma fragment MyShadowFragmentProgram

                        #include "My Shadows.cginc"//我们自定义的include文件

                        ENDCG
                }
        }

```
#### 第一步，基础支持
顶点着色器很简单，只是把位置从模型空间转换到裁剪空间（clipspace）.片元着色器什么都不做，只是返回zero。GPU自己会为我们记录深度值。  
```
#if !defined(MY_SHADOWS_INCLUDED)#define MY_SHADOWS_INCLUDED

#include "UnityCG.cginc"

struct VertexData {
        float4 position : POSITION;};

float4 MyShadowVertexProgram (VertexData v) : SV_POSITION {
        return mul(UNITY_MATRIX_MVP, v.position);}

half4 MyShadowFragmentProgram () : SV_TARGET {
        return 0;}

#endif
```
![图十二]({{ site.url }}{{ site.baseurl }}/assets/../../assets/images/003_12.png)  
这些已经足够去投射方向光的阴影了。

#### 第二部，支持偏移（Bias）
为了支持深度偏移（depth bias），我们可以使用**UnityCG**的UnityApplyLinearShadowBias 方法。  
```
float4 MyShadowVertexProgram (VertexData v) : SV_POSITION {
        float4 position = mul(UNITY_MATRIX_MVP, v.position);
        return UnityApplyLinearShadowBias(position);
}
```
为了也支持深度法线偏移，我们必须移动基于法线向量移动顶点位置。所以我们要在顶点信息中添加法线。接着，我们用再**UnityCG**中定义的**UnityClipSpaceShadowCasterPos**方法来应用偏移。  
```
struct VertexData {
        float4 position : POSITION;
        float3 normal : NORMAL;
};

float4 MyShadowVertexProgram (VertexData v) : SV_POSITION {
        float4 position = UnityClipSpaceShadowCasterPos(v.position.xyz, v.normal);
        return UnityApplyLinearShadowBias(position);
}
```
现在，这个shader是一个功能完全的shadow caster.  

### 在shader中支持接收阴影
当主方向光投射阴影时，unity会寻找有开启**SHADOWS_SCREEN**关键字的Shader.  
```
#pragma multi_compile _ SHADOWS_SCREEN
```
当添加了这个multi_compile后，shader会报错 说_ShadowCoord不存在。发生的原因是当有阴影时，UNITY_LIGHT_ATTENUATION这个宏的行为会发生一些变化,所以我们也要特别处理。
```
#if defined(SHADOWS_SCREEN)
                float attenuation = 1;
#else
                UNITY_LIGHT_ATTENUATION(attenuation, 0, i.worldPos);
#endif
```
#### 阴影采样
为了得到阴影，我们要采样屏幕空间的阴影映射图。为了达到这个目的我们必须知道屏幕贴图的坐标。像采样其他贴图那样我们需要在顶点着色器传递信息给片元着色器的结构体里添加一个字段。因为我们要传递齐次剪裁空间的位置，所以我们用float4.  
```
struct Interpolators {
        …

        #if defined(SHADOWS_SCREEN)
                float4 shadowCoordinates : TEXCOORD5;
        #endif
};
```
我们可以通过定义在 **AutoLight** 的 **_ShadowMapTexture** 来获取屏幕空间阴影映射图。
先展示如果采集阴影贴图（后面会有变化）

```
UnityLight CreateLight (Interpolators i) {
        …

        #if defined(SHADOWS_SCREEN)
         float attenuation = tex2D(_ShadowMapTexture, i.shadowCoordinates.xy);
        #else
         UNITY_LIGHT_ATTENUATION(attenuation, 0, i.worldPos);
        #endif

        …
}
```
##### 1.将-1~1的坐标数值范围转化到0~1
因为在**裁剪空间**，所有可视范围内的XY坐标数值范围在-1~1，我们要转化到**屏幕空间**的0~1.。
因为我们要处理**透视转换**（perspective transformation），我们偏移坐标多少取决于坐标的深度。这种情况下，偏移量和w（齐次坐标的第四象限的值）相等。
```
#if defined(SHADOWS_SCREEN)
     i.shadowCoordinates.xy = (i.position.xy + i.position.w) * 0.5;
            i.shadowCoordinates.zw = i.position.zw;
#endif
```
![图十三]({{ site.url }}{{ site.baseurl }}/assets/../../assets/images/003_13.png)  

##### 2.将齐次坐标转化为屏幕空间坐标
这时，显示仍然时错误的，因为我们仍然使用的是齐次坐标，我们要转化成屏幕坐标，我们需要将结果除以w.
```
#if defined(SHADOWS_SCREEN)
     i.shadowCoordinates.xy = (i.position.xy + i.position.w) * 0.5/ i.position.w;
            i.shadowCoordinates.zw = i.position.zw;
#endif
```
然而结果仍然有问题，因为我们先除以w再结果插值后传给片元着色器的数值范围永远是0~xy/w内的插值，我们想传递给片元着色器范围在0~xy范围内的插值怎么办？  
我们先传递插值，在片元着色器内再除以w。
```

Interpolators MyVertexProgram (VertexData v) {
        …

        #if defined(SHADOWS_SCREEN)
         i.shadowCoordinates.xy =
                        (i.position.xy + i.position.w) * 0.5; // / i.position.w;
                i.shadowCoordinates.zw = i.position.zw;
        #endif
 
        …
}

UnityLight CreateLight (Interpolators i) {
        …

        #if defined(SHADOWS_SCREEN)
         float attenuation = tex2D(
                        _ShadowMapTexture,
                        i.shadowCoordinates.xy / i.shadowCoordinates.w
                );
        #else
         UNITY_LIGHT_ATTENUATION(attenuation, 0, i.worldPos);
        #endif

        …
}
```
如果法线阴影上下颠倒，可能是因为使用的图形API不同，毕竟DirectX和OPENGL屏幕坐标的原点一个在左上角，一个在左下角。
#### 用Unity的代码实现
##### 1.传递阴影映射图的屏幕坐标
传递阴影映射图的屏幕坐标可以用定义在UnityCG的ComputeScreenPos方法。它帮我们处理好了API的不同和平台的限制。 
```
#if defined(SHADOWS_SCREEN)
     i.shadowCoordinates = ComputeScreenPos(i.position);
#endif
```
##### 2.AutoLight
AutoLight定义了三个非常有用的宏，他们是**SHADOW_COORDS**, **TRANSFER_SHADOW**, 和 **SHADOW_ATTENUATION**.

**SHADOW_COORDS**定义了定义了顶点着色器传递给片元着色器的插值结构体中阴影的字段,里面用了_ShadowCoord也就是之前编译器报的缺少的变量。  
![图十四]({{ site.url }}{{ site.baseurl }}/assets/../../assets/images/003_14.png)  

**TRANSFER_SHADOW**给上面宏定义的字段赋值，也就是做了我们之前做的事，转化阴影映射图的屏幕坐标。  


**SHADOW_ATTENUATION**在片元着色器中用这些坐标去采样阴影映射图。    
```
UnityLight CreateLight (Interpolators i) {
        …

        #if defined(SHADOWS_SCREEN)
         float attenuation = SHADOW_ATTENUATION(i);
        #else
         UNITY_LIGHT_ATTENUATION(attenuation, 0, i.worldPos);
        #endif

        …
}
```
最用，**因为UNITY_LIGHT_ATTENUATION**里面早就使用了**SHADOW_ATTENUATION**（这也是我们之前没有根据关键字区别对待出现编译错误的原因），所以我们可以直接这么写。  
![图十五]({{ site.url }}{{ site.baseurl }}/assets/../../assets/images/003_15.png)  

##### 3.使用了这些宏的代价
按照上面的方式写仍然会出现编译错误，因为上面的宏假定了我们对一些数据的命名，如果我们不根据它们假定的去做，就会出错。  
![图十六]({{ site.url }}{{ site.baseurl }}/assets/../../assets/images/003_16.png)  

**a.VexterData，顶点数据中的顶点坐标命名必须为vertex**  
**b.顶点着色器传递给片元着色器的插值数据结构中顶点坐标，必须命名为pos**  

遵守这两个命名后，shader就可以使用了。                           

### 多光源阴影
主方向光在投射阴影，但是第二个方向光没有，因为我们没有在**多编译声明(multi-compile statement)** 中添加SHADOWS_SCREEN关键字。但是SHADOWS_SCREEN只支持方向光，所以我们把已有的**多编译声明(multi-compile statement)** 换成一个就行了  
```
#pragma multi_compile_fwdadd_fullshadows
```
multi_compile_fwdadd_fullshadows包括了这些关键字
```
DIRECTIONAL 
DIRECTIONAL_COOKIE 
POINT POINT_COOKIE 
SHADOWS_CUBE 
SHADOWS_DEPTH
SHADOWS_SCREEN SHADOWS_SOFT 
SPOT
```
#### 锥形光源阴影（Spotlight Shadows）
加上之前用Unity提供的宏和方法，再加上上面的关键字，现在我们不用做任何事，锥形光源已经支持了。  

因为锥形光源是有具体位置的，且光线的方向也不是互相平行的，所以他的**阴影映射图是透视**的。因为上述特点，这些光源是不能换多个角度和位置去生成多张阴影映射图的。所以，它们也不支持**阴影层叠（shadow cascades）**。  

##### 锥形光源阴影——采样阴影映射图
**SHADOW_ATTENUATION** 宏使用了**UnitySampleShadowmap**去采集阴影映射图的。这个方法定义在UnityShadowLibrary中，被AutoLight include了.  
当使用软阴影时，会采样四次并去结果的平均值，效果虽然比不上滤波在屏幕空间的阴影上的应用，但是运行速度但是快很多。  

#### 点光源阴影
**include cginc文件的顺序**
启用点光源时，确保UnityCG或者include UnityCG的文件在AutoLight之前被include，不然会出现编译错误  “UnityDecodeCubeShadowDepth未定义”，UnityShadowLibrary依赖于UnityCG但是却没有include它（不懂为啥要这样……）

**六张阴影映射图**
因为点光源是向四周发射光线，所以采集六张阴影映射图作为cube map来采集。  

##### 点光源阴影——投射阴影
当渲染点光源阴影映射图时，unity会寻找带SHADOWS_CUBE关键字被声明的shader，
SHADOWS_DEPTH被方向光和锥形光。所以我们用multi_compile_shadowcaster这种多编译关键字来声明，它包括了一下关键字
```
SHADOWS_CUBE 
SHADOWS_DEPTH
```
因为unity不用深度cube maps图，因为支持的平台不够多，所以我们不能依赖片元的深度，取而代之的，我们将使用片元的距离。
```
#if defined(SHADOWS_CUBE)//点光源投射阴影太不同了，所以要完全不同的顶点着色器函数和片元着色器函数
 struct Interpolators {
        float4 position : SV_POSITION;
        float3 lightVec : TEXCOORD0;
};

//将顶点
Interpolators MyShadowVertexProgram (VertexData v) {
        Interpolators i;
        i.position = UnityObjectToClipPos(v.position);//顶点在裁剪空间的位置
        //光线的向量
        i.lightVec =
                mul(unity_ObjectToWorld, v.position).xyz - _LightPositionRange.xyz;
        return i;
}
        
float4 MyShadowFragmentProgram (Interpolators i) : SV_TARGET {
          float depth = length(i.lightVec) + unity_LightShadowBias.x;//将距离作为深度并添加偏移量
          //因为我们将depth的范围限制在0~1之内，所以要将距离/光照范围（其实还是会超过1，但是此时1代表了最远处）
         //w代表光照范围的倒数，因为我们要除以光照范围，所以乘以w。
        depth *= _LightPositionRange.w;/
        //UnityEncodeCubeShadowDepth会将深度储存在一张8-bit RGBA的贴图内
        return UnityEncodeCubeShadowDepth(depth);
}
#else
```
##### 点光源阴影——采样阴影映射图
硬阴影（Hard Shadow）：都一样，每张阴影映射图采样一次。
软阴影（Soft Shadow）：和锥形光阴影一样，采样四次取平均值。但是unity不支持对cube map使用滤波，所以点光源的软阴影消耗大且效果差。

## 小结
### 步骤总结
**1.投射阴影**  
创建一个专门投射阴影的pass,lightmode设为“shadowcaster”。  
这部分主要的工作就是在顶点着色函数中将顶点坐标从**模型坐标转换到裁剪空间，并支持偏移（bias）**。  
片元着色器就什么也不做  
**点光源**特殊一点：除了要提供顶点坐标，还要提供世界空间下的光线向量。  
**2.接收阴影**  
这一部分就复杂一点，大致的作用就是采集阴影映射图（shawdowmap），将裁剪空间的坐标转换到屏幕坐标。  

### 关键点：  
**1.阴影层叠（shadow cascades）**:在不提高阴影映射图和阴影具体的情况下优化阴影质量的方法，原理是在方向光的采样几个个不同角度的阴影映射图，每张阴影映射图分别负责不同距离的阴影。  
**2.点光源的阴影**：点光源阴影是最特殊的它的**投射阴影**要返回**顶点坐标**和**光线向量**，生成6张shadowMap(因为是cubemap)，**接收阴影**是采样cubemap.  
**3.Unity提供的方法：** 为了我们能方便使用，unity提供了各种方法方便生成阴影，这个就比较细节了，看一具体看上面，使用起来要注意的小点挺多的，要注意配合它们的命名还有一些cginc文件的include的顺序。看到这里不要失去耐心，复习以下挺快的。  