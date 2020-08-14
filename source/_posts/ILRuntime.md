---
title: ET框架探索之ILRuntime与游戏热更新
date: 2020-5-15 15:48:15
categories: 
- Unity
- ET框架学习笔记
tags :
- ET
---
# 前言
又到了学习ET的前置知识的时间，最近我得加快脚步了，这一个月内需要将ET框架前置的基本知识掌握，并可以熟练使用ET框架进行小型GamePlay的开发，并在自己能力范围内深入研究下ET的源码为以后做准备，这也是我近期树立起的小目标把，那么废话不多说今天的主题是ILRuntime
<!-- more -->
# 游戏热更新简述
## 什么是游戏热更新
一般的单机游戏,都是玩家下载之后直接可以玩的，因为单机游戏一般很少或者不会牵扯到更新的问题，所以单机游戏没有热更新这个概念，当开发者更新的时候直接会先删除原来的游戏包体，然后将更新后的整个包体在下载下来或者直接覆盖原来的内容进行安装，对于单机来说一般不会频繁更新，所以用户也就很少在意这些。而网络游戏这种方式就不太合适了，比如逆水寒这种即便是咸鱼版(最简客户端)也需要23G之多的空间，而网络游戏更新的频率极高基本每个星期都要更新几次，也不免有的时候出现严重漏洞进行连着修复的时候，这时候要解决这种问题就要用到<font color=red>**热更新技术**</font>。
## 热更新案例(个人分析可略过)
我们玩过游戏的都有过这样的经历，一个游戏安装好之后，啥都不用管，每次点开游戏之后读个条，游戏内容就始终是最新的，可能有新的道具、新的活动，可能某个东西过些日子就找不到了。这在手机游戏中是很常见的，比如说最近很火的<公主连结>我也是一开服就进去开始刷副本很有意思，不过这个游戏最重要的一个内容就是剧情，那么我们知道公主连结的剧情非常庞大有CG、语音、对话内容、甚至还有小番剧真的是一个看番游戏哈哈，对于这种游戏如果玩过的可能很清楚在刚开服的时候就要经过一些列的下载流程，有些人甚至被直接劝退了，因为刚下完游戏就要在下**2G多的剧情**这是一个很厌烦的事情，我认为游戏厂商在这里处理的对于我们玩家来说不是很友好，但是可能他们也有自己的运营方案，抛开这些不提，我们应该庆幸游戏厂商这样做，因为如果他们把大量的剧情在刚开始分的就很碎，每一个剧情都是先去资源服务器下载资源，然后才能进行剧情看番的操作，这样对于玩家来说早期是很劝退的一件事，不管你游戏有多好玩，而刚一开始先加载大部分的游戏剧情，可以让玩家们在前期有良好的体验感，那些喜欢看剧情的在早期可以没有困扰的欣赏下下来那2G的剧情，不需要一段剧情加载一段时间，那样看番很让人讨厌，就好比你刚看到最精彩的时候，想立刻看下一篇但是却要先加载，那么对于玩家来说火气就上来了，我也时常有这种感觉，尤其是那些一周一更新的动漫，对于那些喜欢刷本的、PK的玩家他们更希望能快速通过剧情获取奖励，然后继续通关更高难度的关卡，这时先更新比较大部分的剧情更适合他们，总结来说不管是剧情党、副本党、PK党前期有良好的体验更能留住这些用户，流量就是游戏最主要的赚钱方式，到了中后期，有些人慢慢就会被游戏玩法吸引，更多倾向于游戏性，这时加载就变成了分部分加载更适合玩家，玩家可以自主选择只加载剧情不加载语音、全部都加载，这让所有的玩家有了选择的权力。
## 为什么C#没法直接热更新
其实对于安卓平台来说C#是可以进行热更新的，我们只需要在资源中新增一个目录来存放热更项目生成的DLL就可以在主工程中通过反射调用热更项目中DLL中的函数来执行相应的逻辑了，但是可惜的是在2017年的时候，IOS禁止在应用/游戏里面使用Lua或JavaScript脚本进行热更新（国内主要是使用rollout、jspatch的热更新技术框架）这就导致国内大部分热更新的应用和游戏被苹果商店下架，而C#DLL想要运行必须通过JIT(及时编译)方式，而被禁止后就没法直接热更新了
## lua热更新
对于lua来说，原生lua是一个小巧的嵌入型语言，他可以很方便和C#集成达到动态运行的效果，而另一个luajit简单来说是一个高效的lua，因为ios的缘故他的jit方式运行受到了限制，但是luajit分为jit模式和interpreter模式。
关于这两种模式:
- jit模式：这是luajit高效所在，简单地说就是直接将代码编译成机器码级别执行，效率大大提升（事实上这个机制没有说的那么简单，下面会提到）。然而不幸的是这个模式在ios下是无法开启的，因为ios为了安全，从系统设计上禁止了用户进程自行申请有执行权限的内存空间，因此你没有办法在运行时编译出一段代码到内存然后执行，所以jit模式在ios以及其他有权限管制的平台（例如ps4，xbox）都不能使用。
- interpreter模式：那么没有jit的时候怎么办呢？还有一个interpreter模式。事实上这个模式跟原生lua的原理是一样的，就是并不直接编译成机器码，而是编译成中间态的字节码（bytecode），然后每执行下一条字节码指令，都相当于swtich到一个对应的function中执行，相比之下当然比jit慢。但好处是这个模式不需要运行时生成可执行机器码（字节码是不需要申请可执行内存空间的），所以任何平台任何时候都能用，跟原生lua一样。这个模式可以运行在任何luajit已经支持的平台，而且你可以手动关闭jit，强制运行在interpreter模式下。
- 我们经常说的将lua编译成bytecode可以防止破解，这个bytecode是interpreter模式的bytecode，并不是jit编译出的机器码（事实上还有一个在bytecode向机器码转换过程中的中间码SSA IR，有兴趣可以看luajit官方wiki），比较坑的是可供32位版本和64位版本执行的bytecode还不一样，这样才有了著名的2.0.x版本在ios加密不能的坑。

