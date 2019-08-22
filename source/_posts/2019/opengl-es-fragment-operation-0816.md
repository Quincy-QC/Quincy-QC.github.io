---
title: "OpenGL ES学习--片段操作"
catalog: true
toc_nav_num: true
date: 2019-08-16 17:30:01
subtitle: "About OpenGL ES"
header-img: "/img/article_header/article_header.png"
busuanzi: true
tags:
- iOS

---

> 在OpenGL ES 3.0片段管线中执行片段着色器之后，可能应用到整个帧缓冲区或者单独片段的操作。片段着色器的输出是片段的颜色和深度值。下面的操作在片段着色器执行之后发生，可能影响像素的可见性和最终颜色：
> - 裁剪区域测试
> - 模板缓冲区测试
> - 深度缓冲区测试
> - 多重采样
> - 混合
> - 抖动
>
> 片段在前往帧缓冲区途中经历的测试和操作如下图所示。
> ![着色器后的片段管线](/img/article/20190816/1.png)

# 缓冲区

OpenGL ES支持3中缓冲区，每种缓冲区都保存帧缓冲区中每个像素的不同数据：
- 颜色缓冲区（由前台和后台颜色缓冲区组成）
- 深度缓冲区
- 模板缓冲区

缓冲区的大小常被称作”缓冲区深度“（不要与深度缓冲区混淆），由可用于存储单个像素信息的位数来计量。例如，颜色缓冲区有3个分量，用于存储红、绿和蓝色分量以及可选的Alpha分量存储。颜色缓冲区的深度是所有颜色分量位数的总和。深度和模板缓冲区与此相反，这些缓冲区中用单一值表示像素的位深度。例如，深度缓冲区可能每个像素有16位。缓冲区的总大小是所有分量的位深度的总和。常见的帧缓冲区深度包含16位的RGB缓冲区，红色和蓝色各5位，绿色为6位（人类的视觉系统对绿色比对红色或者蓝色更敏感），对于RGBA缓冲区，32位被平均分配。
此外，颜色缓冲区可能是双重缓冲，也就是包含两个缓冲区：一个在输出设备（如监控器或者LCD显示器）上显示，称作”前台“缓冲区；另一个缓冲区对观看者隐藏，但是用于构造将要显示的下一个图像，称作”后台“缓冲区。在双缓冲区应用程序中，通过在后台缓冲区中绘制然后切换前后台缓冲区显示新图像来实现动画。缓冲区的切换通常与显示设备的刷新周期同步，这样将产生连续、流程动画的假象。
虽然每个EGL配置都有一个颜色缓冲区，深度和模板缓冲区是可选的。不过，每个EGL实现必须提供至少一个包含所有3个缓冲区的配置，深度缓冲区至少有16位的深度，模板缓冲区深度至少为8位。

## 清除缓冲区

OpenGL ES是一个交互式渲染系统，它假定在每个帧的开始，要将缓冲区的所有内容初始化为默认值。缓冲区可以通过调用`glClear`函数清除，该函数用一个位掩码表示应该清除为其指定值的各种缓冲区：

``` objc
/**
 清除缓冲区

 @param mask#> 指定要清除的缓冲区，由如下表示各种OpenGL ES缓冲区的位掩码联合组成：GL_COLOR_BUFFER_BIT， GL_DEPTH_BUFFER_BIT，GL_STENCIL_BUFFER_BIT description#>
 @return void
 */
glClear(GLbitfield mask);
```

我们没有要清除每个缓冲区，也没有必要同时清除它们，但是对每个帧仅调用`glClear`一次并同时清除所有需要的缓冲区，可以得到最好的性能。
当请求清除缓冲区时，每个缓冲区都有一个默认值。对于每个缓冲区，可以用如下函数指定需要的清除值：

