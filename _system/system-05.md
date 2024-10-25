---
title: "Android画面显示流程"
collection: system
permalink: /system/system-05
excerpt: ' '
date: 2024-10-25
citation: 'Joe-Bi. (2024). &quot;Android画面绘制显示流程.&quot; <i>GitHub Joe-Bi of Bugs</i>'
---


一直想对Android渲染绘制到画面上屏显示的全流程做个系统性地了解与学习，没想到努比亚技术团队的同仁已做，而且写的非常全面好懂。

### 文章内容有以下几个方面：
1) 我们肉眼可见的摸得着的硬件外观和它们的各个连接方式；<br />
2) 每个硬件部件由来以及功能与作用，它们的名称以及与软件代码层面的抽象对应关系；<br />
3）渲染绘制需要的buffer是如何产生的以及如何合成的最后一帧画面的，涉及到SurfaceFlinger和HWC的完整链路；<br />
4）产生的帧数据是如何送到屏幕显示的，这涉及到linux DRM，介绍了DRM的功能和模块以及简单使用举例，每个模块又是硬件功能的抽象；<br />

### 文章地址如下：
[Android画面显示流程分析(1)](https://www.jianshu.com/p/df46e4b39428)<br />
[Android画面显示流程分析(2)](https://www.jianshu.com/p/f96ab6646ae3)<br />
[Android画面显示流程分析(3)](https://www.jianshu.com/p/3c61375cc15b)<br />
[Android画面显示流程分析(4)](https://www.jianshu.com/p/7a18666a43ce)<br />
[Android画面显示流程分析(5)](https://www.jianshu.com/p/dcaf1eeddeb1)<br />

感谢！