所以对于这种情况就诞生了xlua、tolua、ulua这种游戏热更新框架
拿xLua举例，xLua有两个版本，分别集成了lua5.3和luajit，一个项目只能选择其一。这两个版本C#代码是一样的，不同的是Plugins部分。

lua5.3的特性更丰富些，比如支持原生64位整数，支持苹果bitcode，支持utf8等。出现问题因为是纯c代码，也好定位。比起luajit，lua对安装包的影响也更小。

而luajit胜在性能，如果其jit不出问题的话，可以比lua高一个数量级。目前luajit作者不打算维护luajit，在找人接替其维护，后续发展不太明朗。

项目可以根据自己情况判断哪个更适合。因为目前lua53版本使用较多，所以xLua工程Plugins目录下默认配套是lua53版本。

# ILRuntime诞生
ILRuntime项目为基于C#的平台（例如Unity）提供了一个纯C#实现，**快速**、**方便**且**可靠**的IL运行时，使得能够在不支持JIT的硬件环境（如iOS）能够实现代码的热更新
# ILRuntime原理
这些是我自己总结的可能会有些出入，ilr通过**Mono.Cecil**(一个可加载并浏览现有程序集并进行动态修改并保存的.NET框架)来读取DLL中的PE信息，读取出类型、函数等最终获得方法的IL代码(因为可以在开发机上编译成DLL然后DLL中保存IL代码)，然后通过ilr自己实现的托管栈和托管对象栈进行代码的执行，基本类型的运算等功能，最终实现不依靠JIT也可以解释运行。ilr实现考虑到用C#的Stack会造成拆箱、装箱等不必要的GC Alloc而且会非常耗时效率低下，所以用unsafe的代码以及非托管内存自己实现了IL托管栈，通过struct、enum、和List实现了可以表达C#所有基础类型的功能。
# 托管调用栈
ILRuntime在进行方法调用时，需要将方法的参数先压入托管栈，然后执行完毕后需要将栈还原，并把方法返回值压入栈。

具体过程如下图所示

调用前:　　　　　　　　　　　　　　　　　调用完成后:

|----------------|　　　　　　　　　　　|------------------|

|　　参数1　  &nbsp; |　　　　|-------------->  &nbsp;| &nbsp;&nbsp;&nbsp;&nbsp; [返回值]　　|

|----------------|　　　　|　　　　　　&nbsp;&nbsp; |------------------|

|　　　...　 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|　　 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|　　　　　　&nbsp;&nbsp;&nbsp;| &nbsp;&nbsp;&nbsp;&nbsp; NULL　　　 |

|----------------|　　　　|　　　　　　&nbsp;&nbsp; |------------------|

|　　参数N　&nbsp;&nbsp;&nbsp;|　　　　|　　　　　　&nbsp;&nbsp;|&nbsp;　　　...　　　&nbsp;|

|----------------|　　　　&nbsp;|

|　局部变量1　|　　　　&nbsp;|

|----------------|　　　　&nbsp; |


