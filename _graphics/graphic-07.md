---
title: "Vulkan RenderPass的宏观理解"
collection: graphics
permalink: /graphics/graphic-07
excerpt: ' '
date: 2024-07-18
citation: 'Joe-Bi. (2024). &quot;Vulkan RenderPass.&quot; <i>GitHub Joe-Bi of Bugs</i>'
---
   
发现很多人在学习理解vulkan renderpass时，都是去学习它的数据结构和相关函数API的调用，对图形渲染管线原理和其中的一些抽象概念理解甚少。

熟悉cuda/opencl的话，可以很明显感受到vulkan的设计理念与它们趋同，命令队列（渲染队列）与执行队列（执行流）分离，分层设计，解耦执行，更灵活，更高效。

基于以上这个原因，以前在OpenGL内部实现的renderpass功能就被挪到用户层面配置了，让用户有更多的控制权。开发人员最熟悉自己当前的业务逻辑，让他们去设置，可以最大化优化性能。

熟悉OpenGL光栅化流水线的话，那么对renderpass的简单宏观理解可以是这样的：  

<font face="黑体" size=4>
pipeline是帧绘制需要的动作状态、条件、参数设置等等的所有集合，包含从顶点输入到顶点着色、cull，光栅化、scissor、片段着色、各种测试混合、ROP等这一些列参数状态的所有设置。
最后输出到与renderpass绑定的framebuffer。而renderpass的attchment这些则指明了如何处理渲染结果到framebuffer，subpass又指明了如何引用这些attachment，这是一个长长的流程和细致的控制。
所有的这些都交给了用户去设置。</font>  

### 一句话总结，renderpass是对pipeline一次渲染过程的描述控制，是过程，如果当作结果的话来看的话，可以不太正确的把renderpass类比为framebuffer。