``` objc
/**
 清除颜色缓冲区指定需要的清除值

 @param red，green，blue，alpha#> 当传递给glClear的位掩码中包含GL_COLOR_BUFFER_BIT时，指定颜色缓冲区中所有像素的颜色值（处于[0, 1]区间） description#>
 @return void
 */
glClearColor(GLclampf red, GLclampf green, GLclampf blue, GLclampf alpha);

/**
 清除深度缓冲区指定需要的清除值

 @param depth#> 当传递给glClear的位掩码中包含GL_DEPTH_BUFFER_BIT时，指定深度缓冲区中所有像素的深度值（处于[0, 1]区间） description#>
 @return void
 */
glClearDepthf(GLclampf depth);

/**
 清除模板缓冲区指定需要的清除值

 @param s#> 当传递给glClear的位掩码中包含GL_DEPTH_BUFFER_BIT时，指定模板缓冲区中所有像素的模板值（处于[0, 2^n-1]区间，其中n是模板缓冲区中可用的位数） description#>
 @return void
 */
glClearStencil(GLint s);
```

如果在一个帧缓冲区对象中有多个绘图缓冲区，可以用如下调用清除特定的绘图缓冲区：

``` objc
/**
 清除特定的绘图缓冲区

 @param buffer#> 指定要清除的缓冲区类型，可能是GL_COLOR、GL_FRONT、GL_BACK、GL_FRONT_AND_BACK、GL_LEFT、GL_RIGHT、GL_DEPTH（仅glClearBufferfv）或GL_STENCIL（仅glClearBufferiv） description#>
 @param drawbuffer#> 指定要清除的绘图缓冲区名称。对于深度或者模板缓冲区，必须为0.对颜色缓冲区必须小于GL_MAX_DRAW_BUFFERS description#>
 @param value#> 指定用于清除缓冲区的一个四元素向量（对于颜色缓冲区）和一个数值（对于深度或者模板缓冲区）的指针 description#>
 @return void
 */
glClearBufferiv(GLenum buffer, GLint drawbuffer, const GLint *value);
glClearBufferuiv(GLenum buffer, GLint drawbuffer, const GLuint *value);
glClearBufferfv(GLenum buffer, GLint drawbuffer, const GLfloat *value);
```

为了减少函数调用的数量，可以用`glClearBufferfi`同时清除深度和模板缓冲区：

``` objc
/**
 同时清除深度和模板缓冲区

 @param buffer#> 指定要清除的缓冲区类型；必须是GL_DEPTH_STENCIL description#>
 @param drawbuffer#> 指定要清楚的绘图缓冲区名称；必须为0 description#>
 @param depth#> 指定清除深度缓冲区所用的值 description#>
 @param stencil#> 指定清除模板缓冲区所用的值 description#>
 @return void
 */
glClearBufferfi(GLenum buffer, GLint drawbuffer, GLfloat depth, GLint stencil);
```

## 用掩码控制帧缓冲区的写入

我们也可以通过指定一个缓冲区写入掩码来控制哪些缓冲区或者分量（颜色缓冲区的情况下）可以写入。在像素值被写入缓冲区之前，使用缓冲区掩码验证该缓冲区是否可写入。
对于颜色缓冲区，`glColorMask`例程指定像素被写入时颜色缓冲区中的哪些分量会被更新。如果特定分量的掩码被设置为`GL_FALSE`，则该分量在写入时不会被更新。默认情况下，所有颜色分量都可以写入：

``` objc
/**
 指定颜色分量在写入时是否被更新

 @param red，green，blue，alpha#> 指定颜色缓冲区的特定颜色分量在渲染的时候是否可以修改 description#>
 @return void
 */
glColorMask(GLboolean red, GLboolean green, GLboolean blue, GLboolean alpha);
```

