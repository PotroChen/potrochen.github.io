---
title: "002-Bumpiness(凹凸)——CatLikeCoding学习笔记"
date: 2020-5-15
categories:
  - Post
tags:
  - 渲染
  - CatLikeCoding学习笔记
---
# 002-Bumpiness(凹凸)——CatLikeCoding学习笔记
## HeightMaps(高度图)
将高度信息储存在一张贴图中，我们就可以逐片元得去获得法线信息，而不是逐顶点了（太多顶点造成太多得消耗）。

### 如果通过高度信息获得法线信息呢？
#### 先从二维角度看（获取斜率）
如果我们可以得到通过方程f(u) = h可以获得一条不规则曲线上的任意一个点的高度，我们可以通过夹逼法获得任意一点的**斜率(导数，我们这里也可以当作切线用)**。
![图一]({{ site.url }}{{ site.baseurl }}/assets/../../assets/images/002_1.png)  

当然，如果两点之间如果是凹起或者凸起，都会影响结果的准确性，δ越小，误差就越小。最后，我们得到

![图二]({{ site.url }}{{ site.baseurl }}/assets/../../assets/images/002_2.png)  

上面的公式还是有一定偏差，因为我们想要u点的斜率最好最好取u+δ/2和u-δ/2，由此可得  
![图三]({{ site.url }}{{ site.baseurl }}/assets/../../assets/images/002_3.png)   

#### 再从三维角度看
我们可以通过方程 f(u ,v) = h,我们就可以通过相同方式获得平面的u轴上的切线和在v轴上的切线。  
我们都知道**两点定一直线**和**两条向量可以确定一个平面**。  
我们获得了平面在 u 轴上的切线 t1 和 v 轴上的切线 t2.我们再将两者叉乘就可以获得我们的法线了。  
``` 
n = cross(t1,t2) 
```
#### 回到shader
我们的高度图就是我们的f(u,h)=h  
我们用shader代码来表现就是
``` 
        float2 du = float2(_HeightMap_TexelSize.x * 0.5, 0);
        float u1 = tex2D(_HeightMap, i.uv - du);
        float u2 = tex2D(_HeightMap, i.uv + du);
        float3 tu = float3(1, u2 - u1, 0);

        float2 dv = float2(0, _HeightMap_TexelSize.y * 0.5);
        float v1 = tex2D(_HeightMap, i.uv - dv);
        float v2 = tex2D(_HeightMap, i.uv + dv);
        float3 tv = float3(0, v2 - v1, 1);

        i.normal = cross(tv, tu);
        i.normal = normalize(i.normal);
```

## 法线映射（Normal Mapping）
在用高度图中，我们不得不多次采样并且用夹逼法进行运算，这看上去像是一种浪费，以为法线信息并不会动态改变，我们为什么还要每帧去算呢？于是就有了法线贴图。

将高度图导入unity，并将TextureType改成Normal map，unity会自动帮我们生成法线贴图。原来的高度图仍然存在，不过unity将在内部使用那张法线贴图。  
（unity把y储存在z中，也就是说y和z互换了）

### DXT5nm的解码
虽然，在预览模式我们看法线贴图是RGB的编码格式，但是实际上unity使用的是DXT5nm.

DXT5nm编码**只储存了法线的X和Y元素**。Y元素储存在G通道，X元素储存在A通道，R和B通道没有被用到。

我们用**勾股定理获得另外一个元素**：
```
i.normal.z = sqrt(1 - dot(i.normal.xy, i.normal.xy));
```
因为精确度的限制，dot(i.normal.xy, i.normal.xy)可能会超过边界超过1或者小于0 。所以我们要确保这种情况不发生:
```
i.normal.z = sqrt(1 - saturate(dot(i.normal.xy, i.normal.xy)));
```

### Scaling Bumpiness（缩放凹凸）
通过添加属性字段，我们可以做到用一个缩放系数来控制法线的凹凸程度。
```
Properties {
    _BumpScale ("Bump Scale", Float) = 1
}


void InitializeFragmentNormal(inout Interpolators i) {
        ……
        i.normal.xy *= _BumpScale;
        ……
}
```

