---
title: Jenkins job间触发
description: 得益于各种插件，Jenkins 的用法远比其官方文档上描述的多。这里我使用了两种不同的插件来实现 Jenkins Job 间的触发。
thumbnail: https://res.cloudinary.com/dkzvjuptx/image/upload/v1577004170/Jenkins/jenkinsjobtrigger/jenkins_ua0mg4.png
author: 
	name: qiyuan
	avatar: https://res.cloudinary.com/dkzvjuptx/image/upload/v1578820041/info/favicon_s4pmzz.jpg
pin: false
date: 2019-12-18 11:14:50
tags:
- jenkins
- job触发
categories: Jenkins
keywords: jenkins,job,job触发,jenkinsjob间触发,带参数
---
​		最近在实习,工作内容多多少少跟 Jenkins 有点关系,一段时间的学习下来,对 Jenkins 也有了一定的了解.  
​		就我个人愚见, Jenkins 的用法远比其官方文档所描述的多的多,文档事例仅针对 Jenkins pipeline 举例,对 Github , Gitlab 等代码管理平台上仓库的代码进行管道化的定义,以满足其编译,部署,测试等方面的需求.在我所在的小组维护的 Jenkins 集群中,我了解到了更多的用法. Jenkins 集群可以用于机器管理;一个 Job 可以使用 shell 、python 等脚本语言进行定义,可以实现如 代码编译、代码测试、应用后台管理、微服务部署、集群管理等,甚至可以模仿微服务架构,将一个频繁使用的功能创建多个 Job ,对这些 Job 实现类似"负载均衡"效果.  
​		昨天我导师给我一个需求:简单来说就是要两个 Job 之间带参数触发(实际上还要复杂的多).在经过一段时间了解和学习,现总结一下 Jenkins job 之间实现触发的方式.

#### Parameterized Trigger Plugin  