同样，深度缓冲区的写入通过以指定深度缓冲区是否可写入的`GL_TRUE`或`GL_FALSE`为参数的调用`glDepthMask`进行控制。
在渲染透明物体的时候，深度缓冲区的写入常常被禁用。开始时，启用深度缓冲区的写入（设置为`GL_TRUE`），渲染场景中的所有不透明物体。这能够确保所有不透明物体有正确的深度，而深度缓冲区包含场景的对应深度信息。然后，在渲染透明物体之前，应该调用`glDepthMask`（`GL_FALSE`）来禁用深度缓冲区的写入。在深度缓冲区的写入被禁用时，数值仍然可以从中读出，并用于深度对比。这使得被不透明物体遮盖的透明物体可以正确地缓冲深度，但是不会修改深度缓冲区，从而使不透明的物体被透明物体遮盖：

``` objc
/**
 指定深度缓冲区是否可以修改

 @param flag#> 指定深度缓冲区是否可以修改 description#>
 @return void
 */
glDepthMask(GLboolean flag);
```

最后，可以调用`glStencilMask`来禁用模板缓冲区的写入。与`glColorMask`或`glDepthMask`不同，我们可以提供一个掩码来指定模板缓冲区的哪些位可以写入：

``` objc
/**
 指定模板缓冲区的哪些位可以写入

 @param mask#> 指定一个说明模板缓冲区中的像素哪些位可以修改的位掩码（在[0, 2^n-1]区间，其中n是模板缓冲区位数），使用得较多的是0x00表示禁止写入，0xFF表示允许任何写入。 description#>
 @return void
 */
glStencilMask(GLuint mask);
```

`glStencilMaskSeparate`例程可以根据图元的面顶点顺序（有时候称作”面部特征“）设置模板掩码，这允许对正面和背面的图元使用不同的模板掩码：

``` objc
/**
 对正面和背面的图元使用不同的模板掩码来指定缓冲区哪些位可以写入

 @param face#> 指定根据渲染图元的面顶点顺序应用的模板掩码。有效值为GL_FRONT，GL_BACK，GL_FRONT_AND_BACK description#>
 @param mask#> 指定一个位掩码（在[0, 2^n-1]区间，其中n是模板缓冲区位数），表示模板缓冲区中像素的哪些位由面指定 description#>
 @return void
 */
glStencilMaskSeparate(GLenum face, GLuint mask);
```

# 片段测试和操作

下面几个小节描述可以应用到OpenGL ES片段的各种测试。默认情况下，所有片段测试和操作都被禁用，片段在写入帧缓冲区时按照接收它们的顺序变成像素。通过启用不同的片段，可以应用操作性测试，以选择哪些片段成为像素并影响最终的图像。
每个片段测试都可以通过调用`glEnable`单独启用，该函数所带的标志参数如下表：

glEnable标志 | 描述
- | -
GL_DEPTH_TEST | 控制片段的深度测试
GL_STENCIL_TEST | 控制片段的模板测试
GL_BLEND | 控制片段与颜色缓冲区中存储的颜色的混合
GL_DITHER | 在写入颜色缓冲区前控制片段颜色的抖动
GL_SAMPLE_COVERAGE | 控制样本范围值的计算
GL_SAMPLE_ALPHA_TO_COVERAGE | 控制样本范围值计算中样本Alpha的使用

## 使用裁剪测试

裁剪测试通过指定一个矩形区域（进一步限制帧缓冲区中可以写入的像素）提供了额外的裁剪层次。使用裁剪矩形是两步的过程。首先，需用`glScissor`函数指定矩形区域：

``` objc
/**
 指定裁剪矩形

 @param x，y#> 以视口坐标指定裁剪矩形左下角 description#>
 @param width#> 指定裁剪矩形宽度 description#>
 @param height#> 指定裁剪矩形高度 description#>
 @return void
 */
glScissor(GLint x, GLint y, GLsizei width, GLsizei height);
```

指定裁剪矩形之后，需通过调用`glEnable(GL_SCISSOR_TEST)`启用它，以实施更多的裁剪。所有渲染（包括视口清除）都限于裁剪矩形之内。
一般来说，裁剪矩形是视口中的一个子区域，但是这两个区域不一定真正交叉。当两个区域不交叉时，裁剪操作将在视口区域外渲染的像素上进行。注意，视口的变换发生在片段着色器之前，而裁剪测试发生在片段着色器阶段之后。

