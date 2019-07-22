---
title: "OpenGL ES学习--图元装配和光栅化"
catalog: true
toc_nav_num: true
date: 2019-08-01 19:09:32
subtitle: "About OpenGL ES"
header-img: "/img/article_header/article_header.png"
busuanzi: true
tags:
- iOS

---

> 本章描述OpenGL ES支持的图元和几何形状对象的类型，并说明绘制它们的方法。然后描述发生在顶点着色器处理图元顶点之后的图元装配阶段。在这一阶段，执行裁剪、透视分割和视口变换操作，对这些操作将作详细的讨论。本章以光栅化阶段的描述作为结束。光栅化是将图元转换为一组二维片段的过程，这些片段由片段着色器处理，代表可以在屏幕上绘制的像素。

# 图元

图元是可以用OpenGL ES中的`glDrawArrays`、`glDrawElements`、`glDrawRangeElements`、`glDrawArraysInstanced`、`glDrawElementsInstanced`命令绘制的几何形状对象。图元由一组表示顶点位置的顶点描述。其他如颜色、纹理坐标和几何法线等信息也作为通用属性与每个顶点关联。

OpenGL ES 3.0可以绘制如下图元：
- 三角形
- 直线
- 点精灵

## 三角形

三角形代表着描述由3D应用程序渲染的几何形状对象时最常用的方法。OpenGL ES支持的三角形图元有`GL_TRIANGLES`、`GL_TRIANGLE_STRIP`和`GL_TRIANGLE_FAN`。

![三角形图元类型](/img/article/20190801/1.png)

`GL_TRIANGLES`绘制一系列单独的三角形。如上图所示，绘制了顶点为(v<sup>0</sup>, v<sup>1</sup>, v<sup>2</sup>)和(v<sup>3</sup>, v<sup>4</sup>, v<sup>5</sup>)的两个三角形。总共绘制了n/3个三角形，其中n为`glDraw*`函数中的`count`指定的索引。

`GL_TRIANGLE_STRIP`绘制一系列相互连接的三角形。如上图所示，绘制了4个顶点为(v<sup>0</sup>, v<sup>1</sup>, v<sup>2</sup>)、(v<sup>2</sup>, v<sup>1</sup>, v<sup>3</sup>)、(v<sup>2</sup>, v<sup>3</sup>, v<sup>4</sup>)和(v<sup>4</sup>, v<sup>3</sup>, v<sup>5</sup>)的三角形（注意顺序）。总共绘制了n-2个三角形，其中n为`glDraw*`函数中的`count`指定的索引。

`GL_TRIANGLE_FAN`也绘制一系列相连的三角形。如上图所示，绘制了3个顶点为(v<sup>0</sup>, v<sup>1</sup>, v<sup>2</sup>)、(v<sup>0</sup>, v<sup>2</sup>, v<sup>3</sup>)和(v<sup>0</sup>, v<sup>3</sup>, v<sup>4</sup>)的三角形，总共绘制了n-2个三角形，其中n为`glDraw*`函数中的`count`指定的索引。

## 直线

OpenGL ES支持的直线图元有`GL_LINES`、`GL_LINE_STRIP`和`GL_LINE_LOOP`。

![直线图元类型](/img/article/20190801/2.png)

`GL_LINES`绘制一系列不相连的线段。如上图所示，绘制了端点为(v<sup>0</sup>, v<sup>1</sup>)、(v<sup>2</sup>, v<sup>3</sup>)和(v<sup>4</sup>, v<sup>5</sup>)的单独线段。总共绘制了n/2条线段，其中n为`glDraw*`函数中的`count`指定的索引。

`GL_LINE_STRIP`绘制一系列相连的线段。如上图所示，绘制了3条端点为(v<sup>0</sup>, v<sup>1</sup>)、(v<sup>1</sup>, v<sup>2</sup>)和(v<sup>2</sup>, v<sup>3</sup>)的线段。总共绘制了n-1条线段，其中n为`glDraw*`函数中的`count`指定的索引。

除了最后一条线段从v<sup>n-1</sup>到v<sup>0</sup>之外，`GL_LINE_LOOP`和`GL_LINE_STRIP`的绘制方法类似。如上图所示，绘制了端点为(v<sup>0</sup>, v<sup>1</sup>)、(v<sup>1</sup>, v<sup>2</sup>)、(v<sup>2</sup>, v<sup>3</sup>)、(v<sup>3</sup>, v<sup>4</sup>)和(v<sup>4</sup>, v<sup>0</sup>)的线段。总共绘制了n条线段，其中n为`glDraw*`函数中的`count`指定的索引。

