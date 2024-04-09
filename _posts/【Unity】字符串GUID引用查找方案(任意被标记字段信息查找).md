---
title: "【Unity】字符串GUID引用查找方案(任意被标记字段信息查找)"
date: 2019-12-28
categories:
  - Post
tags:
  - UnityEditor
---
相信很多Unity游戏开发项目都会采用一种资源引用方案。
写一个类,类里面有一个字符串字段，记录了资源的GUID(Path)。游戏运行时要加载资源的时候，再通过这个GUID(或者Path)异步或同步加载这个资源。例如：
```
public SerializedReference
{
	public string guid;
	#if UNITY_EDITOR
	[NonSerialized]
	public Asset editorAsset;
	#endif
}

public SerializedReferenceDrawer
{
	/*用ObjectField绘制，使其他项目内的小伙伴操作上无感*/
}
```
这样做的**好处**就是不用在代码里硬编码写路径，并且一大堆可能用不到的依赖资源上来就被加载到内存中。
**坏处**就是项目内容量变大后，不知道哪些资源已经废弃没有被引用到了。总不能每次新增资源(ScriptableObject)声明了这个类，你要在一个收集引用的类里面添加查找这个类引用的逻辑吧(类似的逻辑还有多语言)……
现在就要提供一个方案处理坏处，**不管是预制体的Component,ScriptableObject,又或者是隐藏在ScriptableObject里面的Subasset,还是Scene里的场景资源(场景要特殊处理以下)，只要你声明了这个类并序列化了下来，都能用一个逻辑提取出来**。

这个方案需要对以下两个类以及Unity如何序列化资源有一定基础性的认知(经常写编辑器的同学应该会比较了解)
* [SerializedObject](https://docs.unity3d.com/ScriptReference/SerializedObject.html)  
* [SerializedProperty](https://docs.unity3d.com/ScriptReference/SerializedProperty.html)  

接下来，开始介绍方案(首先我会讲如何获得一个资源的依赖资源，后面会讲如果**加快查找速度**的问题)
### 1.如何用同一种方法获得Prefab的Child,Component,TimlineAsset的TrackAsset以及所有SubAsset？

首先，这个上面这个小标题里面的**问题本身就是不准确的**。因为Prefab的子节点,Component以及子节点的Component,TimelineAsset底下的TrackAsset和ClipAsset以及所有自定义ScriptableObject都是SubAsset.
有了上面这个认知，所以以上所有东西都可以用这个Unity提供的这个接口获得。
```
AssetDatabase.LoadAllAssetsAtPath(mainAssetPath)
```
通过这个接口，我们可以获得所有Prefab以及Prefab里所有Component和脚本对象资源以及隐藏对象了。
但是，还有场景里GameObject以及他们的Component.
这里我们就要特殊处理以下了
```
if(path is 场景)
{ 
	List<UnityEngine.Object> rtn = new List<UnityEngine.Object>();
	var scene = EditorSceneManager.OpenScene(mainAssetPath, OpenSceneMode.Single);
	var rootGos = scene.GetRootGameObjects();
	foreach (var rootGo in rootGos)//GameObject本身是不需要的
	{
		var allComponents = rootGo.GetComponentsInChildren<Component>(true);
		rtn.AddRange(allComponents);
	}
	return rtn.ToArray();
}
else
{
	return assetDatabase.LoadAllAssetsAtPath(mainAssetPath);
}
```
我们将这一步抽出一个方法
```
UnityEngine.Object[] GetAllSubAssets(string path);
```
### 2.如何用同一种方法获得各种不同脚本对象(Component或者ScriptableObject等)中我们想要的特定类型(或被我们标记了特定属性的类)？
答案就是[SerializedObject](https://docs.unity3d.com/ScriptReference/SerializedObject.html)和[SerializedProperty](https://docs.unity3d.com/ScriptReference/SerializedProperty.html) 然后加上一个Unity引擎的Internal方法。不管你的项目序列化设置是选择了Force Binary还是Force Text(项目太大了可以选择Force Binary加快序列化的速度)都可以将Asset转化成SerializedObject然后通过SerializedProperty一视同仁的获取序列化信息。
**a.获取对象(Component或者ScriptableObject等)，并获取信息**
```
var allSubAssets = GetAllSubAssets(assetPath);
foreach(var subAsset in allSubAssets)
{
	SerializedObject so = new SerializedObject(subAsset);//获得SerializedObject
	var iterator = so.GetIterator();
	var propertyType = iterator.propertyType;
	while (iterator.Next(EnterChildren(propertyType)))//获得所有SerializedProperty，并筛选，并读取信息
	{
		propertyType = iterator.propertyType;
		```
		```
	}
}

static bool EnterChildren(SerializedPropertyType propertyType)//
{   //其他类型String,Vector4之类的没必要再EnterChild
	return propertyType == SerializedPropertyType.Generic
		|| propertyType == SerializedPropertyType.Character
		|| propertyType == SerializedPropertyType.ManagedReference;
}
```
**b.通过SerializedProperty获得FieldInfo,筛选信息**
这时就要依靠Unity源码中的一个Internal方法
```
//ScriptAttributeUtility.cs
internal static FieldInfo GetFieldInfoAndStaticTypeFromProperty(SerializedProperty property, out Type type)
```

### 3.全局引用统计的效率问题如何解决呢？
通过1和2两点，我们可以解决无法找到一个资源的脚本对象上的SerializedReference类型(自定义)引用，但问题是要统计全局的资源引用/被引用数量，这个统计一次效率很慢，这是我们解决的问题。
**增量计算**
假如我们对整个项目的资源，统计了一次引用关系，并缓存了下来，缓存下来的数据格式,如下面的代码。
```
class CachedReferencedData
{
	Dictionary[string,AssetDepData] hashToAssetDepData;
}

class AssetDepData
{
	string guid;
	string hash;
	string[] dependencies;
}
```
第二次统计的话，没必要把那些没修改过的资源再统计一次吧（没修改过，引用关系也不会变）
我们可以借助Unity的接口来判断，资源有没有改变过，然后更新修改过的数据
```
//资源没被修改过，这个hash就不会变
AssetDatabase.GetAssetDependencyHash(path)
```

**最后**，至此如何查找自定义引用类型的方法就写完了
PS:该方法也特别适合在本地化模块使用，定义一个自定义String类型,里面加入id,然后用相同的方法统计所有脚本对象上需要本地化的文本内容.