## 模板缓冲区测试

应用到片段的下一个操作是模板测试。模板缓冲区是一个逐像素掩码，保存可用于确定某个像素是否应该被更新的值。模板测试由应用程序启用或者禁用。
模板缓冲区的使用可以看作两步的操作。第一步是逐像素掩码初始化模板缓冲区，这可以通过渲染几何形状并指定模板缓冲区的更新方法来完成。第二步通常是使用这些值控制后续在颜色缓冲区中的渲染。在两种情况下，都指定参数在模板测试中的使用方式。
模板测试实际上是一个位测试，就像在C程序中使用掩码确定某一位是否置位一样。控制模板测试的运算符和值的模板函数由`glStencilFunc`或`glStencilSeparate`函数控制：

``` objc
/**
 控制模板测试的运算符和值的模板函数

 @param func#> 指定模板测试的比较函数，有效值为GL_EQUAL、GL_NOTEQUAL、GL_LESS、GL_GREATER、GL_LEQUEAL、GL_GEQUAL、GL_ALWAYS、GL_NEVER description#>
 @param ref#> 指定模板测试的参考值 description#>
 @param mask#> 指定在与参考值与模板缓冲区中各位比较之前进行按位与运算的掩码 description#>
 @return void
 */
glStencilFunc(GLenum func, GLint ref, GLuint mask);

/**
 控制模板测试的运算符和值的模板函数

 @param face#> 指定与所提供的模板函数相关的面。有效值为GL_FRONT、GL_BACK和GL_FRONT_AND_BACK description#>
 @param func#> 指定模板测试的比较函数，有效值为GL_EQUAL、GL_NOTEQUAL、GL_LESS、GL_GREATER、GL_LEQUEAL、GL_GEQUAL、GL_ALWAYS、GL_NEVER description#>
 @param ref#> 指定模板测试的参考值 description#>
 @param mask#> 指定在与参考值与模板缓冲区中各位比较之前进行按位与运算的掩码 description#>
 @return void
 */
glStencilFuncSeparate(GLenum face, GLenum func, GLint ref, GLuint mask);
```

为了更精细地控制模板测试，可以使用一个掩码参数来选择模板值中的哪些位应该参加测试。在选择这些位之后，它们的值用提供的运算符与参考值比较。例如，要指定模板缓冲区最低三位等于2的模板测试，应该调用：

``` objc
glStencilFunc(GL_EQUAL, 2, 0x7);
```

并启用模板测试。注意，在二进制格式中，0x7的最后三位为111。

配置了模板测试之后，通常还需要让OpenGL ES 3.0知道模板测试通过时对模板缓冲区中的值进行什么操作。实际上，修改模板缓冲区中的值不仅依赖模板测试，还要加入深度测试的结果。结合模板和深度测试，一个片段可能有3种结果：
1. 片段无法通过模板测试。如果这样，则不对该片段进行任何进一步的测试（也就是深度测试）。
2. 片段通过模板测试，但是无法通过深度测试。
3. 片段既通过模板测试，又通过深度测试。

这些可能的结果都可以用于影响该像素位置的模板缓冲区中的值。`glStencilOp`和`glStencilOpSeparte`函数控制每个测试结果对深度缓冲区进行的操作，模板值上的可能操作如下表：

模板函数 | 描述
- | -
GL_ZERO | 将模板值设置为0
GL_REPLACE | 用`glStencilFunc`或`glStencilFuncSeparate`中指定的参考值代替当前模板值
GL_INCE, GL_DECR | 递增或递减模板值；模板值被限定在0或2<sup>n</sup>，其中n为模板缓冲区位数
GL_INCE_WRAP, GL_DECR_WRAP | 递增或者递减模板值，但是如果模板值上溢或者下溢，则”卷绕“该值（最大值递增产生新的模板值0，0值递减产生最大模板值）
GL_KEEP | 保持当前模板值，实际上没有修改该像素的值
GL_INVERT | 模板缓冲区中值的按位非