### Unity提供了方法上面三件事都做了！！
UnityStandardUtils里含有UnpackScaleNormal方法。它做了采样贴图，正确的方式解码法线贴图以及缩放法线。
```
void InitializeFragmentNormal(inout Interpolators i) {
        i.normal = UnpackScaleNormal(tex2D(_NormalMap, i.uv), _BumpScale);
        i.normal = i.normal.xzy;
        i.normal = normalize(i.normal);
}
```
UnpackScaleNormal会在内部使用UNITY_NO_DXT5nm自动判断是否使用了DXT5nm压缩格式。

## Blending Normals(融合法线)
当我们有多张法线贴图时（比如说还有一张细节贴图和配合这张细节贴图的法线贴图），我们需要融合法线。
**比较常见的想法是，把两个法线向量加起来取平均数，但是事实上效果上并不好**。
```
void InitializeFragmentNormal(inout Interpolators i) {
        float3 mainNormal =
                UnpackScaleNormal(tex2D(_NormalMap, i.uv.xy), _BumpScale);
        float3 detailNormal =
                UnpackScaleNormal(tex2D(_DetailNormalMap, i.uv.zw), _DetailBumpScale);
        i.normal = (mainNormal + detailNormal) * 0.5;
        i.normal = i.normal.xzy;
        i.normal = normalize(i.normal);
}
```
我们合并两张贴图的高度信息，比起取平均数，取它们的和才是更加正确的选择。之前，我们法线向量是通过标准化
![图四]({{ site.url }}{{ site.baseurl }}/assets/../../assets/images/002_4.png)  
获得的。我们的法线贴图里也有这些信息，Y和Z元素需要对调了一下。所以应该是  
![图五]({{ site.url }}{{ site.baseurl }}/assets/../../assets/images/002_5.png)  
法线贴图的信息是经过标准化的，所以应该是有个系数使它的长度为一，所以我们得到下面这个向量。（s代表任意的系数）  
![图六]({{ site.url }}{{ site.baseurl }}/assets/../../assets/images/002_6.png)  
通过上面的向量我们可以看出，当我们除以Z的时候，我们可以把导数单独提取出来（只有Z是0的时候会失败，先不管）。我们有了导数，我们可以把他们加起来，便获得了下面这个向量（还没有被标准化）  
![图七]({{ site.url }}{{ site.baseurl }}/assets/../../assets/images/002_7.png)  
可得以下代码：
```
void InitializeFragmentNormal(inout Interpolators i) {
        float3 mainNormal =
                UnpackScaleNormal(tex2D(_NormalMap, i.uv.xy), _BumpScale);
        float3 detailNormal =
                UnpackScaleNormal(tex2D(_DetailNormalMap, i.uv.zw), _DetailBumpScale);
        i.normal =
                float3(mainNormal.xy / mainNormal.z + detailNormal.xy / detailNormal.z, 1);
        i.normal = i.normal.xzy;
        i.normal = normalize(i.normal);
}
```
反正要标准化，我们直接乘以MzDz来去除分母，得出以下代码
```
void InitializeFragmentNormal(inout Interpolators i) {
        float3 mainNormal =
                UnpackScaleNormal(tex2D(_NormalMap, i.uv.xy), _BumpScale);
        float3 detailNormal =
                UnpackScaleNormal(tex2D(_DetailNormalMap, i.uv.zw), _DetailBumpScale);
        i.normal = float3(mainNormal.xy + detailNormal.xy, mainNormal.z * detailNormal.z);
        i.normal = i.normal.xzy;
        i.normal = normalize(i.normal);
}
```
### Unity提供了方法让我们直接融合法线
UnityStandardUtils里含有BlendNormals方法.
里面是这样子得
```
half3 BlendNormals (half3 n1, half3 n2) {
        return normalize(half3(n1.xy + n2.xy, n1.z * n2.z));
}
```

