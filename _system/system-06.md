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
预览显示组件的旋转方向。TextureView不应该通过<font color=red size=3>setTransform</font>函数设置它的旋转Matrix。

使用camera2 api时，有以下的不同。
1）使用TextureView作为预览显示组件时，必须设置它的旋转Matrix，即通过TextureView的setTransform函数来设置从camera获取的参数计算好的Matrix给它，这里有个坑就是如果在第一次调用<font color=red size=3>setTransform时传递了null参数</font>，那么再传递非空的matrix参数会无效，导致效果上看起来
设备旋转后，画面是不会旋转的；

2）使用SurfaceView作为预览显示组件时，不能设置它的旋转Matrix.

3) 使用OES纹理生成一个SurfaceTexture给Camera作为预览时，如果预览显示组件是TextureView，只需要通过getTransformMatrix函数获取SurfaceTexture的纹理旋转矩阵，将其设置为GLSL中的纹理坐标变换矩阵就OK了，设备旋转预览画面都正常；
```
float[] textureMatrix = new float[16];
mSurfaceTexture.getTransformMatrix(textureMatrix);
mNativeEglRender.native_setParamMatrix(0, mTextureMatName, textureMatrix);
```
4) 如果预览显示组件是SurfaceView时，情况就又不一样了。和TextureView中使用OES纹理生成一个SurfaceTexture给Camera作为预览的过程一样，设置完GLSL中的纹理坐标变换矩阵后，设备旋转时预览画面方向却没有旋转。如果使通过Matrix的rotateM、setRotateM等方法计算出正确的纹理旋转变换矩阵传递给纹理变换，画面也不正确，花屏，扭曲。为了验证这个问题，我将所有数据复制出来在PC端上使用glm执行相同的计算，矩阵结果完全一致，但PC上的画面就是正常的，画面旋转与预期一致。
这看起来是个平台相关的问题，至少在高通平台(小米手机14)上验证是这样的。
那么只有通过设备旋转角，使用Matrix计算出旋转矩阵后，设置给GLSL中的位置Position模型矩阵了，这样修改后，画面果然正确，不再扭曲花屏；
```
float[] modelMatrix = new float[16];
setIdentityM(modelMatrix, 0);
// Note: camera2 api did not set rotation matrix for SurfaceTexture, so need do transform in here
if (mCameraManager instanceof Camera2Manager) {
    int displayRotatimCameraMangetDisplayOrientation();
    boolean isFrontCame((Camera2ManamCameraManaisFrontCmaera());
    int degree = 0;
    switch (displayRotation) {
        case 0:
            if(isFrontCamera)
                degree = 270;
            else
                degree = 90;
            break;
        case 90:
            if(isFrontCamera)
                degree = 180;
            else
                degree = 0;
            break;
        case 180:
            if(isFrontCamera)
                degree = 90;
            else
                degree = 270;
            break;
        case 270:
            if(isFrontCamera)
                degree = 0;
            else
                degree = 180;
            break;
    }
    setRotateM(modelMatrixdegree, 0, 0, 1); // modelMatrixdegree been transport to vert glsl
}

// Vert shader set matrix of position
gl_Position = modelMatrix * vec4(position, 1.0);

```

### EGL Image纹理：

在ndk层创建fbo，绑定EGL Image纹理作为存储渲染结果，当获取pixels时可以通过AHardwareBuffer_lock函数直接获取AHardwareBuffer的buffer，比glReadPixels性能上要高很多。直接copy出来，画面正常，没问题，但这里有个非常隐蔽的坑。

1) 当生成EGL Image纹理时，如果width或height是奇数非4的倍数时(起决于指定AHardwareBuffer_Desc中的format格式是4字节还是3字节又或2字节)，其内存在分配时会字节对齐，那么行长stride与创建时指定的width并不相等，一般比width大，直接copy就会导致花屏。

那如何知道系统分配了多大的stride呢，是公式stride = width + (4 - width % 4)计算的结果吗？
使用这个公式计算的stride验证一下，注意需要逐行copy出AHardwareBuffer_lock获取的数据，还是花屏。

那这个stride究竟是多少，如何计算或如何获知呢。看下面代码，经过观察研究，在创建AHardwareBuffer指定AHardwareBuffer_Desc数据结构时，该结构体有个stride项，是不是有哪个函数可以获取这个stride呢。一番搜索查阅，找到了AHardwareBuffer_describe这个函数，
通过这个函数就可以获得创建AHardwareBuffer的实际AHardwareBuffer_Desc值，也就能获取正确的stride了。
```
AHardwareBuffer_Desc desc = {pixelsWide, pixelsHigh,
                            1,
                            format,
                            AHARDWAREBUFFER_USAGE_CPU_WRITE_OFTEN|AHARDWAREBUFFER_USAGE_GPU_SAMPLED_IMAGE,
                            pixelsWide,
                            0 ,
                            0};
							
AHardwareBuffer_describe(mInBuffer, &desc);
stride = desc.stride;
```

使用这个stride代替公式计算的那个值，验证，完全正确，不再花屏了。

2) 使用EGL Image作为fbo的Texture时，映射AHardwareBuffer的buffer读取时，<font color=red>需要在eglSwapBuffer之前，执行glFinish或glFlush命令</font>，这样才能获取渲染后的结果。但是这样也会
打断GPU自己的执行队列流程，导致功耗会高些，需要在渲染引擎中对这种特殊情况做处理。
```
mMainThreadRoot->onDo(renderPassIndex);

if(mDefaultFbo && mDefaultFbo.get()){
    // If fbo uses EGL textures, then glFinish() should be called, otherwise there is no data output to EGLImage
    sharedRenderTarget target = mDefaultFbo->getRenderTarget();
    if (target && typeid(*(target->getTexture())) == typeid(GLEGLImageTexture))
        glFinish();
}

mMainThreadRoot->getEGLContext()->swapBuffers();
```

这就是这个项目开发中遇到的几个极品问题，记录一下，防避坑。

