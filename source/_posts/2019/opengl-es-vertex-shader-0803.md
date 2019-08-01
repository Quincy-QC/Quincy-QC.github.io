---
title: "OpenGL ES学习--顶点着色器"
catalog: true
toc_nav_num: true
date: 2019-08-03 15:45:11
subtitle: "About OpenGL ES"
header-img: "/img/article_header/article_header.png"
busuanzi: true
tags:
- iOS

---

> 本章首先概述顶点着色器，包括其输入和输出。然后，讨论几个例子，说明如何编写顶点着色器。

# 顶点着色器概述

顶点着色器提供顶点操作的通用可编程方法。下图展示了顶点着色器的输入和输出，顶点着色器的输入包括：
- 属性--用于顶点数组提供的逐顶点数据；
- 统一变量和统一变量缓冲区--顶点着色器使用的不变数据；
- 采样器--代表顶点着色器使用的纹理的特殊统一变量类型；
- 着色器程序--顶点着色器程序源代码或者描述在操作顶点的可执行文件。

顶点着色器的输出称作顶点着色器输出变量。在图元光栅化阶段，为每个生成的片段计算这些变量，并作为片段着色器的输入传入。

![OpenGL ES 3.0顶点着色器](/img/article/20190803/1.png)

## 顶点着色器内建变量

顶点着色器的内建变量可以分为特殊变量（顶点着色器的输入输出）、统一状态（如深度范围）以及规定最大值（如属性数量、顶点着色器输出变量数量和统一变量数量）的常量。

### 内建特殊变量

OpenGL ES 3.0有内建的特殊变量，它们可以作为顶点着色器的输入或者在之后成为片段着色器输入的顶点着色器输出，或者片段着色器的输出。顶点着色器可用的内建特殊变量如下：
- `gl_VertexID`是一个输入变量，用于保存顶点的整数索引。这个整数型变量用`highp`精度限定符声明；
- `gl_InstanceID`是一个输入变量，用于保存实例化绘图调用中图元的实例编号。对于常规的绘图调用，该值为0。`gl_InstanceID`是一个整数型变量，用`highp`精度限定符声明；
- `gl_Position`用于输出顶点位置的裁剪坐标。该值在裁剪和视口阶段用于执行相应的图元裁剪以及从裁剪坐标到屏幕坐标的顶点位置转换。如果顶点着色器未写入`gl_Position`，则`gl_Position`的值未定义。`gl_Position`是一个浮点变量，用`highp`精度限定符表明；
- `gl_PointSize`用于写入以像素表示的点精灵尺寸，在渲染点精灵时使用。顶点着色器输出的`gl_PointSize`值被限定在OpenGL ES 3.0实现支持的非平滑点大小范围之内。`gl_PointSize`是一个浮点变量，用`highp`精度限定符声明；
- `gl_FrontFacing`是一个特殊变量，但不是由顶点着色器直接写入的，而是根据顶点着色器生成的位置值和渲染的图元类型生成的。它是一个布尔变量。

### 内建统一状态

顶点着色器内可用的唯一内建统一状态是窗口坐标中的深度范围。这由内建统一变量名`gl_DepthRange`给出，该变量声明为`gl_DepthRangeParameters`类型的统一变量。

``` objc
struct gl_DepthRangeParameters {
    highp float near; // near Z
    highp float far;  // far Z
    highp float diff; // far - near
}

uniform gl_DepthRangeParameters gl_DepthRange;
```

### 内建常量

顶点着色器内还有如下内建常量：

``` objc
const mediump int gl_MaxVertexAttribs = 16;
const mediump int gl_MaxVertexUniformVectors = 256;
const mediump int gl_MaxVertexOutputVectors = 16;
const mediump int gl_MaxVertexTextureImageUnits = 16;
const mediump int gl_MaxCombinedTextureImageUnits = 32;
```

