---
title: "OpenGL ES学习--片段着色器"
catalog: true
toc_nav_num: true
date: 2019-08-15 17:20:32
subtitle: "About OpenGL ES"
header-img: "/img/article_header/article_header.png"
busuanzi: true
tags:
- iOS

---

> 介绍片段着色器编写方面的细节。

# 固定功能片段着色器

在研究片段着色器的细节之前，值得简单地回顾一下老式的固定功能片段管线，这将帮助我们理解老式的固定功能管线映射到片段着色器的方式。在转到更先进的片段编程技术之前，这是很好的起点。
在OpenGL ES 1.1（以及固定功能桌面OpenGL）中，可以使用一组有限的方程式，确定如何组合片段着色器的不同输入。在固定功能管线中，实际上可以使用3中输入：插值顶点颜色、纹理颜色和常量颜色。顶点颜色通常保存一个预先计算的颜色或者顶点照明计算的结果。纹理颜色来自于使用图元纹理坐标绑定的纹理中读取的值，而常量颜色可以对每个纹理单元设置。
用于组合这些输入的这一组方程式相当有限，例如，在OpenGL ES 1.1中，可用的方程式在下表中列出。方程式的输入A、B、C可能来自于顶点颜色、纹理颜色或者常量颜色。

RGB组合函数 | 方程式
- | -
REPLACE | A
MODULATE | AxB
ADD | A+B
ADD_SIGNED | A+B-0.5
INTERPOLATE | AxC+Bx(1-C)
SUBTRACT | A-B
DOT3_RGB(and DOT3_RGBA) | 4x((A.r-0.5)x(B.r-0.5)+(A.g-0.5)x(B.g-0.5)+(A.b-0.5)x(B.b-0.5))

即使利用这组有限的方程式，实际上也可以实现大量有趣的特效，但是，这比起可编程还有很大的距离，因为片段管线只能以非常固定的一组方式配置。
为什么要回顾这段历史呢？因为它能帮助我们理解如何用着色器实现传统的固定功能技术。例如，假定我们已经用一个基本纹理贴图配置了固定功能管线，希望用顶点颜色调制该贴图。在固定功能OpenGL ES（或者OpenGL）中，我们将启用一个纹理单元，选择MODULATE的组合方程式，并且将方程式的输入设置为顶点颜色和纹理颜色。下面提供OpenGL ES 1.1实现以上功能的代码，作为参考：

``` objc
glTexEnvi(GL_TEXTURE_ENV, GL_TEXTURE_ENV_MODE, GL_COMBINE);
glTexEnvi(GL_TEXTURE_ENV, GL_COMBINE_RGB, GL_MODULATE);
glTexEnvi(GL_TEXTURE_ENV, GL_SOURCE0_RGB, GL_PRIMARY_COLOR);
glTexEnvi(GL_TEXTURE_ENV, GL_SOURCE1_RGB, GL_TEXTURE);
glTexEnvi(GL_TEXTURE_ENV, GL_COMBINE_ALPHA, GL_MODULATE);
glTexEnvi(GL_TEXTURE_ENV, GL_SOURCE0_ALPHA, GL_PRIMARY_COLOR);
glTexEnvi(GL_TEXTURE_ENV, GL_SOURCE1_ALPHA, GL_TEXTURE);
```

这段代码配置固定功能管线，以执行主颜色（顶点颜色）和纹理颜色之间的调制（AxB）。这些代码在OpenGL ES 3.0中已经不存在了。我们只是试图说明这些功能如何映射到片段着色器。在片段着色器中，同样的调制计算可以用如下代码实现：

``` objc
#version 300 es
precision mediump float;
uniform sampler2D s_tex0;
in vec2 v_texCoord;
in vec4 v_primaryColor;
layout(location = 0) out vec4 outColor;
void main()
{
    outColor = texture(s_tex0, v_texCoord) * v_primaryColor;
}
```

片段着色器执行的操作与固定功能设置执行的操作完全相同。纹理值从一个采样器（绑定到纹理单元0）读取，并用一个2D纹理坐标查找该值。然后，纹理读取的结果和顶点着色器传递的输入值`v_primaryColor`相乘。在这个例子中，顶点着色器已经将颜色传递给片段着色器。
可以编写一个片段着色器，执行任何固定功能纹理组合设置的等价计算。当然，也可以编写着色器，执行比固定功能复杂得多的不同计算。