线段的宽度用`glLineWidth`API调用指定：

``` objc
/**
 设置线宽

 @param width#> 指定线宽，以像素数表示；默认宽度为1.0 description#>
 @return void
 */
glLineWidth(GLfloat width);
```

`glLineWidth`指定的宽度将受限于OpenGL ES 3.0实现所支持的线宽范围。此外，指定的宽度将被OpenGL记住，直到应用程序更新。支持的线宽范围可以用如下的命令查询，对于大于1
的线宽，没有强制支持。

``` objc
GLfloat lineWidthRange[2];
glGetFloatv(GL_ALIASED_LINE_WIDTH_RANGE, lineWidthRange);
```

## 点精灵

OpenGL ES支持的点精灵图元是`GL_POINTS`。点精灵对指定的每个顶点绘制。点精灵通常用于将粒子效果当做点而非正方形绘制，从而实现高效渲染。点精灵是指定位置和半径的屏幕对齐的正方形，位置描述正方形的中心，半径用于计算描述点精灵的正方形的4个坐标。

`gl_PointSize`是可用于在顶点着色器中输出点半径（或者点尺寸）的内建变量。与点图元相关的顶点着色器输出`gl_PointSize`很重要，否则，点尺寸值被视为未定义，很可能会造成绘图错误。顶点着色器输出的`gl_PointSize`受到OpenGL ES 3.0实现所支持的非平滑点尺寸范围的限制。这个范围可以用如下命令查询：
``` objc
GLfloat pointSizeRange[2];
glGetFloatv(GL_ALIASED_POINT_SIZE_RANGE, pointSizeRange);
```

默认情况下，OpenGL ES 3.0将窗口原点(0, 0)描述为(左, 下)区域，但是，对于点精灵，点坐标的原点是(左, 上)。

`gl_PointCoord`是只能在渲染图元为点精灵时用于片段着色器内部的内建变量，它用`mediump`精度限定符声明一个vec2变量。随着我们从左侧移到右侧，从顶部移到底部，赋予`gl_PointCoord`的值从0~1变化。

# 绘制图元

OpenGL ES中有5个绘制图元的API调用：`glDrawArrays`、`glDrawElements`、`glDrawRangeElements`、`glDrawArraysInstanced`和`glDrawElementsInstanced`。

``` objc
/**
 绘制图元

 @param mode#> 指定要渲染的图元，有效值为GL_POINTS, GL_LINES, GL_LINES_STRIP, GL_LINES_LOOP, GL_TRIANGLES, GL_TRIANGLES_STRIP, GL_TRIANGLE_FAN description#>
 @param first#> 指定启用的顶点数组中的起始顶点索引 description#>
 @param count#> 指定要绘制的顶点数量 description#>
 @return void
 */
glDrawArrays(GLenum mode, GLint first, GLsizei count);

/**
 绘制图元

 @param mode#> 指定要渲染的图元，有效值为GL_POINTS, GL_LINES, GL_LINES_STRIP, GL_LINES_LOOP, GL_TRIANGLES, GL_TRIANGLES_STRIP, GL_TRIANGLE_FAN description#>
 @param count#> 指定要绘制的索引数量 description#>
 @param type#> 指定indices中保存的元素索引类型，有效值为GL_UNSIGNED_BYTE, GL_UNSIGNED_SHORT, GL_UNSIGNED_INT description#>
 @param indices#> 指向元素索引存储位置的指针 description#>
 @return void
 */
glDrawElements(GLenum mode, GLsizei count, GLenum type, const GLvoid *indices)

/**
 绘制图元

 @param mode#> 指定要渲染的图元，有效值为GL_POINTS, GL_LINES, GL_LINES_STRIP, GL_LINES_LOOP, GL_TRIANGLES, GL_TRIANGLES_STRIP, GL_TRIANGLE_FAN description#>
 @param start#> 指定indices中的最小数组索引 description#>
 @param end#> 指定indices中的最大数组索引 description#> 
 @param count#> 指定要绘制的索引数量 description#>
 @param type#> 指定indices中保存的元素索引类型，有效值为GL_UNSIGNED_BYTE, GL_UNSIGNED_SHORT, GL_UNSIGNED_INT description#>
 @param indices#> 指向元素索引存储位置的指针 description#>
 @return void
 */
glDrawRangeElements(GLenum mode, GLuint start, GLuint end, GLsizei count, GLenum type, const GLvoid *indices)
```

