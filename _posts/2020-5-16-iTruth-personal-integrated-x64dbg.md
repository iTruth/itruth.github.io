---
title: iTruth整合版x64dbg(最新中文版) 内置配色方案+插件
date: 2020-5-16 22:36:00 +0800
categories: [Tool, x64dbg]
tags: [x64dbg, debugger, debug, tools]
---

包含个人仿吾爱OD的配色  
![iTruth x64dbg](\assets\img\2020-5-16-iTruth-personal-integrated-x64dbg\itruth-x64dbg.jpg)

### 包含的拓展功能
* 包含两款反反调试插件(ScyllaHide和SharpOD), 轻松过掉各大程序的反调试
* 两款脱壳插件(OllyDumpEx_X64Dbg和Scylla)
* 中文搜索支持(x64dbg_tol)
* 类似IDA的F5反汇编至C++代码(snowman)
* API断点插件(x64dbgApiBreak)
* 自动调试子进程(DbgChild)
* 类似CE的内存搜索(ClawSearch)
* PE头浏览支持(peviewer)
* 提取PE头信息(PEHeaderDumpUtilities)
* 程序伪调试支持(NaiHeQiao)
* 非侵入式调试支持(GhostDbg)
* 特征码的提取和搜索支持(BaymaxTools)
* 自动标注PEB支持(LabelPEB)
* 类似Ollydbg函数分析(xAnalyzer)
* 多行汇编支持(multiasm_x64dbg)
* 检查安全设置(checksec)
* NtProtectVirtualMemory失败时强制设置页保护(ForcePageProtection)
* 自动生成Yara规则(YaraGen)
* 事件拦截支持(xHotSpots)
* 调试对象传递支持(OllyMigrate_X64Dbg)
* 解决in3断点的异常(StepInt3)
* 调试器透明度调整支持(TransX64Dbg)
* 实用工具集(SwissArmyKnife)
* 实用脚本合集(在Scripts文件夹下)

### 注意
x64dbg和全部插件均为最新版.
ScyllaHide代码已经更新了好几个月了但是还没有Release, 所以ScyllaHide是我自己编译的最新版.

如果需要驱动级(ring0)反反调试请自行安装TitanHide, 在TitanHide_archive.rar里
请勿使用实体机运行TitanHide, 它会使系统变得不再安全. 此整合版x64dbg默认不安装TitanHide
TitanHide_archive.rar 密码: TitanHide

xHotSpots需要安装.NET框架才能正常执行, 否则会使x64dbg崩溃. 考虑到兼容性问题默认不安装xHotSpots(位于plugins\disable文件夹下)
### 下载
#### 第一版
[点此下载iTruth个人整合版x64dbg](https://www.lanzous.com/icphw1c)

#### 第二版 2020/5/18更新
更新日志:
* 考虑到兼容性问题, 默认禁用xHotSpots(位于plugins\disable文件夹下)
* 修复内存寻址缩放大小配色不合理

[点此下载iTruth个人整合版x64dbg-v2](https://www.lanzous.com/icr097a)