# 片段着色器概述

片段着色器为片段操作提供了通用功能的可编程方法。片段着色器的输入由如下部分组成：
- 输入（或者可变值）--顶点着色器生成的插值数据。顶点着色器输出跨图元进行插值，并作为输入传递给片段着色器。
- 统一变量--片段着色器使用的状态，这些常量值在每个片段上不会变化。
- 采样器--用于访问着色器中的纹理图像。
- 代码--片段着色器源代码或者二进制代码，描述在片段上执行的操作。
片段着色器的输出时一个或者多个片段颜色，传递到管线的逐片段操作部分（输出颜色的数量取决于使用多少个颜色附着）。片段着色器的输入和输出如下图所示：

![OpenGL ES 3.0片段着色器](/img/article/20190815/1.png)

## 内建特殊变量

OpenGL ES 3.0有内建特殊变量，这些变量由片段着色器输出或者作为片段着色器的输入。片段着色器中可用的内建特殊变量如下所示：
- `gl_FragCoord`--片段着色器中的一个只读变量。这个变量保存片段的窗口相对坐标(x, y, z, 1/w)。在一些算法中，知道当前片段的窗口坐标是很有用的。例如，可以使用窗口坐标作为某个随机噪声贴图纹理读取的偏移量，噪声贴图的值用于旋转阴影贴图的过滤核心。这种技术用于减少阴影贴图的锯齿失真。
- `gl_FrontFacing`--片段着色器中的一个只读变量。这个布尔变量在片段是正面图元的一部分时为true，否则为false。
- `gl_PointCoord`--一个只读变量，可以在渲染点精灵时使用。它保存点精灵的纹理坐标，这个坐标在点光栅化期间自动生成，处于[0, 1]区间内。
- `gl_FragDepth`--一个只写输出变量，在片段着色器中写入时，覆盖片段的固定功能深度值。这一功能应该谨慎使用（只在必要时），因为它可能禁用许多GPU的深度优化。例如，许多GPU有所谓的”Early-Z“功能，在执行片段着色器之前进行深度测试。使用Early-Z的好处是不能通过深度测试的片段永远不会被着色（从而保护了性能）。但是，使用`gl_FragDepth`时，必须禁用该功能，因为GPU在执行片段着色器之前不知道深度值。

## 内建常量

下面是与片段着色器有关的内建常量：

``` objc
const mediump int gl_MaxFragmentInputVectors = 15;
const mediump int gl_MaxTextureImageUnits = 16;
const mediump int gl_MaxFragmentUniformVectors = 224;
const mediump int gl_MaxDrawBuffers = 4;
const mediump int gl_MinProgramTexelOffset = -8;
const mediump int gl_MaxProgramTexelOffset = 7;
```

内建常量的描述如下最大项：
- `gl_MaxFragmentInputVectors`--片段着色器输入（或者可变值）的最大数量。所有ES 3.0实现支持的最小值为15。
- `gl_MaxTextureImageUnits`--可用纹理图像单元的最大数量。所有ES 3.0实现支持的最小值为15。
- `gl_MaxFragmentUniformVectors`--片段着色器内可以使用的vec4统一变量项目的最大数量。所有ES 3.0实现支持的最小值为224。开发者实际可以使用的vec4统一变量项目的数量在不同实现以及不同片段着色器中可能不一样。
- `gl_MaxDrawBuffers`--多重渲染目标（MRT）的最大支持数量。所有ES 3.0实现支持的最小值为4。
- `gl_MinProgramTexelOffset/gl_MaxProgramTexelOffset`--通过内建ESSL函数texture*Offset()偏移参数支持的最大和最小偏移量。
每个内建常量所指定的值是所有OpenGL ES 3.0实现必须支持的最小值。不同的实现可能支持大于上述最小值的数值。片段着色器内建数值的实际硬件特定值可以从API代码中查询。下面的代码说明如何查询`gl_MaxTextureImageUnits`和`gl_MaxFragmentUniformVectors`的值。

``` objc
GLint maxTextureImageUnits, maxFragmentUniformVectors;
glGetIntegerv(GL_MAX_TEXTURE_IMAGE_UNITS, &maxTextureImageUnits);
glGetIntegerv(GL_MAX_FRAGMENT_UNIFORM_VECTORS, &maxFragmentUniformVectors);
```