``` objc
/**
 更新模板缓冲区中的值

 @param fail#> 指定片段不能通过模板测试时应用到模板位的操作。有效值为上表所述 description#>
 @param zfail#> 指定片段通过模板测试但是没有通过深度测试时应用的操作 description#>
 @param zpass#> 指定片段在模板和深度测试中都通过时应用的操作 description#>
 @return void
 */
glStencilOp(GLenum fail, GLenum zfail, GLenum zpass);

/**
 更新模板缓冲区中的值

 @param face#> 指定与提供的模板函数相关的面。有效值为GL_FRONT、GL_BACK和GL_FRONT_AND_BACK description#>
 @param fail#> 指定片段不能通过模板测试时应用到模板位的操作。有效值为上表所述 description#>
 @param zfail#> 指定片段通过模板测试但是没有通过深度测试时应用的操作 description#>
 @param zpass#> 指定片段在模板和深度测试中都通过时应用的操作 description#>
 @return void
 */
glStencilOpSeparate(GLenum face, GLenum fail, GLenum zfail, GLenum zpass);
```

### 深度缓冲测试

深度缓冲区通常用于隐藏表面的消除。传统上，它保存渲染表面上每个像素与视点最近物体的距离值，对于每个新的输入片段，将其与视点的距离和存储值比较。默认情况下，如果输入片段的深度值小于深度缓冲区中保存的值（意味着它离观看者更近），则输入片段的深度值代替保存在深度缓冲区中的值，然后其颜色值代替颜色缓冲区中的颜色值。这是深度缓冲的标准方法--如果这就是我们想做的，那么只需要在创建窗口时请求一个深度缓冲区，然后调用带`GL_DEPTH_TEST`的`glEnable`启用深度测试。如果深度缓冲区与颜色缓冲区关联，则深度测试总是会通过。
当然，这是使用深度缓冲区的唯一手段。可以通过调用`glDepthFunc`修改深度比较运算符：

``` objc
/**
 修改深度比较运算符

 @param func#> 指定深度值比较函数，可能是GL_LESS、GL_GREATER、GL_LEQUAL、GL_GEQUAL、GL_EQUAL、GL_NOTEQUAL、GL_ALWAYS或GL_NEVER中的一个 description#>
 @return void
 */
glDepthFunc(GLenum func);
```

## 混合

一旦片段通过了所有启用的片段测试，它的颜色将与片段像素位置中已经存在的颜色组合。在两个颜色组合之前，它们与一个比例因子相乘，然后用指定的混合运算符组合。混合方程如下：C<sub>final</sub>=f<sub>source</sub>C<sub>source</sub>opf<sub>destination</sub>C<sub>destination</sub>。
其中，f<sub>source</sub>和C<sub>source</sub>分别是输入片段的比例因子和颜色。同样，f<sub>destination</sub>和C<sub>destination</sub>是像素的比例因子和颜色，op是组合折算值的数学运算符。
比例因子通过调用`glBlendFunc`或者`glBlendFuncSeparate`指定：

``` objc
/**
 指定混合比例因子

 @param sfactor#> 指定输入片段的混合系数 description#>
 @param dfactor#> 指定目标像素的混合系数 description#>
 @return void
 */
glBlendFunc(GLenum sfactor, GLenum dfactor);

/**
 指定混合比例因子

 @param srcRGB#> 指定输入片段红、绿和蓝色分量的混合系数 description#>
 @param dstRGB#> 指定目标像素红、绿和蓝色分量的混合系数 description#>
 @param srcAlpha#> 指定输入片段Alpha值的混合系数 description#>
 @param dstAlpha#> 指定目标像素Alpha值的混合系数 description#>
 @return void
 */
glBlendFuncSeparate(GLenum srcRGB, GLenum dstRGB, GLenum srcAlpha, GLenum dstAlpha);
```

混合系数的取值如下表：

![混合函数](/img/article/20190816/2.png)