内建常量描述如下最大项：
- `gl_MaxVertexAttribs`是可以指定的顶点属性的最大数量，所有ES 3.0实现都支持的最小值为16；
- `gl_MaxVertexUniformVectors`是顶点着色器中可以使用的`vec4`统一变量项目的最大数量。所有ES 3.0实现都支持的最小值为256。开发人员可以使用的`vec4`统一变量项目数量在不同实现和不同顶点着色器中可能不同。例如，有些实现可能将顶点着色器中使用的用户指定字面值计入统一变量限制内。在其他情况下，根据顶点着色器是否使用内建的超越函数，可能需要包含特定于实现的统一变量（或者常量）。目前没有应用程序可以用于找出在特定顶点着色器中可使用的统一变量项目数量的机制。顶点着色器编译失败时，编译日志可能提供关于使用的统一变量项目数量的相关信息。但是，编译日志返回的信息是特定于实现的；
- `gl_MaxVertexOutputVectors`是输出向量的最大数量--也就是顶点着色器可以输出的`vec4`项目数量。所有ES 3.0实现都支持的最小值是16个`vec4`项目；
- `gl_MaxVertexTextureImageUnits`是顶点着色器中可用的纹理单元的最大数量，最小值为16；
- `gl_MaxCombinedTextureImageUnits`是顶点和片段着色器中可用纹理单元最大数量的总和，最小值为32.

每个内建常量指定的值是所有OpenGL ES 3.0实现必须支持的最小值，各种实现可能支持超过上面所述的最小值的常量值。实际支持的值可以用`glGetIntegerv`函数查询，查询参数为`GL_MAX_VERTEX_ATTRIBS`，`GL_MAX_VERTEX_UNIFORM_VECTORS`，`GL_MAX_VARYING_VECTORS`，`GL_MAX_VERTEX_TEXTURE_IMAGE_UNITS`，`GL_MAX_COMBINED_TEXTURE_IMAGE_UNITS`。

## 顶点着色器中的统一变量限制数量

`gl_MaxVertexUniformVectors`描述了可以用于顶点着色器的统一变量的最大数量。任何兼容的OpenGL ES 3.0实现必须支持的`gl_MaxVertexUnifromVectors`最小值为256个`vec4`项目。统一变量存储用于存储如下变量：
- 用统一变量限定符声明的变量
- 常数变量
- 字面值
- 特定于实现的常量

顶点着色器中使用的统一变量和用`const`限定符声明的变量、字面值和特定于实现的常量必须按照之前描述的打包规则与`gl_MaxVertexUniformVectors`相匹配。如果不匹配，顶点着色器就无法编译。开发人员可以应用打包规则，确定存储统一变量、常数变量和字面值所需的统一变量存储总数。然而，确定特定于实现的常量数量是不可能的，因为这个值不仅在不同实现中不同，而且取决于顶点着色器使用的内建着色器语言函数。通常，特定于实现的常量在使用内建超越函数时是必需的。

至于字面值，OpenGL ES 3.0着色语言规范不做任何常量传播。结果是，同一个字面值的多个实例将被多次计算。可以理解的是，在顶点着色器中使用字面值（如0.0或者1.0）更容易，但是我们建议尽可能避免采用这种技术，应该声明相应的常数变量代替字面值。这种方法避免将同一个字面值计算多次，在那种情况下，如果顶点统一变量存储需求超过了实现所能支持的存储量，可能导致顶点着色器无法编译。

# 顶点着色器实例

在iOS中，需要用`GLKViewController`创建动画渲染循环，以指定的帧速率重新绘制视图的内容。

## 矩阵变换

设计一个顶点着色器获取一个位置和其相关的颜色数据作为输入或者属性，用一个4x4矩阵变换位置，并输出变换后的位置和颜色。

``` objc
// glsl.vsh
#version 300 es

uniform mat4 u_mvpMatrix;

layout(location = 0) in vec4 a_position;
layout(location = 1) in vec4 a_color;

out vec4 v_color;

void main() {
    v_color = a_color;
    gl_Position = u_mvpMatrix * a_position;
}
```

变换矩阵：

``` objc
self.elapsedTime += self.timeSinceLastUpdate;
float varyingFactor = sin(self.elapsedTime);
// 缩放矩阵
GLKMatrix4 scaleMatrix = GLKMatrix4MakeScale(varyingFactor, varyingFactor, 1.0);
// 旋转矩阵
GLKMatrix4 rotateMatrix = GLKMatrix4MakeRotation(varyingFactor, 0, 0, 1.0);
// 平移矩阵
GLKMatrix4 translateMatrix = GLKMatrix4MakeTranslation(varyingFactor, 0, 0);
// 矩阵相乘
self.transformMatrix = GLKMatrix4Multiply(translateMatrix, rotateMatrix);
self.transformMatrix = GLKMatrix4Multiply(self.transformMatrix, scaleMatrix);
```

# 待续。。。