如果有一个一系列顺序元素索引描述的图元，且几何形状的顶点不共享，则`glDrawArrays`很好用，但是，游戏或者其他3D应用程序使用的典型对象由多个三角形网格组成，其中的元素索引可能不一定按照顺序，顶点通常在网格的三角形直接共享。

比如考虑上图立方体的绘制，如果我们用`glDrawArrays`绘制，则代码如下：

``` objc
GLfloat vertices[] = { .. };
glEnableVertexAttribArray(0);
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 0, vertices);
glDrawArrays(GL_TRIANGLES, 0, 36);
```

为了用`glDrawArrays`绘制这个立方体，需要为立方体的每一个面调用`glDrawArrays`。共享的顶点必须重复，这意味着需要分配24个顶点（如果将每面当做`GL_TRIANGLE_FAN`绘制）或者36个顶点（如果使用`GL_TRIANGLES`），而不是8个顶点。这不是一个高效的做法。

用`glDrawElements`绘制同一个立方体的代码如下：

``` objc
GLfloat vertices[] = { .. };
GLubyte indices[36] = {
    0, 1, 2, 0, 2, 3,
    0, 3, 4, 0, 4, 5,
    7, 1, 6, 7, 2, 1,
    7, 5, 4, 7, 6, 5,
    7, 3, 2, 7, 4, 3
}
glEnableVertexAttribArray(0);
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 0, vertices);
glDrawElements(GL_TRIANGLES, 36, GL_USINGED_BYTE, indices);
```

即使我们用`glDrawElements`绘制三角形，用`glDrawArrays`和`glDrawElements`绘制一个三角扇形，我们的应用程序在GPU上运行的也比`glDrawArrays`更快，这有很多原因。比如，由于顶点重用，顶点属性数据的尺寸将小于`glDrawElements`。这也导致较小的内存占用和内存贷款需求。

## 图元重启

使用图元重启，可以在一次绘图调用中渲染多个不相连的图元（例如三角扇形或者条带）。这对于降低绘图API的调用的开销是有利的。图元重启的另一种方法是生成退化三角形（需要一些注意事项），这种方法较不简洁。

使用图元重启，可以通过在索引列表中插入一个特殊索引来重启一个用于索引绘图调用（如`glDrawElements`、`glDrawElementsInstances`或`glDrawRangeElements`）的图元。这个特殊索引是该索引类型的最大索引（例如，索引类型为`GL_UNSIGNED_BYTE`或`GL_UNSIGNED_SHORT`时，分别为255或者65535）。

例如，假定两个三角形条带分别有元素索引(0, 1, 2, 3)和(8, 9, 10, 11)。如果我们想利用图元重启在一次调用`glDrawElements*`中绘制两个条带，索引类型为GL_UNSIGNED_SHORT，则组合的元素索引列表为(0, 1, 2, 3, 255, 8, 9, 10, 11)。

可以用如下代码启用和禁用图元启用：
``` objc
glEnable(GL_PRIMITIVE_RESTART_FIXED_INDEX);
// draw primitives
glDisable(GL_PRIMITIVE_RESTART_FIXED_INDEX);
```

代码示例：

``` objc
// 配置
GLfloat vertices[] = {
    -0.75f, 0, 0,
    -0.75f, 0.75f, 0,
    -0.25f, 0, 0,
    -0.25f, 0.75, 0,
    
    0.75f, 0, 0,
    0.75f, 0.75f, 0,
    0.25f, 0, 0,
    0.25f, 0.75, 0
};
GLfloat colors[] = {
    1.0f, 0.0f, 0.0f, 1.0f,
    0.0f, 1.0f, 0.0f, 1.0f,
    0.0f, 0.0f, 1.0f, 1.0f,
    0.0f, 1.0f, 0.0f, 1.0f,
    
    1.0f, 0.0f, 0.0f, 1.0f,
    0.0f, 1.0f, 0.0f, 1.0f,
    0.0f, 0.0f, 1.0f, 1.0f,
    0.0f, 1.0f, 0.0f, 1.0f
};
GLushort indices[] = {0, 1, 2, 3, 65535, 4, 5, 6, 7};

GLuint vbos[3];
glGenBuffers(3, vbos);
glBindBuffer(GL_ARRAY_BUFFER, vbos[0]);
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);

glBindBuffer(GL_ARRAY_BUFFER, vbos[1]);
glBufferData(GL_ARRAY_BUFFER, sizeof(colors), colors, GL_STATIC_DRAW);

glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, vbos[2]);
glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(indices), indices, GL_STATIC_DRAW);

glGenVertexArrays(1, &vaos);
glBindVertexArray(vaos);

glBindBuffer(GL_ARRAY_BUFFER, vbos[0]);
glEnableVertexAttribArray(0);
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(GLfloat), 0);
glBindBuffer(GL_ARRAY_BUFFER, vbos[1]);
glEnableVertexAttribArray(1);
glVertexAttribPointer(1, 4, GL_FLOAT, GL_FALSE, 4 * sizeof(GLfloat), 0);
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, vbos[2]);

glBindVertexArray(0);

// 绘制
glEnable(GL_PRIMITIVE_RESTART_FIXED_INDEX);

glBindVertexArray(vaos);
glDrawElements(GL_TRIANGLE_STRIP, 9, GL_UNSIGNED_SHORT, 0);

glDisable(GL_PRIMITIVE_RESTART_FIXED_INDEX);
```