C<sub>constant</sub>代表调用`glBlendColor`设置的常量颜色：

``` objc
/**
 设置混合常量颜色

 @param red，green，blue，alpha#> 指定常量混合颜色的分量值 description#>
 @return void
 */
glBlendColor(GLfloat red, GLfloat green, GLfloat blue, GLfloat alpha);
```

输入片段和像素颜色乘以各自的比例因子后，它们用由`glBlendEquation`或`glBlendEquationSeparate`指定的运算符组合。默认情况下，混合后的颜色用`GL_FUNC_ADD`运算符累加。`GL_FUNC_SUBTRACT`运算符从输入片段值中减去帧缓冲区中的换算值。同样，`GL_FUNC_REVERSE_SUBTRACT`运算符颠倒混合方程式，从当前像素值中减去输入片段颜色：

```
/**
 设置混合运算符

 @param mode#> 指定混合运算符。有效值是GL_FUNC_ADD、GL_FUNC_SUBTRACT、GL_FUNC_REVERSE_SUBTRACT、GL_MIN、GL_MAX description#>
 @return void
 */
glBlendEquation(GLenum mode);

/**
 设置混合运算符

 @param modeRGB#> 为红、绿和蓝色分量指定混合运算符 description#>
 @param modeAlpha#> 指定Alpha分量混合运算符 description#>
 @return void
 */
glBlendEquationSeparate(GLenum modeRGB, GLenum modeAlpha);
```

# 抖动

在由于帧缓冲区中每个分量的位数导致的帧缓冲区中可用颜色数量有限的系统上，我们可以用抖动（Dithering）模拟更大的色深。抖动算法以某种方式安排颜色，使图像看上去似乎比实际上的可用颜色更多。OpenGL ES 3.0没有规定抖动阶段使用的算法；具体的技术很大程度上依赖于实现。
应用程序对抖动的唯一控制是它是否应用到最终的像素上。这一决策完全通过带`GL_DITHER`的`glEnable`或`glDisable`控制，它指定了管线中抖动的使用。在初始状态下，启用抖动。

# 多重采样抗锯齿

抗锯齿（Anti-aliasing）是通过尝试减少不同像素渲染中产生的视觉伪像来改进生成图像质量的一种重要技术。OpenGL ES 3.0渲染的几何形状图元在一个网格上进行光栅化，它们的边缘可能在这一过程中变形。绘制跨越显示器的对角线时，几何肯定会发现阶梯效应。
为了理解什么是多重采样（Multisampling），以及它是如何解决锯齿问题的，我们有必要更加深入地了解OpenGL光栅器的工作方式。
光栅器是位于最终处理过的顶点之后到片段着色器之前所经过的所有的算法与过程的总和。光栅器会将一个图元的所有顶点作为输入，并将它转换为一系列的片段。顶点坐标理论上可以取任何值，但片段不行，因为它们受限于窗口的分辨率。顶点坐标与片段之间几乎永远也不会有一对一的映射，所以光栅器必须以某种方式来决定每个顶点最终所在的片段/屏幕坐标。

![多重采样三角形](/img/article/20190816/3.png)

这里我们可以看到一个屏幕像素的网格，每个像素的中心包含有一个采样点（Sampler Point），它会被用来决定这个三角形是否遮盖了某个像素。途中红色的采样点被三角形所覆盖，在每一个遮住的像素处都会生成一个片段。虽然三角形边缘的一部分也遮住了某些屏幕像素，但是这些像素的采样点并没有被三角形内部所覆盖，所以它们不会受到片段着色器影响。

现在我们可能已经清楚走样的原因了。完整渲染后的三角形在屏幕上会是这样的：

![多重采样渲染三角形](/img/article/20190816/4.png)

由于屏幕像素总量的限制，有些边缘的像素能够被渲染出来，而有些则不会。结果就是我们使用了不光滑的边缘来渲染图元，导致锯齿边缘。

