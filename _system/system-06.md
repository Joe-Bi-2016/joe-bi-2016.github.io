---
title: "Android Camera2 API与OES纹理在使用过程中的注意事项"
collection: system
permalink: /system/system-06
excerpt: ' '
date: 2024-12-18
citation: 'Joe-Bi. (2024). &quot;Android Camera2 API & EGL Image纹理.&quot; <i>GitHub Joe-Bi of Bugs</i>'
---


最近做了一个有关照相机和滤镜渲染引擎的项目，复盘下在开发过程中碰到的坑点。

#### 项目地址：<https://github.com/Joe-Bi-2016/CameraFilterRenderEngine>

### 项目目的：
1) 开发一套较为通用的OpenGL ES的滤镜渲染引擎；<br />
2) 渲染与显示组件解耦，没有关联性；<br />
3）功能上支持相机、视频、图片的特效编辑，以及GLSL动画；<br />
3）支持非线编，多路滤镜特效算法renderpass在渲染时间线上并行，动态切换；<br />
4）统一支持相机、视频播放器、相册编辑、非线编等业务场景；<br />
5）解耦特效算法与渲染引擎，支持滤镜特效算法的动态脚本导入，自动管理；<br />
6）支持GLSL特效算法网[shaderToy](https://www.shadertoy.com/)；<br />
7）考虑后续支持AE等动画脚本；<br />

### Camera2 API：
在设备旋转时，camera1 api是通过cmaera对象的setDisplayOrientation函数来做的，无需考虑TextureView、SurfaceView等
预览显示组件的旋转方向。TextureView不应该通过setTransform函数设置它的旋转Matrix。

使用camera2 api时，有以下的不同。
1）使用TextureView作为预览显示组件时，必须设置它的旋转Matrix，即通过TextureView的setTransform函数来设置从camera获取的参数计算好的Matrix给它，这里有个坑就是如果在第一次调用setTransform时传递了null参数，那么再传递非空的matrix参数会无效，导致效果上看起来
设备旋转后，画面是不会旋转的；

2）使用SurfaceView作为预览显示组件时，不能设置它的旋转Matrix.

3) 使用OES纹理生成一个SurfaceTexture给Camera作为预览时，如果预览显示组件是TextureView，只需要通过getTransformMatrix函数获取SurfaceTexture的纹理旋转矩阵，将其设置为GLSL中的纹理坐标变换矩阵就OK了，设备旋转预览画面都正常；

4) 如果预览显示组件是SurfaceView时，情况就又不一样了。和TextureView中使用OES纹理生成一个SurfaceTexture给Camera作为预览的过程一样，设置完GLSL中的纹理坐标变换矩阵后，设备旋转时预览画面方向却没有旋转。如果使通过Matrix的rotateM、setRotateM等方法计算出正确的纹理旋转变换矩阵传递给纹理变换，画面也不正确，花屏，扭曲。为了验证这个问题，我将所有数据复制出来在PC端上使用glm执行相同的计算，矩阵结果完全一致，但PC上的画面就是正常的，画面旋转与预期一致。
这看起来是个平台相关的问题，至少在高通平台(小米手机14)上验证是这样的。
那么只有通过设备旋转角，使用Matrix计算出旋转矩阵后，设置给GLSL中的位置Position模型矩阵了，这样修改后，画面果然正确，不再扭曲花屏；

### EGL Image纹理：

在ndk层创建fbo，绑定EGL Image纹理作为存储渲染结果，当获取pixels时可以通过AHardwareBuffer_lock函数直接获取AHardwareBuffer的buffer，比glReadPixels性能上要优越的多。但这里有个非常隐蔽的坑。
当生成EGL Image纹理时，如果width或height是非4的倍数时，其内存在分配时需要4字节对齐，那么行长stride并不是创建时的width，一定比width大。
那么如何知道系统分配了多大的stride呢，是公式stride = width + (4 - width % 4)计算的结果吗？
使用这个公式计算的stride，逐行copy出AHardwareBuffer_lock获取的数据，也是花屏。

那这个stride究竟该是多少，如何计算或如何获知呢。经过研究观察，在通过AHardwareBuffer_Desc创建AHardwareBuffer时，该结构体有个
stride项，那么是否有哪个函数可以获取这个stride呢。一番查找，找到了AHardwareBuffer_describe函数，通过这个函数就可以获得创建
AHardwareBuffer的AHardwareBuffer_Desc了，也就能获取正确的stride了。使用这个stride代替公式计算的那个值，验证完全正确，不再花屏了。

这就是这个项目开发中遇到的几个极品问题，记录一下，防避雷。
