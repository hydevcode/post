+++
date = '2025-04-08T19:20:00+08:00'
draft = false
title = 'RT-Thread仓库的CI自动编译测试'
+++

# 前言

当我们往RT-Thread仓库提交PR代码也就是对仓库做修改的时候，来到提交的PR底下可以看到一些Build Check，如下图所示

![](Git/RT-Thread仓库的CI自动编译测试/images/Pasted%20image%2020250403224733.png)

这些Check的作用就是编译与其对应的BSP，就好像本地使用scons -j4编译一样

当所有测试通过了之后就代表在该PR中修改的代码可以正常编译所在的BSP

如果有报错也就是编译不通过的话，那么就可以很方便的定位问题来源

# 怎么添加BSP的自动编译测试

问题来了，那么当添加了一个新的BSP的时候该如何让它也能支持自动编译呢

来到.github目录下可以找到一个叫ALL_BSP_COMPILE.json的文件
这里面就包含了所有需要自动编译测试的bsp

当我们添加了一个新的BSP时，只需要按照原有的格式，将BSP的路径添加到列表中即可

但是需要注意几点

首先，每个BSP都应该被放入正确的组中。举个例子，如果添加的是STM32F7系列的BSP，那么就应该将其路径放入相应的组。如下图的的RTT_BSP就类似于组名

如果在现有的组中找不到合适的位置，那么就可以按照已有的格式新增一个组

![](Git/RT-Thread仓库的CI自动编译测试/images/Pasted%20image%2020250403230111.png)


添加完成后可以验证一下看是否成功添加

来到PR页面的最下方，找到刚添加的bsp对应的组名，点击链接

这里以星火一号RT-SPARK为例子


![](Git/RT-Thread仓库的CI自动编译测试/images/Pasted%20image%2020250408220004.png)

展开BSP Scons Compile即可看到bsp路径

![](Git/RT-Thread仓库的CI自动编译测试/images/Pasted%20image%2020250408220159.png)
# 如何在自己仓库上进行自动编译测试

(在自己仓库上编译需要修改一下仓库文件，删除.github/workflows/bsp_buildings.yml中的**if: github.repository_owner == 'RT-Thread'** 即可在自己的仓库编译测试，顺便可以把`schedule:- cron:  '0 16 * * *'` 也删了，这个是定时运行，时间长了仓库会有一堆编译记录，这个操作仅用于自己测试，最后提交PR代码的时候记得还原)

首先确保RT-Thread仓库已经fork到自己仓库了(也就是类似于复制一份仓库到自己仓库中)
然后来到自己的RT-Thread仓库

![](Git/RT-Thread仓库的CI自动编译测试/images/Pasted%20image%2020250408220636.png)

点击Actions,然后在左边的列表找到Build Check，在右边可以看到一个Run workflow
，按下那个绿色按钮就会开始自动对所有bsp进行编译了


![](Git/RT-Thread仓库的CI自动编译测试/images/Pasted%20image%2020250408220606.png)

此时刷新一下网页会发现多出来一个棕色的Build Check，这时说明正在进行测试，只需要等待测试结束即可

![](Git/RT-Thread仓库的CI自动编译测试/images/Pasted%20image%2020250408220852.png)

另外CI还有其他好玩的用法，结合起来做项目十分方便

[RT-Thread-还在担心bsp不好维护吗？快使用yml管理主线bspRT-Thread问答社区 - RT-Thread](https://club.rt-thread.org/ask/article/5c41835bb8ff9b41.html)

[RT-Thread-RT-Thread CI编译产物artifacts自动上传功能介绍RT-Thread问答社区 - RT-Thread](https://club.rt-thread.org/ask/article/95e2a0f0a1517189.html)

[RT-Thread-【ci】【github】【bsp】RT-THREAD所有bsp scons编译情况检查RT-Thread问答社区 - RT-Thread](https://club.rt-thread.org/ask/article/8e019d0e64e1ab31.html)

[RT-Thread-\[github\]\[action\] RTT黑科技: 添加手动打包和编译特定bsp功能RT-Thread问答社区 - RT-Thread](https://club.rt-thread.org/ask/article/419a30e57384a239.html)