多重采样所做的正是将单一的采样点变为多个采样点（这也是它名称的由来）。我们不再使用像素中心的是单一采样点，取而代之的是以特定图案排列的4个子采样点（Subsampler）。我们将用这些子采样点来决定像素的遮盖度。当然，这也意味着颜色缓冲的大小会随着子采样点的增加而增加。

![多重采样三角形剖析](/img/article/20190816/5.png)

上图的左侧展示了正常情况下判定三角形是否遮盖的方式。在例子中的这个像素上不会运行片段着色器（所以它会保持空白）。因为它的采样点并未被三角形所覆盖。上图的右侧展示的是实施多重采样之后的版本，每个像素包含有4个采样点。这里，只有两个采样点遮盖了三角形。

> 采样点的数量可以是任意的，更多的采样点能带来更精确的遮盖率。

从这里开始多重采样就变得有趣起来了。我们知道三角形只遮盖了2个子采样点，所以下一步是决定这个像素的颜色。MSAA（Multi-Sampling Anti-Aliasing）真正的工作方式是，无论三角形遮盖了多少个子采样点，（每个图元中）每个像素只允许一次片段着色器。片段着色器所使用的顶点数据会插值到每个像素的中心，所得到的结果颜色会被存储在每个被遮盖住的子采样点中。当颜色缓冲的子样本被图元的所有颜色填满时，所有的这些颜色将会在每个像素内部平均化。因为上图的4个采样点只有2个被遮盖住了，这个像素的颜色将会是三角形颜色与其他两个采样点的颜色（在这里是无色）的平均值，最终形成一种淡蓝色。

这样子做之后，颜色缓冲中所有的图元边缘将会产生一种更平滑的图形：

![多重采样MSAA三角形](/img/article/20190816/6.png)

这里，每个像素包含4个子采样点（不相关的采样点都没有标注），蓝色的采样点被三角形所遮盖，而灰色的则没有。对于三角形的内部的像素，片段着色器只会运行一次，颜色输出会被存储到全部的4个子样本中。而在三角形的边缘，并不是所有的子采样点都被覆盖，所有片段着色器的结果将只会存储到部分的子样本中。根据被遮盖的子样本的数量，最终的像素颜色将由三角形的颜色与其它子样本中所存储的颜色来决定。

简单来说，一个像素中如果有更多的采样点被三角形覆盖，那么这个像素的颜色就会更接近于三角形的颜色。如果我们给上面的三角形填充颜色，就能得到以下效果：

![多重采样渲染三角形](/img/article/20190816/7.png)

对于每个像素来说，越少的子采样点被三角形覆盖，那么它受到三角形的影响越小。三角形的不平滑边缘被稍浅的颜色所包围后，从远处观察时就会显得更加平滑。

不仅仅是颜色值会受到多重采样的影响，深度和模板测试也能够使用多个采样点。对深度测试来说，每个顶点的深度值会在运行深度测试之前被插值到各个子样本中。对模板测试来说，我们对每个子样本，而不是每个像素，存储一个模板值。当然，这也意味着深度和模板缓冲的大小会乘以子采样点的个数。

我们到目前为止讨论的都是多重采样抗锯齿的背后原理，光栅器背后的实际逻辑比目前讨论的要复杂，但现在我们应该已经可以理解多重采样抗锯齿的大体概念和逻辑了。

# 在帧缓冲区读取和写入像素

如果我们想为后代留下渲染过的图像，可以从颜色缓冲区中读回像素值，但是不能从深度或者模板缓冲区中读取。当调用`glReadPixels`时，颜色缓冲区中的像素将从一个前面分配的数组中返回应用程序：