# 用着色器实现固定功能技术

我们已经对片段着色器进行了概述，现在将演示如何用着色器实现几种固定功能技术。OpenGL ES 1.x和桌面OpenGL中的固定功能管线提供API，可以执行多重纹理、雾化、Alpha测试和用户裁剪平面。尽管这些技术在OpenGL ES 3.0中都没有明确提供，但是都可以用着色器实现。

## 多重纹理

多重纹理，是片段着色器中非常常见的操作，用于组合多个纹理贴图。在顶点着色器中以不同的方式组合纹理很简单，就是采用着色语言中存在的许多运算符和内建函数。使用这些技术，能够轻松地实现OpenGL ES以前版本中的固定功能片段管线所能实现的效果。

下面这个实力加载一个基本纹理贴图和照明贴图纹理：

``` objc
#version 300 es
precision mediump float;
in vec2 v_texCoord;
layout(location = 0) out vec4 outColor;
uniform sampler2D s_baseMap;
unifomr sampler2D s_lightMap;
void main()
{
    vec4 baseColor;
    vec4 lightColor;
    baseColor = texture(s_baseMap, v_texCoord);
    lightColor = texture(s_lightMap, v_texCoord);
    // Add a 0.25 ambient light to the texture light color
    outColor = baseColor * (lightColor + 0.25);
}
```

这个片段着色器有两个采样器，每个纹理使用一个。设置纹理单元和采样器的相关代码如下：

``` objc
// Bind the base map
glActiveTexture(GL_TEXTURE0);
glBindTexture(GL_TEXTURE_2D, baseMapTexId);
// Set the base map sampler to texture unit 0
glUniformli(baseMapLoc, 0);
// Bind the light map
glActiveTexture(GL_TEXTURE1);
glBindTexture(GL_TEXTURE_2D, lightMapTexId);
// Set the light map sampler to texture unit 1
glUniform(lightMapLoc, 1);
```

可以看到，上述代码将各个纹理对象绑定到纹理单元0和1。为采样器设置数值，将采样器绑定到对应的纹理单元。在这个例子中，使用单一纹理坐标从两个贴图中读取。在典型的照明贴图处理中，基本贴图和照明贴图应该有一组单独的纹理坐标。照明贴图通常混合到单一的大型纹理中，纹理坐标可以使用离线工具生成。

##  雾化

> 待续

## Alpha测试（使用Discard）

3D应用程序中使用的常见特效之一是绘制某些片段中完全透明的图元，这对于渲染链状栅栏等物体很有用。用几何形状表现栅栏要求大量的图元，然而，在纹理中存储一个遮罩值指定哪些纹素应该是透明的是使用几何形状的另一种方法。例如，可以将链状栅栏保存在一个RGBA纹理中，其中RGB值表示栅栏的颜色，A值表示纹理是否透明的遮罩值。这样会很容易地用一个或者两个三角形并且在片段着色器中遮蔽像素来渲染一个栅栏。
在传统的固定功能渲染中，这种特效用Alpha测试实现。Alpha测试允许你指定一个比较测试，如果片段的Alpha值和参考值的比较失败，该片段将被删除。也就是说，如果一个片段无法通过Alpha测试，该片段便不会被渲染。在OpenGL ES 3.0中没有固定功能Alpha测试，但是在片段着色器中可以使用`discard`关键字实现相同的效果。

``` objc
#version 300 es
precision mediump float;
uniform sampler2D baseMap;
in vec2 v_texCoord;
layout(location = 0) out vec4 outColor;
void main()
{
    vec4 baseColor = texture(baseMap, v_texCoord);
    // Discard all fragments with alpha value less than 0.25
    if (baseColor.a < 0.25) {
        discard;
    } else {
        outColor = baseColor;
    }
}
```

在这个片段着色器中，纹理是一个四通道的RGBA纹理。Alpha通道用于Alpha测试。Alpha颜色与0.25比较；如果小于该值，就用`discard`”杀死“片段。
否则，用纹理颜色绘制片段。这种技术可以用于通过简单地改变对比或者Alpha参考值来实现Alpha测试。

## 用户裁剪平面

> 待续

# 总结

这篇文章介绍了使用片段着色器的多种渲染技术，聚焦于实现OpenGL ES 1.1固定功能部分技术的片段着色器。使用可编程片段着色器时，几乎可以实现无限的着色技术。