|　　　...　　&nbsp; &nbsp;|　　　　&nbsp;|

|----------------|　　　&nbsp;&nbsp; &nbsp; |

|　局部变量1　|　　　　&nbsp; |

|----------------|　　　　&nbsp; |

|　方法栈基址&nbsp; |　　　　&nbsp; |

|----------------|　　　　&nbsp; |

|　 &nbsp;[返回值] &nbsp; &nbsp; |　　------

|----------------|

函数调用进入目标方法体后，栈指针（后面我们简称为ESP）会被指向方法栈基址那个位置，可以通过ESP-X获取到该方法的参数和方法内部申明的局部变量，在方法执行完毕后，如果有返回值，则把返回值写在方法栈基址位置即可（上图因为空间原因写在了基址后面）。

当方法体执行完毕后，ILRuntime会自动平衡托管栈，释放所有方法体占用的栈内存，然后把返回值复制到参数1的位置，这样后续代码直接取栈顶部就可以取到上次方法调用的返回值了。

## ILRuntime的优势
同市面上的其他热更方案相比，ILRuntime主要有以下优点：

- 无缝访问C#工程的现成代码，无需额外抽象脚本API

- 直接使用VS2015进行开发，ILRuntime的解译引擎支持.Net 4.6编译的DLL

- 执行效率是L#的10-20倍

- 选择性的CLR绑定使跨域调用更快速，绑定后跨域调用的性能能达到slua的2倍左右（从脚本调用GameObject之类的接口）

- 支持跨域继承

- 完整的泛型支持

- 拥有Visual Studio的调试插件，可以实现真机源码级调试。支持Visual Studio 2015 Update3 以及Visual Studio 2017和Visual Studio 2019
## ILRuntime如何工作
针对于ILRuntime的一些用法在官网有详细的介绍，而且官网的例子通俗易懂，甚至注释非常丰富基本每行都有注释，一共有10个Demo。这里我们只看他的工作流程。
以Demo中第一个例子Helloworld为例
1. 首先要了解文件目录结构