在 Jenkins 插件管理中添加 Parameterized Trigger Plugin,这个过程不累述.这个插件可以根据已经完成构建的结果,触发新 Job 或者传递参数.  
1. 创建两个 Job (名字不重要):
![Job截图](https://res.cloudinary.com/dkzvjuptx/image/upload/v1576853219/jenkinsjobtrigger/1_krigtv.png)
2. 来看看 odm-job 的配置    
在"Add build step"中选择"Execute shell",我写了一段简单的 shell 来代表此 Job 所做的一系列操作.
![odm-job](https://res.cloudinary.com/dkzvjuptx/image/upload/v1576853220/jenkinsjobtrigger/2_xkt8jz.png)
3. Build 模块下应该会有一个"Post-build Actions"模块,顾名思义,就是在此次 build 结束后触发的操作.我们在"Add post-build action"中选择"Trigger parameterized build on other projects"选项,其他选项都是各自不同的行为,有兴趣的可以自己试一试.
![添加Post-build Actions](https://res.cloudinary.com/dkzvjuptx/image/upload/v1576853220/jenkinsjobtrigger/3_deylfw.png)
4. 添加后,按照提示填写"Projects to build",即需要触发的 Job 名称.勾选"Trigger build without parameters".
在不传递参数的情况下如不勾选此项,将无法触发 Downstream :
		[parameterized-trigger] Downstream builds will not be triggered as no parameter is set.
![填写post-build Actions信息](https://res.cloudinary.com/dkzvjuptx/image/upload/v1576853220/jenkinsjobtrigger/4_hbqtbj.png)
顺便提一下,这里的 Downstream 即 sign-job ,相对的还有 Upstream ,就是指 odm-job .
5. sign-job 的配置很简单, Build 中写一段简单的脚本来代表其构建的过程
![sign-job build](https://res.cloudinary.com/dkzvjuptx/image/upload/v1576853220/jenkinsjobtrigger/5_ltaskm.png)  

保存,构建 odm-job 看看:(我是在 Jenkins 集群上运行的,打码部分为服务器及 workspace 信息)
![运行结果](https://res.cloudinary.com/dkzvjuptx/image/upload/v1576853220/jenkinsjobtrigger/6_rti30x.png)
sign-job 的运行结果:
![运行结果](https://res.cloudinary.com/dkzvjuptx/image/upload/v1576853221/jenkinsjobtrigger/7_o8cgg2.png)  

如果要传递参数,我们来做一些简单的修改:  
1. odm-job 添加两个参数:
![添加参数](https://res.cloudinary.com/dkzvjuptx/image/upload/v1576853221/jenkinsjobtrigger/8_wmmcwz.png)  
2. sign-job 添加两个参数及构建脚本:
![添加参数](https://res.cloudinary.com/dkzvjuptx/image/upload/v1576853220/jenkinsjobtrigger/9_ug7cow.png)  
修改脚本:
![修改脚本](https://res.cloudinary.com/dkzvjuptx/image/upload/v1576853220/jenkinsjobtrigger/10_ohzvwv.png)  
3. 修改 odm-job 配置:
![修改配置](https://res.cloudinary.com/dkzvjuptx/image/upload/v1576853221/jenkinsjobtrigger/11_tzogjv.png)  

再次构建 odm-job 看看结果(记得传参数,我给的参数是 ODM_PATH=111;ODM_TEST=222).odm-job 并没什么不同,主要看看 sign-job 中是否接收到传递过来的参数:
![结果](https://res.cloudinary.com/dkzvjuptx/image/upload/v1576853221/jenkinsjobtrigger/12_y9ipra.png)  
可以看到参数已经传过来了.上述就是使用"Post-build Actions"触发 Job 构建的过程.

#### Conditional BuildStep Plugin  

直译过来就是"有条件的构建步骤"
1. 删除 odm-job 中的 Post-build Actions ,在 Build 模块中添加一个"Conditional step",有 single 和 multiple 两种模式,这里我用 single .
![添加 Conditional step](https://res.cloudinary.com/dkzvjuptx/image/upload/v1576853222/jenkinsjobtrigger/13_unozzu.png)  
2. "Conditional step"可以分为两部分,下半部分与之前 Post-build Actions 没太大差别,上半部分就是所谓的"条件".条件我一般使用 Execute Shell ,在下面的脚本中, exit 0 为触发构建, 除 0 外的 exit 都将不触发构建.除此之外,还有很多其他条件可供选择,有兴趣可以一一试验.
![Conditional step](https://res.cloudinary.com/dkzvjuptx/image/upload/v1576853221/jenkinsjobtrigger/14_zlminv.png)  
这里的条件是如果参数都不为空,则触发构建.  

先用参数构建 odm-job 看看:
![参数构建](https://res.cloudinary.com/dkzvjuptx/image/upload/v1576853222/jenkinsjobtrigger/15_hutjs5.png) 
可以看到触发了 sign-job 参数也成功传递过来了,
![触发](https://res.cloudinary.com/dkzvjuptx/image/upload/v1576853222/jenkinsjobtrigger/16_zvkcho.png) 
再使用空参数构建 odm-job 看看:  
看得出 sign-job 并没有被触发
![空参数构建](https://res.cloudinary.com/dkzvjuptx/image/upload/v1576853222/jenkinsjobtrigger/17_lbjgon.png) 

#### 两种方式的不同及使用场景  
**不同:** Post-build Actions 不能使用条件选择,只要被添加在后面,就一定会被触发; Conditional step 可以使用条件选择,满足一定的条件才可以被触发.
**使用场景:** Post-build Actions 适合用在前者构建结束后必触发后者的场景; Conditional step 用于基于前者的构建情况,有选择性地构建后者,甚至可以设置有多个 Conditional step ,根据不同的情况选择构建不同功能的 Job ,类似编程语言中的条件语句,或者说类似设计模式中的策略模式(但没有满足开闭原则)
