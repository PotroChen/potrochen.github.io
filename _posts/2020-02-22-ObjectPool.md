---
title: "对象池(Object Pool)"
date: 2020-02-22
categories:
  - Post
tags:
  - 设计模式
  - 对象池
---

对象池是游戏编程中非常常用的优化策略以及设计模式。虽然非常简单，但是太常用了，所以我觉得有必要写一篇文章来巩固自己这方面的知识点，也又必要写一个对象池来作为自己放在自己的工具集中。
### 用途
游戏中，我们常常会遇到频繁得创建和销毁大量相同对象的场景。如果我们不做任何的特殊处理，这种场景会出现**两个性能问题——大量的内存碎片以及频繁的分配内存空间**。而对象池能供完美得解决这两个问题。  
### 原理
当创建对象时，对象池将对象放入池管理的某种内存连续的数据结构中（数组或者栈等）。当不需要对象时，对象池并不销毁对象，而是将对象回收到池中，下次需要的时候再次从池中拿出来。  
因为，对象储存在内存连续的数据结构中，所以解决了**内存碎片**的问题。  
因为，对象每次用完以后就放回池中循环利用而不是再次创建和销毁，这样就解决了**频繁的内存分配和销毁**的问题。

### 一个我自己的对象池
接下来，我介绍一些我自己写的一个对象池。比较简单，主要时根据接口将相应规则定下来，以便可以快速的实现各种类的对象池的开发。
#### 定义关于对象池的接口
``` csharp
using System;

/// <summary>
/// 对象池接口
/// </summary>
/// <typeparam name="T"></typeparam>
public interface IPool<T>
{
    event Action<T> OnRecycle;
    /// <summary>
    /// 获取IPoolObject
    /// </summary>
    /// <returns></returns>
    IPoolObject<T> Get();

    /// <summary>
    /// 回收IPoolObject
    /// </summary>
    /// <param name="poolObject"></param>
    void Recycle(IPoolObject<T> poolObject);

    /// <summary>
    /// 销毁IPoolObject
    /// </summary>
    /// <param name="poolObject"></param>
    void Dispose(IPoolObject<T> poolObject);
}

/// <summary>
/// 对象池被管理单位
/// </summary>
/// <typeparam name="T"></typeparam>
public interface IPoolObject<T>
{
    /// <summary>
    /// 对象池的引用
    /// </summary>
    /// <value></value>
    IPool<T> Pool { get; }

    /// <summary>
    /// 内容
    /// </summary>
    /// <value></value>
    T Content { get; }

    /// <summary>
    /// 回收自己
    /// </summary>
    /// <param name="poolObject"></param>
    void Recycle();

    /// <summary>
    /// 销毁自己IPoolObject
    /// </summary>
    /// <param name="poolObject"></param>
    void Dispose();
}
```
#### 根据接口实现一个管理GameObject的对象池类
##### GameObjectPool
``` csharp
using System.Collections.Generic;
using UnityEngine;
using System;

/// <summary>
/// GameObject对象池
/// </summary>
public class GameObjectPool : IPool<GameObject>
{
    public event Action<GameObject> OnRecycle;

    public int MaxSize { get; set; }
    public int TotalCount { get { return objectsInPools.Count + objectsPoped.Count; } }
    private List<IPoolObject<GameObject>> objectsInPools = new List<IPoolObject<GameObject>>();
    private List<IPoolObject<GameObject>> objectsPoped = new List<IPoolObject<GameObject>>();
    private GameObject prefab;

    private GameObject poolRoot;
    public GameObjectPool(GameObject prefab, int maxSize)
    {
        this.prefab = prefab;
        this.MaxSize = maxSize;
        poolRoot = new GameObject("GameObjectPool");
    }

    public void WarmUp()
    {
        if (TotalCount < MaxSize)
        {
            GameObject content = GameObject.Instantiate(prefab);
            content.transform.SetParent(poolRoot.transform);
            content.SetActive(false);
            IPoolObject<GameObject> poolObject = new GameObjectPoolObject(this, content);
            objectsInPools.Add(poolObject);
        }
    }

    public IPoolObject<GameObject> Get()
    {
        IPoolObject<GameObject> poolObject = null;

        if (objectsInPools.Count == 0)
        {
            GameObject content = GameObject.Instantiate(prefab);
            poolObject = new GameObjectPoolObject(this, content);

            poolObject.Content.SetActive(true);

            objectsPoped.Add(poolObject);
        }
        else
        {
            int lastIndex = objectsInPools.Count - 1;
            poolObject = objectsInPools[lastIndex];

            poolObject.Content.SetActive(true);

            objectsInPools.RemoveAt(lastIndex);
            objectsPoped.Add(poolObject);
        }
        poolObject.Content.transform.SetParent(null);
        return poolObject;
    }

    public void Recycle(IPoolObject<GameObject> poolObject)
    {
        if (poolObject.Pool != this)
            return;

        OnRecycle?.Invoke(poolObject.Content);
        if (TotalCount > MaxSize)
        {
            poolObject.Dispose();
        }
        else
        {
            objectsPoped.Remove(poolObject);
            objectsInPools.Add(poolObject);

            poolObject.Content.transform.SetParent(poolRoot.transform);
            poolObject.Content.SetActive(false);
        }
    }

    public void Dispose(IPoolObject<GameObject> poolObject)
    {
        if (poolObject.Pool != this)
            return;

        GameObject.DestroyImmediate(poolObject.Content);

        if (objectsPoped.Contains(poolObject))
            objectsPoped.Remove(poolObject);

        if (objectsInPools.Contains(poolObject))
            objectsInPools.Remove(poolObject);
    }
}
```
##### GameObjectPoolObject
``` csharp
using UnityEngine;

/// <summary>
/// GameObject对象池被管理对象
/// </summary>
public class GameObjectPoolObject : IPoolObject<GameObject>
{
    public IPool<GameObject> Pool { get; set; }
    public GameObject Content { get; set; }

    public GameObjectPoolObject(GameObjectPool pool, GameObject content)
    {
        Pool = pool;
        Content = content;
    }

    public void Recycle()
    {
        Pool.Recycle(this);
    }

    public void Dispose()
    {
        Pool.Dispose(this);
    }
}
```
##### MaxSize 
关于我的这个对象池，有一个MaxSize.定义对象池可容纳的最大数量。关于超过MaxSize的处理，有以下几个策略。    
1.无法再次创建，返回空对象。这种比较严格，但是一些音效，比如很多脚步声，没有的话玩家也不会注意到的。  
2.强制回收一个已有的对象。同样也要用在玩家，不会注意到的地方，比如声音，粒子什么的。  
3.增加池的大小。这样，就不会担心创建不出对象的问题了，但是要考虑要不要缩回去。（我的GameObject当超出MAXSIZE的时候，回收对象，不会再放回池中，会直接被销毁，知道池里的对象小于MAXSIZE位置）

Github仓库网址：  
[https://github.com/PotroChen/ObjectPool](https://github.com/PotroChen/ObjectPool)
参考：  
[游戏编程模式](https://gpp.tkchu.me/state.html)