![目录结构](https://s1.ax1x.com/2020/05/18/YhyXa6.png)

2. 然后我们可以进到启动脚本中看看
```csharp
using UnityEngine;
using System.Collections;
using System.IO;
using ILRuntime.Runtime.Enviorment;

public class HelloWorld : MonoBehaviour
{
    //AppDomain是ILRuntime的入口，最好是在一个单例类中保存，整个游戏全局就一个，这里为了示例方便，每个例子里面都单独做了一个
    //大家在正式项目中请全局只创建一个AppDomain
    AppDomain appdomain;

    System.IO.MemoryStream fs;
    System.IO.MemoryStream p;
    void Start()
    {
        StartCoroutine(LoadHotFixAssembly());
    }

    IEnumerator LoadHotFixAssembly()
    {
        //首先实例化ILRuntime的AppDomain，AppDomain是一个应用程序域，每个AppDomain都是一个独立的沙盒
        appdomain = new ILRuntime.Runtime.Enviorment.AppDomain();
        //正常项目中应该是自行从其他地方下载dll，或者打包在AssetBundle中读取，平时开发以及为了演示方便直接从StreammingAssets中读取，
        //正式发布的时候需要大家自行从其他地方读取dll

        //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
        //这个DLL文件是直接编译HotFix_Project.sln生成的，已经在项目中设置好输出目录为StreamingAssets，在VS里直接编译即可生成到对应目录，无需手动拷贝
        //工程目录在Assets\Samples\ILRuntime\1.6\Demo\HotFix_Project~
        //以下加载写法只为演示，并没有处理在编辑器切换到Android平台的读取，需要自行修改
#if UNITY_ANDROID
        WWW www = new WWW(Application.streamingAssetsPath + "/HotFix_Project.dll");
#else
        WWW www = new WWW("file:///" + Application.streamingAssetsPath + "/HotFix_Project.dll");
#endif
        while (!www.isDone)
            yield return null;
        if (!string.IsNullOrEmpty(www.error))
            UnityEngine.Debug.LogError(www.error);
        byte[] dll = www.bytes;
        www.Dispose();

        //PDB文件是调试数据库，如需要在日志中显示报错的行号，则必须提供PDB文件，不过由于会额外耗用内存，正式发布时请将PDB去掉，下面LoadAssembly的时候pdb传null即可
#if UNITY_ANDROID
        www = new WWW(Application.streamingAssetsPath + "/HotFix_Project.pdb");
#else
        www = new WWW("file:///" + Application.streamingAssetsPath + "/HotFix_Project.pdb");
#endif
        while (!www.isDone)
            yield return null;
        if (!string.IsNullOrEmpty(www.error))
            UnityEngine.Debug.LogError(www.error);
        byte[] pdb = www.bytes;
        fs = new MemoryStream(dll);
        p = new MemoryStream(pdb);
        appdomain.LoadAssembly(fs, p, new ILRuntime.Mono.Cecil.Pdb.PdbReaderProvider());

        InitializeILRuntime();
        OnHotFixLoaded();
    }

    void InitializeILRuntime()
    {
#if DEBUG && (UNITY_EDITOR || UNITY_ANDROID || UNITY_IPHONE)
        //由于Unity的Profiler接口只允许在主线程使用，为了避免出异常，需要告诉ILRuntime主线程的线程ID才能正确将函数运行耗时报告给Profiler
        appdomain.UnityMainThreadID = System.Threading.Thread.CurrentThread.ManagedThreadId;
#endif
        //这里做一些ILRuntime的注册，HelloWorld示例暂时没有需要注册的
    }

    void OnHotFixLoaded()
    {
        //HelloWorld，第一次方法调用
        appdomain.Invoke("HotFix_Project.InstanceClass", "StaticFunTest", null, null);

    }

    private void OnDestroy()
    {
        if (fs != null)
            fs.Close();
        if (p != null)
            p.Close();
        fs = null;
        p = null;
    }

    void Update()
    {
        if (Input.GetKeyDown(KeyCode.A))
        {
            StartCoroutine(LoadHotFixAssembly());
            appdomain.Invoke("HotFix_Project.InstanceClass", "StaticFunTest", null, null);
        }  
    }
}
```
我们可以看里面内容非常齐全，大致流程简单概括就是，在开始的时候声名`AppDomain appdomain`为全局ilr入口一般为单例全场只存在一个,然后开启协程给它生成实例对象，然后通过WWW进行资源加载，我们加载的是刚才StreamingAssets下的DLL和PDB文件(用于调试),加载完成后用`appdomain.LoadAssembly`加载DLL程序集中的内容，最后通过invoke执行
<font color = red>"HotFix_Project.InstanceClass"</font>(前面分别为命名空间和类名)下的
<font color = red>"StaticFunTest"</font>方法。

这么说可能不太直观，我们可以找到Hotfix的工程(也就是DLL的源工程)他在**Assets\Samples\ILRuntime\1.6\Demo\HotFix_Project~** 目录下可能因为Unity没有识别要从文件夹中进入我们使用vs打开这个文件夹的.sln就能打开热更层的代码了，我们找到上面那个类的.cs文件如下:
```csharp
using System;
using System.Collections.Generic;

namespace HotFix_Project
{
    public class InstanceClass
    {
        private int id;

        public InstanceClass()
        {
            UnityEngine.Debug.Log("!!! InstanceClass::InstanceClass()");
            this.id = 0;
        }

        public InstanceClass(int id)
        {
            UnityEngine.Debug.Log("!!! InstanceClass::InstanceClass() id = " + id);
            this.id = id;
        }

        public int ID
        {
            get { return id; }
        }

        // static method 刚才主工程调用的就是这个方法
        public static void StaticFunTest()
        {
            UnityEngine.Debug.Log("!!! InstanceClass.StaticFunTest()");
        }

        public static void StaticFunTest2(int a)
        {
            UnityEngine.Debug.Log("!!! InstanceClass.StaticFunTest2(), a=" + a);
        }

        public static void GenericMethod<T>(T a)
        {
            UnityEngine.Debug.Log("!!! InstanceClass.GenericMethod(), a=" + a);
        }

        public void RefOutMethod(int addition, out List<int> lst, ref int val)
        {
            val = val + addition + id;
            lst = new List<int>();
            lst.Add(id);
        }
    }
}
```

我们发现了在场景开始脚本中调用的方法，接下来我们进行热更新的测试，我们稍稍修改下主工程的脚本
```csharp
//...HelloWorld.cs这个脚本
    void Update()
    {
        if (Input.GetKeyDown(KeyCode.A))
        {
            StartCoroutine(LoadHotFixAssembly());
            appdomain.Invoke("HotFix_Project.InstanceClass", "StaticFunTest", null, null);
        }  
    }
```

为了方便测试我们在Update中增加输入检测，当玩家按下A键的时候先去加载DLL文件，接着执行刚才我们找到的那个方法。
- 我们第一次先将输出内容改成"游戏上线"
```csharp
        // static method
        public static void StaticFunTest()
        {
            UnityEngine.Debug.Log("游戏上线");
        }
```
然后在主工程中运行并看到结果

![热更前](https://s1.ax1x.com/2020/05/18/Yhyqq1.png)

- 第二次我们不停止游戏将热更曾代码输出内容改为"游戏突发bug进行紧急修复完成 并进行热更新"然后别忘记要重新生成(编译)热更新层的代码
```csharp
        // static method
        public static void StaticFunTest()
        {
            UnityEngine.Debug.Log("游戏突发bug进行紧急修复完成 并进行热更新");
        }
```
我们再次按下A键发现内容改变了，至此简单热更新流程完毕

![热更后](https://s1.ax1x.com/2020/05/18/YhyOVx.png)

其他ILRuntime还提供了很多强大的功能等着我们去探索，它的代码里写了很多让我们自己去实现的东西(你就不能直接帮我们实现了吗(狗头)，还有ilr还可以跨域继承、跨域委托、反射、重定向、绑定等需要我们仔细研究文档或者实战中去体会他们的用法。
# iOS IL2CPP类型裁剪问题
IL2CPP在打包时会自动对Unity工程的DLL进行裁剪，将代码中没有引用到的类型裁剪掉，以达到减小发布后ipa包的尺寸的目的。然而在实际使用过程中，很多类型有可能会被意外剪裁掉，造成运行时抛出找不到某个类型的异常。特别是通过反射等方式在编译时无法得知的函数调用，在运行时都很有可能遇到问题。

Unity提供了一个方式来告诉Unity引擎，哪些类型是不能够被剪裁掉的。具体做法就是在Unity工程的Assets目录中建立一个叫link.xml的XML文件，然后按照下面的格式指定你需要保留的类型：
```xml
<linker>
  <assembly fullname="UnityEngine" preserve="all"/>
  <assembly fullname="Assembly-CSharp">
    <namespace fullname="MyGame.Utils" preserve="all"/>
    <type fullname="MyGame.SomeClass" preserve="all"/>
  </assembly>  
</linker>
```
# 泛型实例
每个泛型实例实际上都是一个独立的类型，`List<A>` 和 `List<B>`是两个完全没有关系的类型，这意味着，如果在运行时无法通过JIT来创建新类型的话，代码中没有直接使用过的泛型实例都会在运行时出现问题。

在ILRuntime中解决这个问题有两种方式，一个是使用CLR绑定，把用到的泛型实例都进行CLR绑定。另外一个方式是在Unity主工程中，建立一个类，然后在里面定义用到的那些泛型实例的public变量。这两种方式都可以告诉IL2CPP保留这个类型的代码供运行中使用。

因此建议大家在实际开发中，尽量使用热更DLL内部的类作为泛型参数，因为DLL内部的类型都是ILTypeInstance，只需处理一个就行了。此外如果泛型模版类就是在DLL里定义的的话，那就完全不需要进行任何处理。

# 泛型方法
跟泛型实例一样，foo.Bar`<TypeA>` 和foo.Bar`<TypeB>`是两个完全不同的方法，需要在主工程中显式调用过，IL2CPP才能够完整保留，因此需要尽量避免在热更DLL中调用Unity主工程的泛型方法。如果在iOS上实际运行遇到报错，可以尝试在Unity的主工程中随便写一个static的方法，然后对这个泛型方法调用一下即可，这个方法无需被调用，只是用来告诉IL2CPP我们需要这个方法

# C# vs Lua
目前市面上主流的热更方案，主要分为Lua的实现和用C#的实现，两种实现方式各有各的优缺点。

Lua是一个已经非常成熟的解决方案，但是对于Unity项目而言，也有非常明显的缺点。就是如果使用Lua来进行逻辑开发，就势必要求团队当中的人员需要同时对Lua和C#都特别熟悉，或者将团队中的人员分成C#小组和Lua小组。不管哪一种方案，对于中小型团队都是非常痛苦的一件事情。

用C#来作为热更语言最大的优势就是项目可以用同一个语言来进行开发，对Unity项目而言，这种方式肯定是开发效率最高的。

Lua的优势在于解决方案足够成熟，之前的C++团队可能比起C#，更加习惯使用Lua来进行逻辑开发。此外借助luajit，在某些情况下的执行效率会非常不错，但是luajit现在维护情况也不容乐观，官方还是推荐使用公版Lua来开发。
# 结束语
由于篇幅限制就说到这里，Lua热更新想着新开一篇博客继续讲，我们下次再见
# 参考连接
ILRuntime官网:http://ourpalm.github.io/ILRuntime/public/v1/guide/index.html