## 切线空间
为了将凹凸转换到世界空间，我们必须定义一个切线空间（以U,V和N（法线）3个向量作为轴的空间）。

我们早就可以获得法线向量'N',所以我们只再获得一个向量就可以通过获得第三个。

另外一个向量作为一部分的模型顶点数据被提供——也就是**切线向量（T）**。按照惯例，这个向量定义了U轴，指向右边。

第三个向量B也就是**副切线（bitangent）** 或者叫做副法线（binormal,Unity把它叫做副法线）。这个向量定义了V轴，指向前方。正常获取方法是“B = N X T”.但是这种方法只会产生一个向量指向后方，而不是前方。所以，我们还要乘以-1.这个-1会被作为T的第四个元素。

由这三个向量组成的空间我们称之为切线空间或者TBN space。

### 切线空间在Shader中的运用
在顶点着色器中，我们可以通过TANGENT语义来标记字段来获得模型切线。
```
struct VertexData {
        float4 position : POSITION;
        float3 normal : NORMAL;
        float4 tangent : TANGENT;
        float2 uv : TEXCOORD0;
};
```
我们需要将这个模型切线转换到世界空间，当然需要转换XYZ部分，用UnityCG的UnityObjectToWorldDir.
```
Interpolators MyVertexProgram (VertexData v) {
        Interpolators i;
        ……
        i.tangent = float4(UnityObjectToWorldDir(v.tangent.xyz), v.tangent.w);
        ……
        return i;
}
```
我们要小心不要用采样获得的法线向量替换掉模型的法线向量。采样获得的法线向量是存在于切线空间的，所以我们要将它们分开。
```
void InitializeFragmentNormal(inout Interpolators i) {
        float3 mainNormal =
                UnpackScaleNormal(tex2D(_NormalMap, i.uv.xy), _BumpScale);
        float3 detailNormal =
                UnpackScaleNormal(tex2D(_DetailNormalMap, i.uv.zw), _DetailBumpScale);
        float3 tangentSpaceNormal = BlendNormals(mainNormal, detailNormal);
        tangentSpaceNormal = tangentSpaceNormal.xzy;

        float3 binormal = cross(i.normal, i.tangent.xyz) * i.tangent.w;
}
```
我们将采样法线贴图获得的法线向量从切线空间转换到世界空间。
```
      float3 binormal = cross(i.normal, i.tangent.xyz) * i.tangent.w;

        i.normal = normalize(
                tangentSpaceNormal.x * i.tangent +
                tangentSpaceNormal.y * binormal +
                tangentSpaceNormal.z * i.normal
                );
```
还有一个小细节，当一个模型缩放为（-1，1，1）.这意味着模型被镜像了，我们不得不反转副切线为了让切线空间正确。UnityShaderVariables的unity_WorldTransformParams变量帮我们做这件事，当我们需要反转的时候，这个变量的第四元素为-1.
```
float3 binormal = cross(i.normal, i.tangent.xyz) *
                (i.tangent.w * unity_WorldTransformParams.w);
```
### 同步切线空间（Synched Tangent Space）
当模型师创建一个模型，最有用的方法是先创建一个高模，然后在游戏里用低模，然后把细节烘焙到各种贴图中。    

高模的法线被烘焙进法线贴图，这一步是通过将法线从世界空间转化到切线空间做到的。**只要所有转化算法和切线空间是一样的，所以都会是正确的**。所以我们必须保证你的法线贴图生成器，Unity的模型导入处理，和shader都是同步的，这一步叫做**同步切线空间工作流**。

自从版本5.3，Unity使用mikktspace确保法线生成器也是用mikktspace。

### 在哪计算副切线
我们可以考虑在哪计算副切线，可以在顶点着色器还是在片元着色器。在顶点着色器的优点是不用每个像素都计算一遍，缺点是多一个插值字段要传送，两种都可以，没有明确的定论哪种好，看实际情况好了
。


