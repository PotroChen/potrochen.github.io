---
title: "GPU Instancing For Skeletal Animation"
date: 2022-05-24
categories:
  - Post
tags:
  - 渲染
  - 优化
---
## 一.需要的前置知识
### 1.什么是GPU Instancing?
这里简单介绍一下，Unity官方文档链接：[GPU Instancing]([https://docs.unity3d.com/Manual/GPUInstancing.html)
GPU Instancing是一种优化方法，用一个DrawCall去渲染多个物体——他们使用相同的材质，他们的Mesh拷贝自同一份Mesh资源。每一份Mesh的拷贝称为Instancing.这种优化方法，经常用于渲染场景重复多次出现的事物（比如植被，树木等）。  
GPU Instancing渲染的每个Instancing可以拥有不同的材质属性比如Color,Scale等。（通过MaterialPropertyBlock）

### 2.骨骼动画与蒙皮
[[Unity3d杂记]骨骼蒙皮动画](https://zhuanlan.zhihu.com/p/87583171)

## 二.为什么Skeletal Animation不能使用GPU Instancing?
其实是可以的，但是Unity默认的方案SkinnedMeshRenderer不行。  
SkinnedMeshRenderer，每个角色的顶点数据是运行时计算好蒙皮（Skinning），再分别传递到GPU的。。不能像MeshRenderer一样，所有物体使用同一份顶点数据。

## 三.如何让播放Skeletal Animation的物体，能够使用GPU Instancing这个优化策略呢？
我看了一些资料发现有两种方案:
**方案一**:跳过蒙皮,直接将角色播放动画时的顶点信息储存在贴图中，播放动画时，让Shader在顶点函数获得贴图中的顶点位置。
文章:[https://www.cnblogs.com/murongxiaopifu/p/7250772.html](https://www.cnblogs.com/murongxiaopifu/p/7250772.html)
仓库:[https://github.com/chenjd/Render-Crowd-Of-Animated-Characters](https://github.com/chenjd/Render-Crowd-Of-Animated-Characters)
缺点:贴图内存过大，并且一个动画就是一张贴图。模型顶点数小还行，模型顶点数稍微大一点，就可能超过unity可创建的最大贴图尺寸。

**方案二（GPUSkinning）**:将所有骨骼动画的每帧的骨骼在模型空间的TRS矩阵储存在贴图中,播放动画时，让Shader在顶点函数进行蒙皮计算获得蒙皮后的顶点位置。
文章:[https://zhuanlan.zhihu.com/p/36896547](https://zhuanlan.zhihu.com/p/36896547)
仓库:[https://github.com/Unity-Technologies/Animation-Instancing](https://github.com/Unity-Technologies/Animation-Instancing)
这个方案因为只记录骨骼的的TRS，所以骨骼数量和动画长度决定了贴图大小，骨骼数量远没有顶点数量这么多，所以贴图数量小很多，甚至所有动画的信息都可以记录在一张贴图中。

## 四.GPUSkinning过程简述
1.将动画信息和模型信息记录在一个GPUAnimation(ScriptableObject)内

``` csharp
class GPUSkinningAnimation
{
	GPUSkinningBone[] bones;
	GPUSkinningClip[] clips;
	
}

//骨骼信息
class GPUSkinningBone
{
	string name;
	Transform transform;//骨骼对应Gameobject上的Transform节点
	Matrix4x4 bindpose;//模型空间到骨骼空间的转换矩阵
	int parentBoneIndex = -1;//父骨骼
	int[] childrenBonesIndices = null;//子骨骼
}

//动画信息
class GPUSkinningClip
{
	string name;
	float length;//Clip 动画长度
	int fps;
	WarpMode warpMode;//Once or Loop
	int pixelSegmentation;//像素索引，这段动画信息在贴图中的起点位置
	GPUSkinningFrame[] frames = null;//帧信息
}

//帧信息
public GPUSkinningFrame
{
	Matrix4x4[] matrices = null;//骨骼在模型空间的TRS矩阵
}
```  

2.将所有Clips的每帧骨骼模型空间下TRS矩阵储存在贴图中

a.首先，计算每一帧动画中所有骨骼模型空间下的TRS矩阵,关键代码
```csharp
//计算单帧所有骨骼的模型空间TRS
GPUSkinningBone[] bones = samplingAniamtion.bones;
for (int i = 0; i < bones.Length; ++i)
{
	GPUSkinningBone currentBone = bones[i];

	frame.matrices[i] = currentBone.bindpose;
	do
	{
		Matrix4x4 mat = Matrix4x4.TRS(currentBone.transform.localPosition, 											
									  currentBone.transform.localRotation, 	
									  currentBone.transform.localScale);
		frame.matrices[i] = mat * frame.matrices[i];
		if (currentBone.parentBoneIndex == -1)
		{
			break;
		}
		else
		{
			currentBone = bones[currentBone.parentBoneIndex];
		}
	}
	while (true);
}
```

b.将所有动画每帧中的所有骨骼的TRS矩阵储存在贴图中
```csharp
//计算贴图长宽
for (int index = 0; index < clips.Length; ++index)
{
	GPUSkinningClip clip = clips[index];
	clip.pixelSegmentation = pixelCount;

	GPUSkinningFrame[] frames = clip.frames;
	int frameCount = frames.Length;
	pixelCount += animation.bones.Length * 3/* 3 x 4个通道,frame.matrices 中的 	3x4 */ * frameCount;
}
```
```csharp
public Texture2D CreateAnimationMap(GPUSkinningAnimation animation)
{
	Texture2D texture = new Texture2D(animation.textureWidth, animation.textureHeight, TextureFormat.RGBAHalf, false, true);
	Color[] pixels = texture.GetPixels();
	int pixelIndex = 0;
	for (int clipIndex = 0; clipIndex < animation.clips.Length; ++clipIndex)
	{
		GPUSkinningClip clip = animation.clips[clipIndex];
		GPUSkinningFrame[] frames = clip.frames;
		int numFrames = frames.Length;
		for (int frameIndex = 0; frameIndex < numFrames; ++frameIndex)
		{
			GPUSkinningFrame frame = frames[frameIndex];
			Matrix4x4[] matrices = frame.matrices;
			int numMatrices = matrices.Length;
			for (int matrixIndex = 0; matrixIndex < numMatrices; ++matrixIndex)
			{
				Matrix4x4 matrix = matrices[matrixIndex];
				pixels[pixelIndex++] = new Color(matrix.m00, matrix.m01, matrix.m02, matrix.m03);
				pixels[pixelIndex++] = new Color(matrix.m10, matrix.m11, matrix.m12, matrix.m13);
				pixels[pixelIndex++] = new Color(matrix.m20, matrix.m21, matrix.m22, matrix.m23);
			}
		}
	}
	texture.anisoLevel = 0;
	texture.filterMode = FilterMode.Point;
	texture.SetPixels(pixels);
	texture.Apply();

	return texture;
}

```
c.播放动画的shader代码
``` hlsl
//通过在顶点函数调用这个方法，获取蒙皮计算后的顶点位置
float4 GetProcessedPositionOSFromAnimationMap(float4 positionOS,float4 uv2,float4 uv3)
{
	//通过MaterialPropertyBlock,告知Shader播放哪个动画，哪一帧
	//蒙皮计算
	float frameStartIndex = GetFrameStartIndex();
	
	float4x4 bone1TRS = GetBoneTRSMatrix_OS(frameStartIndex,uv2.x);
	float bone1Weight = uv2.y;

	float4x4 bone2TRS = GetBoneTRSMatrix_OS(frameStartIndex,uv2.z);
	float bone2Weight = uv2.w;

	float4x4 bone3TRS = GetBoneTRSMatrix_OS(frameStartIndex,uv3.x);
	float bone3Weight = uv3.y;

	float4x4 bone4TRS = GetBoneTRSMatrix_OS(frameStartIndex,uv3.z);
	float bone4Weight = uv3.w;

	float4 processedPositionOS = mul(bone1TRS, positionOS) * bone1Weight + mul(bone2TRS, positionOS) * bone2Weight + mul(bone3TRS, positionOS) * bone3Weight + mul(bone4TRS, positionOS) * bone4Weight;

	return processedPositionOS;
}
```
d.写一些应用层代码，通过MaterialPropertyBlock改变材质属性控制Shader播放哪一个动画的哪一帧。