## 驱动顶点

如果没有限定符，那么顶点着色器的输出值在图元中使用线性插值。但是，使用平面着色时没有发生插值。因为没有发生插值，所以片段着色器中只有一个顶点值可用。对于给定的图元实例，这个驱动顶点确定使用顶点着色器的哪一个顶点输出，因为只能使用一个顶点。[插值限定符](/article/2019/opengl-es-glsl-0715/#插值限定符)。

图元i的类型 | 驱动顶点
- | -
GL_POINT | i
GL_LINES | 2i
GL_LINE_LOOP | 如果i < n，为i + 1；如果i = n，为1
GL_LINE_STRIP | i + 1
GL_TRIANGLES | 3i
GL_TRIANGLE_STRIP | i + 2
GL_TRIANGLE_FAN | i + 2

> 第i个图元实例的驱动顶点选择，顶点的编号从1到n，n是绘制的顶点数量。

## 几何形状实例化

几何形状实例化很高效，可以用一次API调用多次渲染具有不同属性（例如不同的变换矩阵、颜色或者大小）的一个对象。这一功能在渲染大量类似对象时很有用，例如对人群的渲染。几何图形实例化降低了向OpenGL ES引擎发送许多API调用的CPU处理开销。要使用实例化绘图调用渲染，可以使用如下命令：

``` objc
/**
 实例化绘图

 @param mode#> 指定要渲染的图元，有效值为GL_POINTS, GL_LINES, GL_LINES_STRIP, GL_LINES_LOOP, GL_TRIANGLES, GL_TRIANGLES_STRIP, GL_TRIANGLE_FAN description#>
 @param first#> 指定要启用的顶点数组中的起始顶点索引 description#>
 @param count#> 指定绘制的索引数量 description#>
 @param instancecount#> 指定绘制的图元实例数量 description#>
 @return void
 */
glDrawArraysInstanced(GLenum mode, GLint first, GLsizei count, GLsizei instancecount);

/**
 实例化绘图

 @param mode#> 指定要渲染的图元，有效值为GL_POINTS, GL_LINES, GL_LINES_STRIP, GL_LINES_LOOP, GL_TRIANGLES, GL_TRIANGLES_STRIP, GL_TRIANGLE_FAN description#>
 @param count#> 指定绘制的索引数量 description#>
 @param type#> 指定保存在indices中的元素索引类型，有效值为GL_UNSIGNED_BYTE, GL_UNSIGNED_SHORT, GL_UNSIGNED_INT description#>
 @param indices#> 指定元素索引存储位置的一个指针 description#>
 @param instancecount#> 指定绘制的图元实例数量 description#>
 @return void
 */
glDrawElementsInstanced(GLenum mode, GLsizei count, GLenum type, const GLvoid *indices, GLsizei instancecount);
```

可以使用两种方法访问每个实例的数据。第一种方法是用如下命令指示OpenGL ES对每个实例读取一次或者多次顶点属性：

``` objc
/**
 对每个实例读取一次或多次顶点属性

 @param index#> 指定通用顶点属性索引 description#>
 @param divisor#> 指定index位置的通用属性更新之间传递传递的实例数量 description#>
 @return void
 */
glVertexAttribDivisor(GLuint index, GLuint divisor);
```

默认情况下，如果没有指定`glVertexAttribDivisor`或者每个顶点属性的`divisor`等于零，对每个顶点将读取一次顶点属性。如果`divisor`等于1，则每个图元实例读取一次顶点属性。[代码例子](https://github.com/danginsburg/opengles3-book/blob/master/Chapter_7/Instancing/Instancing.c)

第二种方法是使用内建输入变量`glInstanceID`作为顶点着色器中的缓冲区索引，以访问每个实例的数据（仅限OpenGL ES 3.0）。使用前面提到的几何形状实例化API调用时，`gl_InstanceID`将保存当前图元实例的索引。使用非实例化绘图调用时，`gl_InstanceID`将返回0。

## 性能提示

应用程序应该确保用尽可能大的图元尺寸调用`glDrawElements`和`glDrawElementsInstanced`。如果我们绘制`GL_TRIANGLES`，这很容易做到，但是，如果有三角形条带或者扇形的网格，则可以用图元重启将这些网格连接在一起，而不用对每个三角形条带网格单独调用`glDrawElements*`。

如果无法使用图元重启机制将网格连接到一起（为了维护和旧版本OpenGL ES的兼容性），可以添加造成退化三角形的元素索引，代价是使用更多的索引。退化三角形是两个或者更多顶点相同的三角形。GPU可以非常简单的检测和拒绝退化三角形，所以这是很好的性能改进，我们可以讲一个很大的图元放入右GPU渲染的队列。

代码示例：

``` objc
// 配置
GLfloat vertices[] = {
    -0.75f, 0, 0,
    -0.75f, 0.75f, 0,
    -0.25f, 0, 0,
    -0.25f, 0.75, 0,
    
    0.75f, 0, 0,
    0.75f, 0.75f, 0,
    0.25f, 0, 0,
    0.25f, 0.75, 0
};
GLfloat colors[] = {
    1.0f, 0.0f, 0.0f, 1.0f,
    0.0f, 1.0f, 0.0f, 1.0f,
    0.0f, 0.0f, 1.0f, 1.0f,
    0.0f, 1.0f, 0.0f, 1.0f,
    
    1.0f, 0.0f, 0.0f, 1.0f,
    0.0f, 1.0f, 0.0f, 1.0f,
    0.0f, 0.0f, 1.0f, 1.0f,
    0.0f, 1.0f, 0.0f, 1.0f
};
GLushort indices[] = {0, 1, 2, 3, 3, 4, 4, 5, 6, 7};

GLuint vbos[3];
glGenBuffers(3, vbos);
glBindBuffer(GL_ARRAY_BUFFER, vbos[0]);
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);

glBindBuffer(GL_ARRAY_BUFFER, vbos[1]);
glBufferData(GL_ARRAY_BUFFER, sizeof(colors), colors, GL_STATIC_DRAW);

glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, vbos[2]);
glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(indices), indices, GL_STATIC_DRAW);

glGenVertexArrays(1, &vaos);
glBindVertexArray(vaos);

glBindBuffer(GL_ARRAY_BUFFER, vbos[0]);
glEnableVertexAttribArray(0);
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(GLfloat), 0);
glBindBuffer(GL_ARRAY_BUFFER, vbos[1]);
glEnableVertexAttribArray(1);
glVertexAttribPointer(1, 4, GL_FLOAT, GL_FALSE, 4 * sizeof(GLfloat), 0);
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, vbos[2]);

glBindVertexArray(0);

// 绘制
glBindVertexArray(vaos);
glDrawElements(GL_TRIANGLE_STRIP, 10, GL_UNSIGNED_SHORT, 0);
```

# 图元装配

下图展示了图元装配阶段。通过`glDraw***`提供的顶点由顶点着色器执行，顶点着色器变换的每个顶点包括描述顶点(x, y, z, w)值的顶点位置。图元位置和顶点索引确定将被渲染的单独图元。对于每个单独图元（三角形、直线和点）及其对应的顶点，图元装配阶段执行图中所示的操作。

![OpenGL ES图元装配阶段](/img/article/20190801/3.png)

## 坐标系统

下图展示了顶点通过顶点着色器和图元装配阶段时的坐标系统。顶点以物体或者本地坐标空间输入到OpenGL ES，这是最可能用来建模和存储一个对象的坐标空间。在顶点着色器执行之后，顶点位置被认为是在裁剪坐标空间内。顶点位置从本地坐标系统（也就是物体坐标）到裁剪坐标的变换通过加载执行这一转换的对应矩阵来完成，这些矩阵保存在顶点着色器中定义的统一变量中。

![坐标系统](/img/article/20190801/4.png)

### 裁剪

为了避免在可视景体之外处理图元，图元被裁剪到裁剪空间。执行顶点着色器之后的顶点位置处于裁剪坐标空间内。裁剪坐标是由(x<sub>c</sub>, y<sub>c</sub>, z<sub>c</sub>, w<sub>c</sub>)指定的同类坐标。在裁剪空间(x<sub>c</sub>, y<sub>c</sub>, z<sub>c</sub>, w<sub>c</sub>)中定义的顶点坐标根据视景体（又称裁剪体）裁剪。

裁剪体由6个裁剪平面定义，这些平面称作近、远、左、右、上、下裁剪平面。在裁剪坐标中，裁剪体如下：-w<sub>c</sub> <= x<sub>c</sub> <= w<sub>c</sub>，-w<sub>c</sub> <= y<sub>c</sub> <= w<sub>c</sub>，-w<sub>c</sub> <= z<sub>c</sub> <= w<sub>c</sub>。

![视景体](/img/article/20190801/5.png)

裁剪阶段将把每个图元裁剪为上图所示的裁剪体。我们在这里说的“图元”，是指用`GL_TRIANGLES`绘制的单独三角形列表中的每一个三角形，或者一个三角形条带或者扇形中的一个三角形，或者用`GL_LINES`绘制的单独直线列表中的一条直线，或者一个直线条带或者闭合折线中的一条直线，或者点精灵列表中的一个特定点。对于每种图元类型，执行如下操作：
- 裁剪三角形--如果三角形完全在视景体内部，则不执行任何裁剪。如果三角形完全在视景体之外，则该三角形被放弃。如果三角形部分在视景体内，则根据相应的平面裁剪三角形。裁剪操作将生成新的顶点，这些顶点被裁剪到安排为三角形扇形的平面。
- 裁剪直线--如果直线完全在视景体内部，则不执行任何裁剪。如果直线完全在视景体之外，则该直线被放弃。如果直线部分在视景体内，则直线被裁剪并生成相应的新顶点。
- 裁剪点精灵--如果点位置在近或者远裁剪平面之外，或者如果表示点精灵的正方形在裁剪体之外，裁剪阶段将抛弃点精灵。否则，它将不做变化地通过该阶段，点精灵将在其裁剪体内部移到外部时裁剪，反之亦然。
在图元根据六个裁剪平面进行裁剪时，顶点坐标家里透视分割，从而成为规范化的设备坐标。规范化的设备坐标范围为-1.0到1.0。

## 透视分割

透视分割取得裁剪坐标(X<sub>c</sub>, Y<sub>c</sub>, Z<sub>c</sub>, W<sub>c</sub>)指定的点，并将其投影到屏幕或者视口上。这个投影通过将(X<sub>c</sub>, Y<sub>c</sub>, Z<sub>c</sub>)除以W<sub>c</sub>进行。执行(X<sub>c</sub>/W<sub>c</sub>)、(Y<sub>c</sub>/W<sub>c</sub>)和(Z<sub>c</sub>/W<sub>c</sub>)之后，我们得到规范化的设备坐标(X<sub>d</sub>, Y<sub>d</sub>, Z<sub>d</sub>)。这些坐标被称为规范化设备坐标，因为他们落在[-1.0...1.0]区间。这些规范化的(X<sub>d</sub>, Y<sub>d</sub>)坐标根据视口的大小将被转换为真正的屏幕（或者窗口）坐标。规范化的(Z<sub>d</sub>)坐标将用`glDepthRangef`指定的`near`和`far`深度值转换为屏幕的Z值，这些转换在视口变换阶段进行。

## 视口变换

视口是一个二维矩阵窗口区域，是所有OpenGL ES渲染操作最终显示的地方。视口变换可用如下API调用设置：

``` objc
/**
 视口窗口

 @param x#> 指定视口左下角的窗口坐标x，以像素数表示 description#>
 @param y#> 指定视口左下角的窗口坐标y，以像素数表示 description#>
 @param width#> 指定视口的宽度（以像素数表示）；必须大于0 description#>
 @param height#> 指定视口的高度（以像素数表示）；必须大于0 description#>
 @return void
 */
glViewport(GLint x, GLint y, GLsizei width, GLsizei height);
```

从规范化设备坐标(x<sub>d</sub>, y<sub>d</sub>, z<sub>d</sub>)到窗口坐标(x<sub>w</sub>, y<sub>w</sub>, z<sub>w</sub>)的转换用如下变换给出：

![矩阵变换](/img/article/20190801/6.png)

在这个变换中，o<sub>x</sub>=x+w/2，o<sub>y</sub>=y+h/2，n和f代表所需的深度范围。
深度范围值n和f可以用如下API调用设置：

```objc
/**
 设置深度值

 @param zNear#> 指定所需的深度范围，默认值为0.0，限于(0.0, 1.0)区间内 description#>
 @param zFar#> 指定所需的深度范围，默认值为1.0，限于(0.0, 1.0)区间内 description#>
 @return void
 */
glDepthRangef(GLclampf zNear, GLclampf zFar);
```

`glDepthRangef`和`glViewport`指定的值用于将顶点位置从规范化设备坐标转换为窗口（屏幕）坐标。

# 光栅化

下图展示了光栅化管线。在顶点变换和图元裁剪之后，光栅化管线取得单独图元（如三角形、线段或者点精灵），并为该图元生成对应的片段。每个片段由屏幕空间中的整数位置(x, y)标识。片段代表了屏幕空间中(x, y)指定的像素位置和由片段着色器处理而生成片段颜色的附加片段数据。

![OpenGL ES光栅化阶段](/img/article/20190801/7.png)

## 剔除

在三角形被光栅化之前，我们需要确定它们是正面（也就是面向观看者）或者背面（也就是背向观看者）。剔除（culling）操作抛弃背向观看者的三角形。要确定三角形是正面还是背面，首先需要知道它的方向。

三角形的方向指定从第一个顶点开始，经过第二个和第三个顶点，最后回到第一个顶点的弯曲方向或者路径顺序。下图展示了弯曲顺序为顺时针和逆时针的两个三角形实例。

![顺时针和逆时针的三角形](/img/article/20190801/8.png)

三角形的方向通过以窗口坐标表示的有符号三角形的面积来计算。我们现在需要将计算出来的三角形面积符号翻译为顺时针（CW）或者逆时针（CCW）方向。这种从三角形面积的符号到顺时针或者逆时针方向的映射由应用程序用如下API调用指定：

```objc
/**
 指定正面三角形的方向

 @param mode#> 指定正面三角形的方向，有效值为GL_CW或者GL_CCW，默认值为GL_CCW description#>
 @return void
 */
glFrontFace(GLenum mode);
```

要确定需要提出的三角形，需要知道三角形将被剔除的面。通过如下API调用：

``` objc
/**
 剔除三角形

 @param mode#> 指定要被剔除的三角形的面，有效值为GL_FROUNT、GL_BACK、GL_FROUNT_AND_BACK，默认值为GL_BACK description#>
 @return void
 */
glCullFace(GLenum mode);
```

最后一个要点是，需要知道剔除操作是否应该执行。如果`GL_CULL_FACE`状态启用，剔除操作将被执行。通过如下API调用启用或者禁用：

``` objc
/**
 启用剔除

 @param cap#> 设置为GL_CULL_FACE，默认情况下剔除被禁用 description#>
 @return void
 */
glEnable(GLenum cap);

/**
 禁用剔除

 @param cap#> 设置为GL_CULL_FACE，默认情况下剔除被禁用 description#>
 @return void
 */
glDisable(GLenum cap);
```

概括起来，要剔除合适的三角形，OpenGL ES应用程序首先必须用`glEnable(GL_CULL_FACE)`启用剔除，用`glCullFace`设置相应的剔除面，并用`glFrontFace`设置正面三角形的方向。

> 剔除应该始终启用，以避免GPU浪费时间去光栅化不可见的三角形。启用剔除应该能够改善OpenGL ES应用程序的整体性能。

## 多边形偏移

考虑绘制两个相互重叠的多边形的情况。很可能会有伪像，这些伪像被称为深度冲突伪像，是因为三角形光栅化的精度有限而发生的，这种精度限制可能影响逐片段生成的深度值的精度，造成伪像。三角形光栅化使用的参数和生成的逐片段深度值的有限精度将越来越好，但是这个问题永远无法完全解决。

为了避免看到伪像，我们需要在执行深度测试和深度值写入深度缓冲区之前，在计算出来的深度值上添加一个偏移量。如果深度测试通过，原始的深度值--而不是原始深度值+偏移--将被保存到深度缓冲区中。

多边形偏移：

``` objc
/**
 多边形偏移

 @param factor#> 因数 description#>
 @param units#> 单位数 description#>
 @return void
 */
glPolygonOffset(GLfloat factor, GLfloat units);
```

深度偏移的计算如下：深度偏移 = m * 因数 + r * 单位数。m是三角形的最大深度斜率，斜率项在三角形光栅化阶段期间由OpenGL ES实现计算；r是一个OpenGL ES实现定义的常量，代表深度值中可以保证产生差异的最小值。
多边形偏移可以分别用`glEnable(GL_POLYGON_OFFSET_FILL)`和`glDisable(GL_POLYGON_OFFSET_FILL)`启用或者禁用。

例子：
``` objc
const float polygonOffsetFactor = -1.0f;
const float polygonOffsetUnits = -2.0f;

glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);

// load vertex shader
// set the appropriate transformation matrices
// set the vertex attribute state

// draw the SMALLER quad
glDrawArrays(GL_TRIANGLE_FAN, 0, 4);

// set the depth func to <= as polygons are coplanar
glDrawFunc(GL_LEQUAL);

glEnable(GL_POLYGON_OFFSET_FILL);

glPolygonOffset(polygonOffsetFactor, polygonOffsetUnits);

// set the vertex attribute state

// draw the LARGER quad
glDrawArrays(GL_TRIANGLE_FAN, 0, 4);
```

# 遮挡查询

遮挡查询用查询对象来跟踪通过深度测试的任何片段或者样本。这种方法可用于不同的技术，例如镜头炫光特效的可见性测试以及避免在包围体被遮挡的不可见对象上进行几何形状处理的优化。

遮挡查询的开始和结束：

``` objc
/**
 遮挡查询开始

 @param target#> 指定查询对象的目标类型；有效值是GL_ANY_SAMPLES_PASSED，GL_ANY_SAMPLES_PASSED_CONSERVATIVE，GL_TRANSFORM_FEEDBACK_PRIMITIVES_WRITTEN description#>
 @param id#> 指定查询对象的名称 description#>
 @return void
 */
glBeginQuery(GLenum target, GLuint id);
    
    
/**
 遮挡查询结束

 @param target#> 指定查询对象的目标类型；有效值是GL_ANY_SAMPLES_PASSED，GL_ANY_SAMPLES_PASSED_CONSERVATIVE，GL_TRANSFORM_FEEDBACK_PRIMITIVES_WRITTEN description#>
 @return void
 */
glEndQuery(GLenum target);
```

使用`GL_ANY_SAMPLES_PASSED`目标奖返回表示是否有样本通过深度测试的精确布尔状态；`GL_ANY_SAMPLES_PASSED_CONSERVATIVE`目标将提供更好的性能，但是答案的精确度较低；使用`GL_ANY_SAMPLES_PASSED_CONSERVATIVE`，有些实现将在没有样本通过深度测试时返回`GL_TRUE`。

`id`用`glGenQueries`创建，用`glDeleteQueries`删除：

``` objc
/**
 创建查询id

 @param n#> 指定生成的查询名称对象的数量 description#>
 @param ids#> 指定一个数组，以存储查询名称对象的列表 description#>
 @return void
 */
glGenQueries(GLsizei n, GLuint *ids);
    
    
/**
 删除查询id

 @param n#> 指定要删除的查询名称对象的数量 description#>
 @param ids#> 指定一个需要删除的查询名称对象的列表数组 description#>
 @return void
 */ 
glDeleteQueries(GLsizei n, const GLuint *ids);
```

在用`glBeginQuery`和`glEndQuery`指定查询对象边界之后，可以使用`glGetQueryObjectuiv`检索查询对象的结果：

``` objc
/**
 检索查询对象的结果

 @param id#> 指定查询对象的名称 description#>
 @param pname#> 指定需要检索的查询对象参数，可以为GL_QUERY_RESULT或GL_QUERY_RESULT_AVAILABLE description#>
 @param params#> 指定存储返回参数值得对象类型的数组 description#>
 @return void
 */
glGetQueryObjectuiv(GLuint id, GLenum pname, GLuint *params);
```

# 总结

这篇文章主要学习了OpenGL ES支持的图元类型，并且了解了如何用常规的非实例化和实例化绘图调用高效地绘制它们。还讨论了在顶点上执行坐标变换的方法。还学习了关于光栅化阶段的知识，在这个阶段，图元被转换为代表屏幕上绘制的像素的偏度。