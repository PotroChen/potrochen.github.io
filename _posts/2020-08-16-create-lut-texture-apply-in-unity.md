---
title: "如何制作LUT贴图，并应用到unity中？"
date: 2021-08-16
categories:
  - Post
tags:
  - 渲染
  - 技术美术
---

我们介绍LUT贴图前，先从这几个方面认识LUT贴图。

* 什么是LUT贴图？
  * LUT是LookUpTable的缩写，意思是查找表，它是一张3D贴图。  
* 使用原理时什么?
  * 它的使用原理是当我们输入一个像素的颜色值(R,G,B),我们把颜色值(R,G,B)作为一个3D坐标，去采样这张贴图，并的出一个新的颜色值。  
* 它在游戏中的应用场景是什么?
  * 当我们截取了游戏里的一帧,在PS里面调整颜色(色调整体调暖,亮度更亮，blabla等等)，我们想储存PS中的调整，并作用于游戏画面的每一帧，我们就可以使用LUT贴图。(LUT是影视后期的常用技术)

## 如何制作LUT贴图？
1. 首选截取一张你想要校色的高分辨率图像  
![image01]({{ site.url }}{{ site.baseurl }}/assets/../../assets/images/如何制作LUT贴图，并应用到unity中_image01.png)
2. 在Photoshop中打开截取的图片,并创建调整图层，并从中调整参数(对比度,亮度，饱和度，白平衡等)
![image02]({{ site.url }}{{ site.baseurl }}/assets/../../assets/images/如何制作LUT贴图，并应用到unity中_image02.png)
3. 在PhotoShop中打开未经处理的LUT图片(Neutral Color LUT),将刚才的调整图层拖拽到未经处理的LUT图片(Neutral Color LUT)上，由此得到一张经过调整的LUT图片，保存它,制作LUT贴图完成？
![image03]({{ site.url }}{{ site.baseurl }}/assets/../../assets/images/如何制作LUT贴图，并应用到unity中_image03.png)
**Note** :  
* LUT图片的尺寸根据项目设置而定
* 未经处理的LUT贴图哪里来的？  
欸~我也是网上找的，应该还是蛮容易找的，这里我贴出其中两个尺寸好了  
1024x32
![image04]({{ site.url }}{{ site.baseurl }}/assets/../../assets/images/如何制作LUT贴图，并应用到unity中_image04.png)
256x16  
![image05]({{ site.url }}{{ site.baseurl }}/assets/../../assets/images/如何制作LUT贴图，并应用到unity中_image05.png)
## 怎么在Unity中使用？
将LUT贴图导入项目，TextureType为Default,TextureShap为2D，并将sRGB关掉。  
默认渲染管线：
1. 导入PostProcessing 插件包。
2. 创建PostProcessingProfile,并添加ColorGrading效果，在LDR模式下打开LUT功能并选择LUT贴图。  

URP渲染管线  
1. 创建VolumeProfile,并添加ColorLUT效果，选择LUT贴图（前提和项目设置的LUT贴图尺寸一样）

参考资料:  
[Using Lookup Tables (LUTs) for Color Grading](https://docs.unrealengine.com/4.26/en-US/RenderingAndGraphics/PostProcessEffects/UsingLUTs/)  

[CatLikeCoding/Tutorials/CustomSRP/ColorGrading](https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/)