``` objc
/**
 从颜色缓冲区中读回像素值

 @param x，y#> 指定从颜色缓冲区中要读取的像素矩形左下角的视口坐标 description#>
 @param width，height#> 指定从颜色缓冲区中读取的像素矩形的尺寸 description#>
 @param format#> 指定想要返回的像素格式。有3种可用格式：GL_RGBA、GL_RGBA_INTEGER以及查询GL_IMPLEMENTATION_COLOR_READ_FORMAT返回的值（特定于实现的像素格式） description#>
 @param type#> 指定返回的像素数据类型。有5中可用类型：GL_UNSIGNED_BYTE、GL_UNSIGNED_INT、GL_INT、GL_FLOAT以及查询GL_IMPLEMENTATION_COLOR_READ_TYPE返回的值（特定于实现的像素格式） description#>
 @param pixels#> 一个连续的字节数组，在glReadPixels返回之后包含从颜色缓冲区读取的值 description#>
 @return void
 */
glReadPixels(GLint x, GLint y, GLsizei width, GLsizei height, GLenum format, GLenum type, GLvoid *pixels);
```

## 像素打包缓冲区对象

当用`glBindBuffer`将一个非零的缓冲区对象绑定到`GL_PIXEL_PACK_BUFFER`时，`glReadPixels`命令将立即返回，并且启动DMA传输，从帧缓冲区读取像素，并将数据写入像素缓冲区对象（PBO）。
为了保持CPU忙碌，可以在`glReadPixels`调用之后计划一些CPU处理，是CPU计算和DMA传输重叠。根据应用程序的不同，数据可能立即可用；在这种情况下，可以使用多个PBO解决方案，在CPU从一个PBO传输的数据时，可以处理之前从另一个PBO传输的数据。

# 多重渲染目标

多重渲染目标（MRT）允许应用程序一次渲染到多个颜色缓冲区。利用多重渲染目标，片段着色器输出多个颜色（可以用于保存RGBA颜色、发现、深度或者纹理坐标），每个颜色用于一个连接的颜色缓冲区。MRT用于多种高级渲染算法中，例如延迟着色和快速环境遮蔽计算。

下面的步骤说明了设置MRT的方法：

1. 用`glGenFramebuffers`和`glBindFramebuffer`命令初始化帧缓冲区对象（FBO）：

``` objc
glGenFramebuffers(1, &fbo);
glBindFramebuffer(GL_FRAMEBUFFER, fbo);
```

2. 用`glGenTextures`和`glBindTexture`命令初始化纹理：

``` objc
glBindTexture(GL_TEXTURE_2D, textureId);
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, textureWidth, textureHeight, 0, GL_RGBA, GL_UNSIGNED_BYTE, NULL);
// Set the filtering mode
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
```

3. 用`glFramebufferTexture2D`或`glFramebufferTextureLayer`命令将相关纹理绑定到FBO：

``` objc
glFramebufferTexture2D(GL_DRAW_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, textureId, 0);
```

4. 用`glDrawBuffers`命令为渲染指定颜色附着：

``` objc
/**
 为渲染指定颜色附着

 @param n#> 指定bufs中的缓冲区数量 description#>
 @param bufs#> 指定一个符号常量数组，这些常量指定了片段颜色或者数据值将要写入的缓冲区 description#>
 @return void
 */
glDrawBuffers(GLsizei n, const GLenum *bufs);

const GLenum attachments[4] = { GL_COLOR_ATTACHMENT0, GL_COLOR_ATTACHMENT1, GL_COLOR_ATTACHMENT2, GL_COLOR_ATTACHMENT3 };
glDrawBuffers(4, attachments);
```

可以调用以符号常量`GL_MAX_COLOR_ATTACHMENTS`为参数的`glGetIntegerv`查询颜色附着的最大数量。所有OpenGL 3.0实现都支持的颜色附着的最小数量为4.

5. 在片段着色器中声明和使用多个着色器输出。例如，如下的声明将把片段着色器输出`fragData0 ~ fragData3`分别复制到绘图缓冲区0 ~ 3：

``` objc
layout(location = 0) out vec4 fragData0;
layout(location = 1) out vec4 fragData1;
layout(location = 2) out vec4 fragData2;
layout(location = 3) out vec4 fragData3;
```

# 总结

这篇文章介绍了有关片段着色器之后发生的测试和操作的内容。这是OpenGL ES 3.0管线中的最后阶段。