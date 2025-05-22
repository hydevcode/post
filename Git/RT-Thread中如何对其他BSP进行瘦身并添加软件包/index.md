+++
date = '2025-05-22T01:59:41+08:00'
draft = false
title = 'RT-Thread中如何对其他BSP进行瘦身并添加软件包'
+++

# 介绍

瘦身的目的就是减少整个仓库的文件体积，为此，我们需要分析当前各个 BSP 中占用空间较大的部分

在此之前，已有开发者发布了一篇详细的瘦身指南，具体在下方链接可以查看:
[RT-Thread-rt-thread主仓 BSP瘦身指南RT-Thread问答社区 - RT-Thread](https://club.rt-thread.org/ask/article/4839095b91c2f849.html)

这里以NXP的为例子，我们可以用SpaceSniffer查看该目录的文件占用

![](Git/RT-Thread中如何对其他BSP进行瘦身并添加软件包/images/Pasted%20image%2020250519154446.png)

从图中可以看出，仓库中占用空间较多的主要是一些 library 及与 SDK 相关的内容。这些库本身并不直接依赖 RT-Thread 内核，而是通过适配层与系统进行对接。因此，我们可以将这些库拆分出来，分别制作成独立的软件包进行管理和维护，以减少 BSP 体积。

# 开始操作

为了方便演示瘦身的具体操作步骤，我们以图中右下角的MCX为例进行说明，该BSP适配的比较标准，适合作为示例，能够更直观地展示瘦身的原理和过程

瘦身主要分为三个步骤，首先是创建对应的SDK仓库，然后是修改Package软件包索引仓库，最后是修改RT-Thread主仓仓库中的BSP

瘦身本质的核心就是将占用比较多的文件或者库分成单独的仓库

这一套流程不仅适用于 BSP 的瘦身，同样适用于向 RT-Thread 生态中添加新的软件包


本文中提交PR的步骤可以参考
[RT-Thread-【RT-Thread】记录一次对主仓的bsp进行修复并提交pr的总结RT-Thread问答社区 - RT-Thread](https://club.rt-thread.org/ask/article/62273896dee18b09.html)

## 创建SDK（软件包）仓库

在创建仓库前，首先需要分析当前目录结构，明确哪些内容需要拆分成独立仓库

来到nxp/mcxa目录下，可以看到该目录下包含两个 BSP，以及一个位于 BSP 外部的 `Library` 文件夹

其中`Library` 中包含一个通用的 **CMSIS** 文件夹，这是两个 BSP 在编译过程中都会引用的公共部分，需要作为通用 **SDK** 提取出来

`drivers` 是用于适配 RT-Thread 的驱动代码，属于 BSP 的一部分，不需要拆分为软件包

此外还有两个 BSP 专属的库文件夹，这两部分可以**合并为一个软件包**，并通过 `SConstruct` 文件中的配置来控制编译时启用哪个文件夹，以适配不同的 BSP

![](Git/RT-Thread中如何对其他BSP进行瘦身并添加软件包/images/Pasted%20image%2020250519163959.png)

（注：除了library之类的代码，如果还有对应bsp需要的工具或文件的话，比如IMX6ULL-Smart这个bsp目录下还有个emmc文件夹，那么可以把其一起放到对应的软件包文件夹下 ）

在明确library库中的内容后，我这里打算分别创建出nxp-mcx-cmsis和nxp-mcx-series两个仓库

### 制作软件包并测试

这里可以在package下创建文件夹模拟测试下是否能通过编译：

以 `frdm-mcxa156` 为例,来到其目录下，打开env，输入pkgs --update让他创建出Packages文件夹，接下来在这个文件夹下分别创建出nxp-mcx-cmsis和nxp-mcx-series的文件夹

将原 `Library` 文件夹中的 `cmsis` 目录内容放到nxp-mcx-cmsis中(可以理解为将 `cmsis` 改名为 `nxp-mcx-cmsis`)，另外两个 BSP 专属的库文件夹，则统一放入 `nxp-mcx-series` 软件包中，后续可通过 `SConstruct` 控制编译时选择具体子目录，以适配不同 BSP。最后的结果如下图所示，然后原本的cmsis和库文件就可以删除了。

![](Git/RT-Thread中如何对其他BSP进行瘦身并添加软件包/images/Pasted%20image%2020250519171830.png)
### scons是如何编译需要的.c文件的

在默认情况下，SCons 的编译流程主要依赖于 `SConscript` 文件中指定的路径来递归查找 BSP 目录下所有需要编译的 `.c` 文件。

如果源码目录结构中存在多层嵌套，**那么每一层目录下都需要有一个 `SConscript` 文件**。因为 SCons 会从上层向下逐级查找 `SConscript`，并通过它们逐层递归构建编译路径。

一旦某一层目录中缺失了 `SConscript`，则该路径下更深层的 `.c` 文件将无法被发现和编译，从而导致编译不完整或链接错误。

但是从上图中可以发现nxp-mcx-series下是少了SConscript的，这会导致里面的文件编译不进去，于是我们可以手动给它加上

其中的内容可以参考下图，每个专属的库都对应的一个SOC宏定义，当这个判断到这个宏定义后就会把对应的库中的SConscript中指定的C文件加入到构建中

![](Git/RT-Thread中如何对其他BSP进行瘦身并添加软件包/images/Pasted%20image%2020250519172251.png)

然后检查下对应的库中的SConscript，我这里是MCXA156,可以打开检查下MCXA156\SConscript

![](Git/RT-Thread中如何对其他BSP进行瘦身并添加软件包/images/Pasted%20image%2020250519173049.png)

发现CMSIS也包含在里面，那么把这个CMSIS改成nxp-mcx-cmsis是不是nxp-mcx-cmsis就不用创建SConscript了，实际上这样会报错，在拉取软件包的时候由于用的是最新的代码，那么nxp-mcx-cmsis的命名就会变成nxp-mcx-cmsis_latest,这时就会报错

为了方便可能出现的不同版本，同时也让软件包更整洁一点，还是要在nxp-mcx-cmsis创建一个SConscript,内容为下图所示

![](Git/RT-Thread中如何对其他BSP进行瘦身并添加软件包/images/Pasted%20image%2020250519173715.png)

那么原来的nxp-mcx-series中的SConscript则删掉 ***cwd + '/../CMSIS/Core/Include',*** 即可

这样nxp-mcx-cmsis和nxp-mcx-series两个软件包就做好了

接下来是把原本bsp查找library路径修改一下就好了

前面提到，SCons 的构建过程主要依赖于各级目录下的 `SConscript` 文件来组织和管理需要编译的 `.c` 文件。那么，SCons 最开始是从哪里开始执行的呢？

实际上，**SCons 会从BSP根目录下的 `SConstruct` 文件开始执行**，这是整个构建系统的入口文件。`SConstruct` 正常情况下只会从bsp根目录下开始搜索，进而递归来添加该 BSP 所需编译的所有源文件。

![](Git/RT-Thread中如何对其他BSP进行瘦身并添加软件包/images/Pasted%20image%2020250519174711.png)

如果需要搜索bsp目录外的文件夹，则需要指定对应的目录，比如说上图，首先objs添加了RTT内核，其次是drivers驱动，最后是library里的库文件

所以为了适配软件包，则需要把这一行添加库文件夹的代码去掉

```
objs.extend(SConscript(os.path.join(libraries_path_prefix, rtconfig.BSP_LIBRARY_TYPE, 'SConscript')))
```

这时候又由于默认会自动搜索BSP根目录下所有文件夹包括packages里的SConscript，所以软件包方面不用额外添加路径

这时候就可以尝试编译一下

![](Git/RT-Thread中如何对其他BSP进行瘦身并添加软件包/images/Pasted%20image%2020250519175558.png)

结果可以看到库文件的路径来自于Packages，编译也是通过的

### 提交到github仓库

首先，需要在 GitHub 上创建两个仓库，并分别命名为 **nxp-mcx-cmsis** 和 **nxp-mcx-series**，同时记录下这两个仓库对应的 Git 拉取地址。

接着，在本地打开 **packages** 文件夹内对应这两个文件夹路径的 Git Bash 窗口。

最后，按照以下命令将 SDK 提交至远端仓库中

```
#初始化仓库
git init

#设置远端仓库
git remote add 拉取对应仓库用的git地址

#拉取代码
git pull origin master

#将修改的文件添加到暂存
git add .

#将暂存区的文件提交成commit
git commit "First Commit"

#推送到github服务器上
git push origin master
```

## 修改Package软件包索引仓库

接下来是修改软件包的索引仓库，效果就是Menuconfig打开相应的宏定义即可自动下载对应的软件包

来到软件包仓库[GitHub - RT-Thread/packages: packages index repository for rt-thread.](https://github.com/RT-Thread/packages)

首先需要将仓库Fork成自己的package仓库

![](Git/RT-Thread中如何对其他BSP进行瘦身并添加软件包/images/Pasted%20image%2020250520102853.png)

然后来到自己的package仓库，找到clone地址

![](Git/RT-Thread中如何对其他BSP进行瘦身并添加软件包/images/Pasted%20image%2020250520103457.png)

在本地找一个地方拉取一下

***这里推荐一个package拉取的地方***

正常来说，env工具本身也会拉取package，于是我们可以来到env的安装目录下，找到对应的package目录，删除后重新在该位置拉取一个package，这样在调试的时候就能随时使用到最新代码

拉取到本地后，用vscode打开packages，然后来到peripherals\hal-sdk目录下

找到对应bsp的文件夹，没有的话可以根据其他文件夹的命名规范来新建

然后在bsp下创建Kconfig和package.json，如下图所示

![](Git/RT-Thread中如何对其他BSP进行瘦身并添加软件包/images/Pasted%20image%2020250521185137.png)

其中的内容可以参考其他sdk的配置来修改

![](Git/RT-Thread中如何对其他BSP进行瘦身并添加软件包/images/Pasted%20image%2020250522100054.png)

注意，这里面的Kconfig跟上面提到的SConscript有点类似，也是为了能让env找到sdk的配置，所以每个文件夹下都有

![](Git/RT-Thread中如何对其他BSP进行瘦身并添加软件包/images/Pasted%20image%2020250522100515.png)

将对应的配置路径填入到上层文件夹中的Kconfig即可，具体可以参考其他SDK

全部写好后就可以提交pr了，打开git bash

```
#将文件添加到暂存区
git add .

#创建分支
git checkout -b nxp_sdk

#提交暂存区文件到commit
git commit -m "这里填写commmit描述"

#推送远端仓库
git push origin nxp_sdk
```

推送到github上自己的仓库后，就可以在网页端操作提交pr了

关于软件包相关的部分详细可以参考其他的文章和视频比如

[\[RTT\]\[ENV\]\[PACKAGE\]如何制作软件包\_rtt env packages-CSDN博客](https://blog.csdn.net/lt6210925/article/details/105176863?spm=1001.2014.3001.5501)

[Env上手指南\_哔哩哔哩\_bilibili](https://www.bilibili.com/video/av79943543)

## 修改RT-Thread主仓仓库

首先如果需要在主仓上提交代码，我们需要把仓库fork一下，然后拉取自己的RT-Thread仓库到本地即可,其步骤跟修改索引仓库一样

在第一步的测试中，BSP 已经能够正常编译。接下来，如果要实现对应 SDK 的自动下载功能，则需要借助软件包索引仓库中的宏定义来完成。

当使用menuconfig保存退出后，env会自动将Kconfig下开启的宏定义放到rtconfig.h中，这时如果使用了pkgs --update并检测到PKG_开头的宏定义就会根据索引仓库的配置下载对应的软件包

于是按照这个思路，首先找到这个系列的bsp都会开启的选项，然后在这个选项下面加入刚才索引仓库配置的宏定义就可以了

如下图的例子中，我找到了一个SOC_MCX的一个config，当menuconfig保存退出后，就会自动搜索打开select中的选项，这里把sdk的config开关填进去即可

![](Git/RT-Thread中如何对其他BSP进行瘦身并添加软件包/images/Pasted%20image%2020250522101935.png)

到了这里，bsp的基础瘦身就完成了,接下来是添加一些提示之类的，告诉其他开发者在编译前需要输入pkgs --update下载sdk

打开当前bsp的SConscript,加入下图中的代码，作用就是检测packages目录下有没有对应的sdk文件夹,没有的话就会报错并提示

![](Git/RT-Thread中如何对其他BSP进行瘦身并添加软件包/images/Pasted%20image%2020250522102530.png)

注意：软件包再拉下来的时候会自动在文件夹名后加上-latest，修改的时候不要漏了

最后可以测试一下，把原来的文件夹删除后即可输入pkgs --update下载sdk，编译通过后即可开始提交pr

可以参考下面的链接:
[RT-Thread-【RT-Thread】记录一次对主仓的bsp进行修复并提交pr的总结RT-Thread问答社区 - RT-Thread](https://club.rt-thread.org/ask/article/62273896dee18b09.html)

# 总结

这里总结下步骤
首先，需要创建对应的SDK仓库；
这个仓库会在pkgs --update时下载到packages文件夹里，然后scons编译工具会根据其中的SConscript来决定编译哪个c文件

接着，修改Package软件包索引仓库；
这个仓库主要是用来链接软件包的仓库，并将该软件包跟宏定义绑定，这样env才可以找到该软件包的仓库并下载下来

最后，更新RT-Thread主仓仓库中的BSP；
为了使 BSP 在执行 **pkgs --update** 时能够正确工作，需要确保对应软件包的宏定义处于开启状态。因此，必须在 Kconfig 文件中选择一个默认开启的选项，并将该宏定义包含其中。这样一来，在通过 menuconfig 配置后，就能启用该宏定义。 （注：menuconfig 中的选项均来源于 Kconfig 文件。）
