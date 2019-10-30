---
title:      "如何对一个大型Android项目进行模块化重构"
description:   "相册模块化方案及心得"
date:       2019-2-2 12:00:00
author:     "安地"
tags:
      - Android

---


## 背景

相册作为一个系统应用，17年才转到gradle编译，带着很多沉重的包袱，基本只有一个app模块，编译速度慢。
同时相册上又有许多新的功能，整体代码量层线性增长，一些相对独立的新功能会把代码资源都放在一个单独文件夹中，但编译时还是编译在同一个app模块里，各部分之前耦合度高。
这些单独的模块是不是应该抽离成单独Module？
第一问之后又会延伸出下面几个问题：
基础组件是不是应该独立出来？
各部分代码耦合是不是应该抽离？
工具类依赖业务类是否需要重构？

## 为什么要模块化

做一件事得思考利益，讲投入产出。模块化主要有这些好处：
1.使各模块解耦，提高代码的可维护性，减小bug产生
2.增加模块独立运行和测试能力
3.提升编译速度

对于2各个功能目前都没有单独运行的需求，单独测试有一定需求；对于3提升速度有限必要性不大。
1是我们最重要的需求，是长期的好处。模块化后各个组件结构清晰，对于项目维护，新增功能，减少bug，都会有很大的好处。
所以这是一件值得做的是，优先级是是重要不紧急。

## 如何着手做

实际模块化也就是代码重构的事，我们日常功能需求比较多，这上面只能用少部分时间。所以我定了一个小步慢走的方案，每周用一天左右的时间进行代码重构。

## 思路选择


思路一：直接外移方案
当前app module不动，逐渐外移单独module，让单独module依赖app module，最后app module会全部被外移。
这种方案开始影响最小，但不适合我们的项目，因为我们的主工程是非常大的，分支小，app module没法也没必要全部外移拆分，这样外移的小模块依赖大模块，没有意义。

思路二：先整理后外移方案
单独module依赖的东西有统计，网络，preference，imageLoader，基础utils等，那先整理这些基础组件，进行下移，然后再进行独立module的重构，独立module和app module之间要互相不依赖，互相调用使用模块化接口。
这种方案对代码的整理较多，这也是我们最终需要做的，最终决定使用这种方案。


## 实现

### 基础组件下移

提前调研需要独立的模块，分析依赖的基础模块，进行基础组件的下移。
下移时尽量轻量进行，命名无非必要不进行更改，需要重构可以后续再进行，确保下移的影响最小。
这部分工作是实际最耗时间精力的，因为很多工具类和业务类耦合很严重，需要针对性梳理和划分。

### 模块化通信

独立模块之间如何通信，因为互相有接口调用，所以采用接口化方案，接口定义在公共模块，实现在独立功能模块，然后就需要注册和获取。
新建了一个iModule的模块，模块化的类需要依赖此模块，此模块里面有模块化接口的定义，各个模块中做对应实现。app启动时进行模块注册。


``` Java
class ModuleManagerImpl implements IModuleManager {

    private static final String TAG = "ModuleManagerImpl";

    private static final IModuleManager mModuleManager = new ModuleManagerImpl();
    private final HashMap<Class<? extends IModule>, IModuleImpl> mBuiltinModules = new LinkedHashMap();

    public static IModuleManager getModuleManager() {
        return mModuleManager;
    }

    private ModuleManagerImpl() {
    }

    @Override
    public IModule getModule(Class<? extends IModule> cls) {
        return mBuiltinModules.get(cls);
    }

    public void addModule(Class<? extends IModule> cls, IModuleImpl moduleImpl) {
        this.mBuiltinModules.put(cls, moduleImpl);
    }
}
```

ModuleManageImpl保存了一个map，key为IModule接口类，value为IModuleImpl类，这样通过接口就可以获取到对应的实现，进行模块间通信。

### 代码隔离

上面说到app启动时需要进行模块注册，如果在app模块注册的话，app模块就依赖了独立module。
我们可以使用gradle的implement进行隔离，implement的作用是依赖的模块的代码不对外暴露给依赖我的模块，如果需要暴露就使用api进行模块引入。
新增一个moduleservice，moduleservice依赖所有的独立模块，在moduleservice进行模块注册，没有其它功能仅仅是为了做隔离，app依赖moduleservice，app启动时调用moduleservice的模块注册方法。


### 层次结构
这样我们的应用就分成了这样的结构：

1.主功能层：app模块，包括主体业务功能
2.模块注册层：moduleservice， 用于隔离直接依赖
3.独立模块层：包括照片电影，拼图等独立功能模块，以library存在，可单独添加application的测试模块
4.基础组件层：包括网络，图片库，统计，基础工具等

### 进一步自动化

这个地方再进一步自动化的话，可以使用gradle插件，在集成时添加依赖，在编译期修改代码，自动添加模块注册方法；
自动识别运行的application,对模块留一份单独运行的代码,做到即可做library又可独立测试运行。

这部分参照一些组件化框架进行了尝试，当前还没有在工程上使用，这些是锦上添花的工作，主要的还是把前面几步完成好，循序渐进。


## 总结

本文介绍了相册模块化中的思路，对于一个庞杂的系统应用该如何梳理，实现模块化改造。
现在大家经常说的是组件化，和模块化相比更强调组件的复用性。这次梳理没有进行业务的组件梳理，只独立出一些基础功能组件，所以主要基于的还是模块化。
组件化，模块化方案思路很多，一定要根据业务需求进行选择，先业务后技术，而不是先技术后业务。
当前模块化完成后只是提供基本的框架结构，后续还需一步步优化改进，代码重构是一项持续的工作。



