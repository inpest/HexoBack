---
title: ET框架探索之AssetBundle
date: 2020-5-28 15:48:15
categories: 
- Unity
- ET框架学习笔记
tags :
- ET
---
<!-- more -->
# 什么是Asset Bundle？
AssetBundle是将资源使用Unity提供的一种用于存储资源的压缩格式打包后的集合，简单理解就是一种压缩包，我们一般称他为AB包。

我们先来看下官方的解释

>What’s in an AssetBundle?

>“AssetBundle” can refer to two different, but related >things.

>First is the actual file on disk. This is called the AssetBundle archive. The AssetBundle archive is a container, like a folder, that holds additional files inside it. These additional files consist of two types:

>- A serialized file, which contains your Assets broken out into their individual objects and written out to this single file.
>- Resource files, which are chunks of binary data stored separately for certain Assets (Textures and audio) to allow Unity to efficiently load them from disk on another thread.

这个内容大致是讲述了AssetBundle里面保存的内容，分为两种一种是serialized file(序列化文件)和Resource files(资源文件),序列化文件就是我们平常保存的预制体(.prefab)，资源文件代表声音、图片这类二进制形式存储的资源文件，我们将他们分别打包成一个或多个AssetBundle包(后面我就称之为AB包)，然后上传到服务器，然后玩家通过下载服务器AB包进行游戏资源下载并开始玩。

值得一提的是在Unity不断的更新迭代中，Unity推出了一种对基于AssetBundle架构延伸出来的高级管理系统—`可寻址资源系统(Addressable Assets)`,这里简单说一下，新的可寻址资源概括来说是将手动的AssetBundle流程变为自动化，方便了开发流程，而且提供包体裁切、包体切换、包体拆离、模拟服务器加载资源包流程、查看资源使用情况的工具等强大的功能，我准备用单独一篇文章来介绍他。

# 简述AB包加载流程
1. 首先开发者先要对资源进行打包，但是我们需要判断有些常驻内存的资源实际是不需要动态加载的。所以大致资源非为两大类:
    - 需要打入AB包进行动态加载的
    - 不需要动态加载直接在场景中放着就行的

    了解完后，我们发现我们只要注重AB包的资源就行，那怎么区分呢，Unity为我们准备了一种标记，我们可以将要打包的资源，先打上一个标记，然后通过脚本来找到有标记的资源然后进行打包。

    ![](https://s1.ax1x.com/2020/05/19/Y4suPU.png)
    
    我们发现标记可以指定两部分，前面为AB包的名称，后面是他的后缀，这里我们可以随便起名，比如我就叫wall.ab这样这个球体资源就标记完成了
2. 标记完成后我们就可以编写打包的脚本了
    ```csharp
        //这个代表可以在Assets菜单下添加一个名为"Build AssetBundles"的按钮，我们按下按钮就会触发这个函数
        [MenuItem("Assets/Build AssetBundles")]
        public static void BulidAllAssetBundles()
        {
            //这里指定刚才标记过的球体的预制体的路径
            string dir = "AssetBundles";
            //如果没有这个目录就创建一个
            if (!Directory.Exists(dir))
                Directory.CreateDirectory(dir);
            //这里通过指定的路径、压缩算法、打包的平台就可以通过打包流水线进行打包了
            BuildPipeline.BuildAssetBundles(dir, BuildAssetBundleOptions.None, BuildTarget.StandaloneWindows64);
        }
    ```
    压缩算法有三种:
    - None LZMA算法 包小 加载慢
    - UncompressedAssetBundle 不压缩 包大 加载快
    - ChunkBaseedCompression LZ4算法 压缩率比LZMA高 但加载快 不用解压全部
3. 然后我们就可以将打包出来的资源文件上传到服务器上(FTP)了
    
    我们先观察一下打包之后的文件结构
    ![](https://s1.ax1x.com/2020/05/19/Y463Ax.png)

    可以发现除了描述文件.manifest和资源文件意外，还多出了AssetBundles和AssetBundles.manifest,这两个文件作用是记录所有AB包的信息。

    wall.ab用记事本打开后后是一堆二进制数字，我们再打开wall.ab.manifest，这是对wall.ab这个AB包的描述文件我们打开后看到:
    ```java
    //这是AB包的版本号
    ManifestFileVersion: 0
    //这是在进行AB包网络传输时的校验码(循环冗余)有兴趣的可以去网上了解一下计算机网络的知识
    CRC: 628802961
    //哈希值
    Hashes:
    AssetFileHash:
        serializedVersion: 2
        Hash: 2b9fbe7f986add5e812adefc605b247b
    TypeTreeHash:
        serializedVersion: 2
        Hash: f84699563d2dd09d978bf6c2e03adf13
    HashAppended: 0
    ClassTypes:
    - Class: 1
    Script: {instanceID: 0}
    - Class: 4
    Script: {instanceID: 0}
    - Class: 21
    Script: {instanceID: 0}
    - Class: 23
    Script: {instanceID: 0}
    - Class: 33
    Script: {instanceID: 0}
    - Class: 43
    Script: {instanceID: 0}
    - Class: 48
    Script: {instanceID: 0}
    - Class: 65
    Script: {instanceID: 0}
    //这里就是记录当前AB包中有哪些资源的地方了
    Assets:
    - Assets/Cube.prefab
    //这里是如果你将贴图等信息单独打成了一个AB包，这个AB包又依赖于贴图的AB包，那么这里会记录依赖的信息
    Dependencies: []
    ```
4. 加载AB包

    这里加载的方式有很多我先介绍一种(假设资源在本地)，直接上代码:
    ```csharp
    void Start()
    {
        //获得正方体资源
        AssetBundle ab = AssetBundle.LoadFromFile("AssetBundles/scenes/cubewall.ab");
        //加载指定名字资源 参数为预制体的名字
        GameObject wallPrefab= ab.LoadAsset<GameObject>("cubeWall");
        Instantiate(wallPrefab);

        //加载一个AssetBundle中所有资源
        object[] objs = ab.LoadAllAssets();
        foreach (var item in objs)
        {
            Instantiate(item as GameObject);
        }
    }
    ```
至此一个简单AB包加载流程就结束了。
# 待完善。。。
