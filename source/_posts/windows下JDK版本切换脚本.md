---
title: windows下JDK版本切换脚本
subtitle: jdk版本切换
description: jdk版本切换 
keywords: [jdk,版本,切换]
author: liyz
date: 2018-09-18 10:07:04
tags: [tool,jdk,思考]
category: [tool]
---

# 初衷

前几天在一个技术交流群中看到技术人应该怎样去扩展自己的知识，去发现新的技术。其中有一条就是：**当你对当前的工作感到厌倦的时候就应该去思考是否可以对其进行优化**，比如我在重复的打开环境变量，修改JDK版本号的时候，就为每天都要进行此操作而感到厌倦，以至于内心开始拒绝去切换JDK版本，拒绝去做需要在另一个版本上的工作。

# research过程

首先我搜索的关键字是`jdk版本切换`，其搜索结果都是怎样设置多JDK版本，怎样去修改环境变量。但是这些结果并不是我想要的，不过我确实是想要切换JDK版本啊，为什么没搜到结果呢。

**当搜索不到结果的时候，首先考虑我们的搜索关键字是否准备**

再次思考，我其实不是想切换JDK版本，而是想更方便的切换JDK版本，怎样会更方便呢，比如只点一个按钮即可。那其实可以通过脚本去实现这个功能，所以我的搜索条件变成了`windos切换JDK版本脚本`然后就搜索到了想要的结果。

# 脚本

## 运行截图

![运行截图](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/tool/script/jdkVSwitch.png)

## 脚本内容
```bash
@echo off

rem --- Base Config 配置JDK的安装目录 ---
:init 
set JAVA_HOME_1_8=C:\Program Files\Java\jdk8
set JRE_HOME_1_8=C:\Program Files\Java\jre8

set JAVA_HOME_1_7=C:\Program Files\Java\jdk7
set JRE_HOME_1_7=C:\Program Files\Java\jre7
:start 
echo 当前使用的JDK 版本: 
java -version 
echo. 
echo ============================================= 
echo jdk版本列表: 
echo  jdk1.8 
echo  jdk1.7
echo ============================================= 

:select
set /p opt=请输入JDK版本。[7代表jdk1.7],[8代表jdk1.8]： 
if %opt%==8 (
    set TARGET_JAVA_HOME=%JAVA_HOME_1_8%
	set TARGET_JRE_HOME=%JRE_HOME_1_8%
)
if %opt%==7 (
    set TARGET_JAVA_HOME=%JAVA_HOME_1_7%
	set TARGET_JRE_HOME=%JRE_HOME_1_7%
)


echo 当前选择的Java路径:
echo JAVE_HOME:%TARGET_JAVA_HOME%
echo JRE_HOME:%TARGET_JRE_HOME%

wmic ENVIRONMENT where "name='JAVA_HOME'" delete
wmic ENVIRONMENT create name="JAVA_HOME",username="<system>",VariableValue="%TARGET_JAVA_HOME%"

wmic ENVIRONMENT where "name='JRE_HOME'" delete
wmic ENVIRONMENT create name="JRE_HOME",username="<system>",VariableValue="%TARGET_JRE_HOME%"

rem -- refresh env ---
call RefreshEnv

echo 请按任意键退出!   
pause>nul

@echo on
```

## 脚本下载

- [RefreshEnv.exe](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/tool/script/RefreshEnv.exe)
- [switchVersion.bat](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/tool/script/switchVersion.bat)

# 注意事项

- 是否需要配置`JRE_HOME`和安装JDK的路径有关系，下图是我的安装路径

![jdk安装路径](https://sustblog.oss-cn-beijing.aliyuncs.com/blog/2018/tool/script/jdkPath.png) 

- 需要修改`JAVA_HOME`的值为你对应的JDK安装路径
- 需要以管理员权限运行脚本



