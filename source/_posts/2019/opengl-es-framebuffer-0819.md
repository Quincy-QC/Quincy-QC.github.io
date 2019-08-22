---
title: "OpenGL ES学习--帧缓冲区对象"
catalog: true
toc_nav_num: true
date: 2019-08-19 20:03:12
subtitle: "About OpenGL ES"
header-img: "/img/article_header/article_header.png"
busuanzi: true
tags:
- iOS

---

> 在本篇中我们将描述帧缓冲区对象的概念、应用程序创建它们的方法以及应用程序使用它们渲染到屏幕外缓冲区或者纹理的方法。

# 为什么使用帧缓冲区对象

在应用程序调用任何OpenGL ES命令之前，需要首先创建一个渲染上下文和绘图表面，并使之成为现行上下文和表面。渲染上下文和绘图表面通常由原生窗口系统通过EGL等API提供。渲染上下文包含正确操作所需的对应状态。由原生窗口系统提供的绘图表面可以是一个在屏幕上显示的表面（称为窗口系统提供的帧缓冲区），也可以是屏幕外表面（称作pbuffer）。创建EGL绘图表面的调用让我们以像素数的形式指定表面的宽度和高度、表面是否使用颜色、深度和模板缓冲区以及这些缓冲区的位深。
默认情况下，OpenGL ES使用窗口系统提供的帧缓冲区作为绘图表面。如果应用程序只在屏幕上的表面绘图，则窗口系统提供的帧缓冲区通常很高效。但是，许多应用程序需要渲染到纹理，为此，使用窗口系统提供的帧缓冲区作为绘图表面通常不是理想的选择。
应用程序可以使用以下两种技术之一渲染到纹理：
- 通过绘制到窗口系统提供的帧缓冲区，然后将帧缓冲区的对应区域复制到纹理来实现渲染到纹理。这可以用`glCopyTexImage2D`和`glCopyTexSubImage2D`API实现。顾名思义，这些API执行从帧缓冲区到纹理缓冲区的复制，这一复制操作往往对性能有不利影响。此外，这种方法只有在纹理的尺寸小于或者等于帧缓冲区尺寸的时候才有效。
- 通过使用连接到纹理的pbuffer来实现渲染到纹理。我们知道，窗口系统提供的表面必须连接到一个渲染上下文。这在某些对每个pbuffer和窗口表面需要不同上下文的实现中可能效率低下。此外，在窗口系统提供的可绘制表面之间切换有时候需要OpenGL ES实现清除所有切换之前渲染的图像。这可能在渲染管线中造成代价很高的“起泡效应”（CPU闲置）。在这种系统上，我们建议避免使用pbuffer渲染到纹理，因为与上下文和窗口系统提供的可绘制表面之间的切换相关的开销很大。

上述两种方法对于渲染到纹理或者其他屏幕外表面来说都不理想。作为替代，我们需要允许应用程序直接渲染到纹理的API，或者在OpenGL ES API中具备创建屏幕外表面的能力，并将它作为渲染目标。帧缓冲区对象和渲染缓冲区对象允许应用程序完成这些操作，不需要额外创建渲染上下文。结果是，我们在使用窗口系统提供的可绘制表面时，不再需要担心上下文和可绘制表面切换的开销。因此，帧缓冲区对象提供了渲染到纹理或者屏幕外表面的更好、更有效的方法。
帧缓冲区对象API支持如下操作：
- 仅适用OpenGL ES命令创建帧缓冲区对象
- 在单一EGL上下文中创建和使用多个缓冲区对象--也就是说，不需要每个帧缓冲区都有一个渲染上下文
- 创建屏幕外颜色、深度或者模板渲染缓冲区和纹理，并将它们连接到帧缓冲区对象
- 在多个帧缓冲区之间共享颜色、深度或者模板缓冲区
- 将纹理直接连接到帧缓冲区作为颜色或者深度，从而避免了进行复制操作的必要
- 在帧缓冲区之间复制并使帧缓冲区内容失效

# 帧缓冲区和渲染缓冲区对象

渲染缓冲区对象是一个由应用程序分配的2D图像缓冲区。渲染缓冲区可以用于分配和存储颜色、深度或者模板值，可以用作帧缓冲区对象中的颜色、深度或者模板附着。渲染缓冲区类似于屏幕外的窗口系统的可绘制表面--如pbuffer。但是，渲染缓冲区不能直接用作GL纹理。
帧缓冲区对象（FBO）是一组颜色、深度和模板纹理或者渲染目标。各种2D图像可以连接到帧缓冲区对象中的颜色附着点。这些附着点包括一个渲染缓冲区对象，它保存颜色值、2D纹理或者立方图面的mip级别、2D数组纹理的层次甚至3D纹理中一个2D切片的mip级别。同样，包含深度值的各种2D图像可以连接到FBO的深度附着点。这些附着点包括渲染缓冲区、2D纹理的mip级别或者保存深度值的一个立方图面。可以连接到FBO模板附着点的唯一2D图像是保存模板值的渲染缓冲区对象。
下图展示了帧缓冲区对象、渲染缓冲区对象和纹理之间的关系。注意，一个帧缓冲区对象中只能有一个颜色、深度和模板附着。

![帧缓冲区对象、渲染缓冲区对象和纹理](/img/article/20190819/1.png)

## 选择渲染缓冲区与纹理作为帧缓冲区附着的对比

对于渲染到纹理的用例，我们应该将一个纹理对象连接到帧缓冲区对象。这方面的例子包括渲染到一个用作颜色纹理的颜色缓冲区以及渲染到用作阴影的深度纹理的深度缓冲区。
使用渲染缓冲区代替纹理有以下几种原因：
- 渲染缓冲区支持多重采样。
- 如果图像没有被当做纹理使用，则使用渲染缓冲区可能带来性能上的好处。出现这种好处是因为OpenGL ES实现可能以更为高效的格式存储渲染缓冲区，比起纹理来说更适合于渲染。但是，如果预先知道图像不被用作纹理，那么OpenGL ES实现所能做的也仅仅如此。

## 帧缓冲区对象与EGL表面的对比

FBO和窗口系统提供的可绘制表面之间的区别：
- 像素归属测试确定帧缓冲区中(x<sub>w</sub>), y<sub>w</sub>)位置的像素目前是否归OpenGL ES所有，这个测试允许窗口系统控制帧缓冲区中的哪些像素属于OpenGL ES上下文--例如，当OpenGL ES渲染的窗口被遮蔽时。对于应用程序创建的帧缓冲区对象，像素归属测试始终成功，因为帧缓冲区对象拥有所有像素。
- 窗口系统可能只支持双缓冲表面。相反，帧缓冲区对象只支持单缓冲区附着。
- 使用帧缓冲区对象可以实现帧缓冲区之间模板和深度缓冲区的共享，而窗口系统提供的帧缓冲区通常不能实现这一功能。在窗口系统提供的可绘制表面中，模板和深度缓冲区及其对应的状态通常是隐含分配的，因此，无法在可绘制表面之间共享。对于应用程序创建的帧缓冲区对象，模板和深度缓冲区可以独立创建，然后在必要时通过将这些缓冲区连接到多个缓冲区对象中的对应连接点，实现与帧缓冲区对象的关联。

# 创建帧缓冲区和渲染缓冲区对象

创建帧缓冲区和渲染缓冲区对象类似于在OpenGL ES 3.0中创建纹理或者顶点缓冲区对象。
`glGenRenderbuffers`API调用用于分配渲染缓冲区对象名称：

``` objc
/**
 创建渲染缓冲区对象

 @param n#> 返回的渲染缓冲区对象名称的数量 description#>
 @param renderbuffers#> 返回一个有n个元素的数组的指针，分配的渲染缓冲区对象名称将在该数组中返回 description#>
 @return void
 */
glGenRenderbuffers(GLsizei n, GLuint *renderbuffers);
```

`glGenRenderbuffers`分配n个渲染缓冲区对象名称，并在`renderbuffers`中返回。由`glGenRenderbuffers`返回的渲染缓冲区对象名称是不为0的无符号整数。这些名称被标记为`在用`但是没有任何关联的状态。数值0由OpenGL ES保留，不能用于指代一个渲染缓冲区对象。试图修改或者查询帧缓冲区对象0的缓冲区对象状态的应用程序将产生一个对应的错误。
`glGenFramebuffers`API调用用于分配帧缓冲区对象名称：

``` objc
/**
 创建帧缓冲区对象

 @param n#> 返回的帧缓冲区对象名称的数量 description#>
 @param framebuffers#> 指向一个包含n个元素的数组的指针，分配的帧缓冲区对象在该数组中返回 description#>
 @return void
 */
glGenFramebuffers(GLsizei n, GLuint *framebuffers);
```

`glGenFramebuffers`分配n个帧缓冲区对象名称，并在`framebuffers`中返回它们。由`glGenFramebuffers`返回的帧缓冲区对象名称是不为0的无符号整数。这些名称被标记为“在用”但是没有任何关联的状态。数值0由OpenGL ES保留，不能用于指代一个窗口系统提供的帧缓冲区。视图修改或者查询帧缓冲区对象0的缓冲区对象状态的应用程序将产生一个对应的错误。

# 使用渲染缓冲区对象

要为特定的渲染缓冲区对象指定图像的数据存储、格式和尺寸，需要使该对象成为当前渲染缓冲区对象。`glBindRenderbuffer`命令用于设置当前渲染缓冲区对象：

``` objc
/**
 绑定渲染缓冲区对象

 @param target#> 必须设置为GL_RENDERBUFFER description#>
 @param renderbuffer#> 渲染缓冲区对象名称 description#>
 @return void
 */
glBindRenderbuffer(GLenum target, GLuint renderbuffer);
```

注意，`glGenRenderbuffers`不一定要在用`glBindRenderbuffer`绑定之前分配渲染缓冲区对象名称。虽然调用`glGenRenderbuffers`是一个好的做法，但许多应用程序还是为其缓冲区指定编译时常量。应用程序可以将未使用的渲染缓冲区名称指定到`glBindRenderbuffer`。但是，我们建议OpenGL ES应用程序调用`glGenRenderbuffers`，并使用`glGenRenderbuffers`返回的渲染缓冲区对象名称，而不是指定自己的缓冲区对象名称。
在第一次通过调用`glBindRenderbuffer`绑定渲染缓冲区对象名称时，渲染缓冲区对象被分配相应的默认状态。如果分配成功，分配的对象将成为新绑定的渲染缓冲区对象。下面是与渲染缓冲区对象相关的状态和默认值：
- 以像素数表示的宽度和高度--默认值为0。
- 内部格式--描述了渲染缓冲区中存储的像素格式，必须是颜色、深度或者模板可渲染格式。
- 颜色位深--这只有在内部格式是颜色可渲染格式时有效。默认值为0。
- 深度位深--这只有在内部格式是深度可渲染格式时有效。默认值为0。
- 模板位深--这只有在内部格式是模板可渲染格式时有效。默认值为0。

`glBindRenderbuffer`也可以用于绑定到现有的渲染缓冲区对象（也就是之前已经分配使用因而有关联有效状态的缓冲区对象）。绑定命令对于新绑定的渲染缓冲区对象的状态不做任何改变。
一旦绑定渲染缓冲区对象，就可以指定保存在渲染缓冲区中的图像大小和格式。`glRenderbufferStorage`命令可用于这个目的。
除了不提供图像数据，`glRenderbufferStorage`看起来与`glTexImage2D`很相似。我们也可以使用`glRenderbufferStorageMultisample`命令创建一个多重采样渲染缓冲区。`glRenderbufferStorage`等价于样本数设置为0的`glRenderStorageMultisample`。渲染缓冲区的宽度和高度以像素数指定，其值必须小于OpenGL ES实现所支持的最大渲染缓冲区尺寸。所有OpenGL ES实现都支持的最小尺寸为1，实际支持的最大尺寸可以用如下代码查询：

``` objc
GLint maxRenderbufferSize = 0;
glGetIntegerv(GL_MAX_RENDERBUFFER_SIZE, &maxRenderbufferSize);
```

``` objc
/**
 指定渲染缓冲区的图像大小和格式

 @param target#> 必须设置为GL_RENDERBUFFER description#>
 @param internalformat#> 必须为可用于颜色缓冲区、深度缓冲区或者模板缓冲区的格式 description#>
 @param width#> 以像素数表示的渲染缓冲区宽度；必须小于或者等于GL_MAX_RENDERBUFFER_SIZE description#>
 @param height#> 以像素数表示的渲染缓冲区高度；必须小于或者等于GL_MAX_RENDERBUFFER_SIZE description#>
 @return void
 */
glRenderbufferStorage(GLenum target, GLenum internalformat, GLsizei width, GLsizei height);

/**
 创建一个多重采样渲染缓冲区

 @param target#> 必须设置为GL_RENDERBUFFER description#>
 @param samples#> 用于渲染缓冲区对象存储的样本数。必须小于GL_MAX_SAMPLES description#>
 @param internalformat#> 必须为可用于颜色缓冲区、深度缓冲区或者模板缓冲区的格式 description#>
 @param width#> 以像素数表示的渲染缓冲区宽度；必须小于或者等于GL_MAX_RENDERBUFFER_SIZE description#>
 @param height#> 以像素数表示的渲染缓冲区高度；必须小于或者等于GL_MAX_RENDERBUFFER_SIZE description#>
 @return void
 */
glRenderbufferStorageMultisample(GLenum target, GLsizei samples, GLenum internalformat, GLsizei width, GLsizei height);
```

渲染缓冲区对象可以连接到帧缓冲区对象的颜色、深度或者模板附着，而不需要指定渲染缓冲区存储格式和大小。渲染缓冲区的存储格式和大小可以在渲染缓冲区对象连接到帧缓冲区对象前后指定。但是，这些信息必须在帧缓冲区对象和渲染缓冲区附着用于渲染之前指定。

## 多重采样渲染缓冲区

多重采样渲染缓冲区使应用程序可以用多重采样抗锯齿技术渲染到屏幕外帧缓冲区。多重采样渲染缓冲区不能直接绑定到纹理，但是可以用新推出的[帧缓冲区位块传送](#帧缓冲区位块传送)解析为单采样纹理。

# 使用帧缓冲区对象

我们将说明如何使用帧缓冲区对象渲染到一个屏幕外缓冲区（也就是渲染缓冲区）或者渲染到一个纹理。在使用帧缓冲区对象并指定其附着之前，需要使其成为当前帧缓冲区对象。`glBindFramebuffer`命令用于设置当前帧缓冲区对象：

``` objc
/**
 设置当前帧缓冲区对象

 @param target#> 必须设置为GL_READ_FRAMEBUFFER、GL_DRAW_FRAMEBUFFER或GL_FRAMEBUFFER description#>
 @param framebuffer#> 帧缓冲区对象名称 description#>
 @return void
 */
glBindFramebuffer(GLenum target, GLuint framebuffer);
```

注意，对于在用`glBindFramebuffer`绑定帧缓冲区对象之前指定其名称来说，`glGenFramebuffers`并不是必需的。应用程序可以将未用的帧缓冲区对象名称指定给`glBindFramebuffer`。但是，我们建议OpenGL ES应用程序调用`glGenFramebuffers`，并使用`glGenFramebuffers`返回的帧缓冲区对象名称，而不是指定自己的缓冲区对象名称。
在某些OpenGL ES 3.0实现上，当调用`glBindFramebuffer`首次绑定帧缓冲区对象名称时，为帧缓冲区对象分配对应的默认状态。如果分配成功，分配的对象将作为渲染上下文的当前缓冲区对象进行绑定。
与帧缓冲区对象相关的状态如下：
- 颜色附着点--颜色缓冲区的附着点
- 深度附着点--深度缓冲区的附着点
- 模板附着点--模板缓冲区的附着点
- 帧缓冲区完整性状态--帧缓冲器是否处于完整状态，是否可以用于渲染
对于每个附着点，指定如下信息：
- 对象类型--指定与附着点相关的对象的类型。如果连接一个渲染缓冲区对象，则类型可以是`GL_RENDERBUFFER`；如果连接一个纹理对象，那么类型可以是`GL_TEXTURE`。默认值为`GL_NONE`。
- 对象名称--指定连接的对象的名称，可以是渲染缓冲区对象名称或者纹理对象名称。默认值为0。
- 纹理级别--如果连接一个纹理对象，则指定了与附着点线管的纹理的mip级别。默认值为0。
- 纹理立方图面--如果连接一个纹理对象，切纹理为立方图，则指定了6个立方图面中哪一个用于该附着点。默认值为`GL_TEXTURE_CUBE_MAP_POSITIVE_X`。
- 纹理层次--指定3D纹理中用于该附着点的2D切片。默认值为0。
`glBindFramebuffer`也可以用于绑定到现有的帧缓冲区对象（也就是之前已经分配并使用因而有相关的有效状态的对象）。新绑定的帧缓冲区对象的状态没有任何变化。
一旦绑定了帧缓冲区对象，当前绑定的帧缓冲区对象的颜色、深度和模板附着就可以设置为一个渲染缓冲区对象或者一个纹理。如下图所示，颜色附着可以设置为存储颜色值的渲染缓冲区、2D纹理的一个mip级别或者一个立方图面、2D数组纹理的一个层次或者3D纹理中一个2D切片的mip级别。深度附着可以设置为存储深度值或者经过打包的深度和模板值的渲染缓冲区、2D深度纹理的一个mip级别或者深度立方图面。模板附着必须设置为存储模板值或者打包的深度和模板值的渲染缓冲区。

## 连接渲染缓冲区作为帧缓冲区附着

`glFramebufferRenderbuffer`命令用于将一个渲染缓冲区对象连接到帧缓冲区附着点：

``` objc
/**
 将一个渲染缓冲区对象连接到帧缓冲区附着点

 @param target#> 必须设置为GL_READ_FRAMEBUFFER、GL_DRAW_FRAMEBUFFER或GL_FRAMEBUFFER description#>
 @param attachment#> 必须为如下枚举值之一：GL_COLOR_ATTACHMENTi、GL_DEPTH_ATTACHMENT、GL_STENCIL_ATTACHMENT、GL_DEPTH_STENCIL_ATTACHMENT description#>
 @param renderbuffertarget#> 必须设置为GL_RENDERBUFFER description#>
 @param renderbuffer#> 应该用作附着的渲染缓冲区对象；必须为0或者现有渲染缓冲区对象名称 description#>
 @return void
 */
glFramebufferRenderbuffer(GLenum target, GLenum attachment, GLenum renderbuffertarget, GLuint renderbuffer);
```

如果调用`glFramebufferRenderbuffer`时`renderbuffer`不为0，这个渲染缓冲区对象将被用作`attachment`参数值指定的新颜色、深度或者模板附着点。
附着点的状态将被修改为：
- 对象类型 = GL_RENDERBUFFER
- 对象名称 = renderbuffer
- 纹理级别和纹理层次 = 0
- 纹理立方图面 = GL_NONE

新连接的渲染缓冲区对象状态或者缓冲区内容不做任何修改。
如果`glFramebufferRenderbuffer`调用中`renderbuffer`等于0，则`attachment`指定的颜色、深度或者模板缓冲区将被断开并重置为0。

## 连接一个2D纹理作为帧缓冲区附着

`glFramebufferTexture2D`命令用于将一个2D纹理的某个mip级别或者立方图面连接到帧缓冲区附着点。它可以用来将纹理作为颜色、缓冲区或者模板附着点连接：

``` objc
/**
 将一个2D纹理的某个mip级别或者立方图面连接到帧缓冲区附着点

 @param target#> 必须设置为GL_READ_FRAMEBUFFER、GL_DRAW_FRAMEBUFFER、GL_FRAMEBUFFER description#>
 @param attachment#> 必须为如下枚举值之一：GL_COLOR_ATTACHMENTi、GL_DEPTH_ATTACHMENT、GL_STENCIL_ATTACHMENT、GL_DEPTH_STENCIL_ATTACHMENT description#>
 @param textarget#> 指定纹理目标；这是glTexImage2D中的target参数指定的值 description#>
 @param texture#> 指定纹理对象 description#>
 @param level#> 指定纹理图像的mip级别 description#>
 @return void
 */
glFramebufferTexture2D(GLenum target, GLenum attachment, GLenum textarget, GLuint texture, GLint level);
```

如果`glFramebufferTexture2D`调用中`texture`不为0，则颜色、深度或者模板附着将被设置为`texture`。如果`glFramebufferTexture2D`发生错误，帧缓冲区的状态将不做修改。
附着点状态将被修改为：
- 对象类型 = GL_TEXTURE
- 对象名称 = texture
- 纹理级别 = level
- 纹理立方图面在纹理附着为立方图时有效，是如下值之一： GL_TEXTURE_CUBE_MAP_POSITIVE_(X | Y | Z)，GL_TEXTURE_CUBE_MAP_NEGATIVE_(X | Y | Z)
- 纹理层次 = 0
`glFramebufferTexture2D`不修改新连接的纹理对象状态或者图像内容。注意，纹理对象的状态和图像可以连接到帧缓冲区对象之后修改。
如果`glFramebufferTexture2D`调用中`texture`等于0，则颜色、深度或者模板附着将被断开并重置为0。

## 连接3D纹理的一个图像作为帧缓冲区附着

`glFramebufferTextureLayer`命令用于将3D纹理的一个2D切片或者某个mip级别或者2D数组纹理的一个级别连接到帧缓冲区附着点。

``` objc
/**
 将3D纹理的一个2D切片或者某个mip级别或者2D数组纹理的一个级别连接到帧缓冲区附着点

 @param target#> 必须设置为GL_READ_FRAMEBUFFER、GL_DRAW_FRAMEBUFFER、GL_FRAMEBUFFER description#>
 @param attachment#> 必须为如下枚举值之一：GL_COLOR_ATTACHMENTi、GL_DEPTH_ATTACHMENT、GL_STENCIL_ATTACHMENT、GL_DEPTH_STENCIL_ATTACHMENT description#>
 @param texture#> 指定纹理对象 description#>
 @param level#> 指定纹理图像的mip级别 description#>
 @param layer#> 指定纹理图像层次。如果texture是GL_TEXTURE_CUBE_MAP_3D，则level必须大于或等于0并且小于或等于GL_MAX3D_TEXTURE_SIZE值以2位底的对数。如果texture是GL_TEXTURE_2D_ARRAY，则level必须大于或等于0且不大于GL_MAX_TEXTURE_SIZE值以2为底的对数 description#>
 @return void
 */
glFramebufferTextureLayer(GLenum target, GLenum attachment, GLuint texture, GLint level, GLint layer);
```

`glFramebufferTextureLayer`不修改新连接的纹理对象状态或者其图像的内容。注意，纹理对象的状态和图像可以在连接到帧缓冲器对象之后修改。
附着点的状态被修改为：
- 对象类型 = GL_TEXTURE
- 对象名称 = texture
- 纹理级别 = level
- 纹理立方图面 = GL_NONE
- 纹理层次 = 0
如果`glFramebufferTextureLayer`调用中`texture`等于0，则附着将被断开并重置为0。

## 检查帧缓冲区完整性

帧缓冲区对象必须定义为完整的才能够用作渲染目标。如果当前绑定的帧缓冲区对象不完整，绘制图元或者读取像素的OpenGL ES命令将会失败，并产生表示帧缓冲区不完整原因的对应错误。
帧缓冲区对象被视为完整的规则如下：
- 确保颜色、深度和模板附着有效。帧缓冲区至少有一个有效的附着，如果没有任何附着，帧缓冲区就是不完整的，因为没有可以绘制或者读取的区域。
- 与帧缓冲区对象相关的有效附着必须由相同的宽度和高度。
- 如果存在深度和模板附着，则它们必须是相同的图像。
- 所有渲染缓冲区附着的`GL_RENDERBUFFER_SAMPLES`值都相同。如果附着是渲染缓冲区和纹理的组合，则`GL_RENDERBUFFER_SAMPLES`的值为0。

`glCheckFramebufferStatus`命令可用于验证帧缓冲区对象是否完整：

``` objc
/**
 验证帧缓冲区对象是否完整

 @param target#> 必须设置为GL_READ_FRAMEBUFFER、GL_DRAW_FRMAEBUFFER或GL_FRAMEBUFFER description#>
 @return void
 */
glCheckFramebufferStatus(GLenum target);
```

如果`target`不等于`GL_FRAMEBUFFER`，则`glCheckFramebufferStatus`返回0。如果`target`等于`GL_FRAMEBUFFER`，则返回如下枚举值之一：
- GL_FRAMEBUFFER_COMPLETE -- 帧缓冲区完整
- GL_FRAMEBUFFER_UNDEFINED -- 如果`target`是默认的帧缓冲区但是它不存在
- GL_FRAMEBUFFER_INCOMPLETE_ATTACHMENT -- 帧缓冲区附着点不完整。这可能是因为需要的附着为0，或者不是有效纹理或渲染缓冲区对象
- GL_FRAMEBUFFER_INCOMPLETE_MISSING_ATTACHMENT -- 帧缓冲区中没有有效的附着
- GL_FRAMEBUFFER_UNSUPPORTED -- 帧缓冲区中的附着使用的内部格式组合造成不可渲染的目标
- GL_FRAMEBUFFER_INCOMPLETE_MULTISAMPLE -- 所有渲染缓冲区附着的GL_RENDERBUFFER_SAMPLES不都相同，或者GL_RENDERBUFFER_SAMPLES在附着是渲染缓冲区和纹理的组合时不为0

如果当前绑定的帧缓冲区对象不完整，则试图使用该对象读取和写入像素将会失败。接着，绘制图元的调用（如`glDrawArrays`和`glDrawElements`）和读取帧缓冲区的命令（如`glReadPixels`、`glCopyTeximage2D`、`glCopyTexSubimage2D`和`glCopyTexSubimage3D`）将生成一个`GL_INVALID_FRAMEBUFFER_OPERATION`错误。

# 帧缓冲区位块传送

帧缓冲区位块传送（Blit）可以高效地将一个矩形区域的像素值从一个帧缓冲区（读帧缓冲区）复制到另一个帧缓冲区（绘图帧缓冲区）。帧缓冲区位块传送的关键应用之一是将一个多重采样渲染缓冲区解析为一个纹理（用一个帧缓冲区对象，纹理绑定为它的颜色附着）。
可以用如下命令执行上述操作：

``` objc
/**
 将一个矩形区域的像素从一个帧缓冲区复制到另一个帧缓冲区

 @param srcX0，srcY0，srcX1，srcY1#> 指定读缓冲区内的来源矩形的边界 description#>
 @param dstX0，dstY0，dstX1，dstY1#> 指定写缓冲区内的目标矩形的边界 description#>
 @param mask#> 指定表示哪些缓冲区被复制的标识的按位或；组成如下：GL_COLOR_BUFFER_BIT，GL_DEPTH_BUFFER_BIT，GL_STENCIL_BUFFER_BIT，GL_DEPTH_STENCIL_ATTACHMENT description#>
 @param filter#> 指定图像被拉伸时应用的插值；必须为GL_NEAREST或GL_LINEAR description#>
 @return void
 */
glBlitFramebuffer(GLint srcX0, GLint srcY0, GLint srcX1, GLint srcY1, GLint dstX0, GLint dstY0, GLint dstX1, GLint dstY1, GLbitfield mask, GLenum filter);
```

举例：

``` objc
// set the default framebuffer for writing
glBindFramebuffer(GL_DRAW_FRAMEBUFFER, defaultFramebuffer);
// set the fbo with color attachments for reading
glBindFramebuffer(GL_READ_FRAMEBUFFER, fbo);
// copy the output buffer to target
glReadBuffer(GL_COLOR_ATTACHMENT0);
glBlitFramebuffer(0, 0, width, height, 0, 0, width/2, height, GL_COLOR_BUFFER_BIT, GL_LINEAR);
glReadBuffer(GL_COLOR_ATTACHMENT1);
glBlitFramebuffer(0, 0, width, height, width/2, 0, width/2, height, GL_COLOR_BUFFER_BIT, GL_LINEAR);
```

# 帧缓冲区失效

帧缓冲区失效为应用程序提供了一个通知驱动程序不再需要帧缓冲区内容的机制。这使得驱动程序可以采取多种优化步骤：（1）跳过在块状渲染（TBR）架构中为了进一步渲染到帧缓冲区而做的不必要的图块内容恢复，（2）跳过多CPU系统中GPU之间不必要的数据复制，（3）跳过某些实现中为了改进性能而对特定缓存的刷新。这种功能对于许多应用程序中实现峰值性能很重要，特别是那些执行大量屏幕外渲染的应用。
让我们研究一下TBR GPU的设计，以理解帧缓冲区失效对这些GPU的重要性。TBR GPU尝尝部署在移动设备上，以最小化GPU和系统内存之间的数据传输量，从而减少最大的电力消耗者之一--内存带宽。这通过添加能够保存少量像素数据的芯片内建快速存储器来实现。然后，帧缓冲区被分为许多个图块。对于每个图块，图元被渲染到芯片内建的存储器中，然后结果在完成时被复制到系统内存。因为每个像素的最少量数据（最终像素结果）被复制到系统内存，所以这种方法节约了GPU和系统内存之间的内存带宽。
有了帧缓冲区失效机制，GPU就可以删除不再需要的帧缓冲区内容，以减少每个帧保留的内容数量。此外，如果图块数据不再有效，GPU还可以消除从芯片内建存储器到系统内存不必要的数据传输，因为GPU和系统内存之间内存带宽需求明显降低，所以电力消耗随之下降，性能则得到改善。
`glInvalidateFramebuffer`和`glInvalidateSubFramebuffer`命令用于使整个帧缓冲区或者帧缓冲区的像素子区域失效：

``` objc
/**
 使整个帧缓冲区失效

 @param target#> 必须设置为GL_READ_FRAMEBUFFER、GL_DRAW_FRAMEBUFFER或GL_FRAMEBUFFER description#>
 @param numAttachments#> attachments列表中的附着数量 description#>
 @param attachments#> 指定包含numAttachments个附着的数组的指针 description#>
 @return void
 */
glInvalidateFramebuffer(GLenum target, GLsizei numAttachments, const GLenum *attachments);

/**
 使帧缓冲区的像素子区域失效

 @param target#> 必须设置为GL_READ_FRAMEBUFFER、GL_DRAW_FRAMEBUFFER或GL_FRAMEBUFFER description#>
 @param numAttachments#> attachments列表中的附着数量 description#>
 @param attachments#> 指定包含numAttachments个附着的数组的指针 description#>
 @param x，y#> 指定要失效的像素矩形的左下角原点（左下角为0，0） description#>
 @param width#> 指定要失效的像素矩形宽度 description#>
 @param height#> 指定要失效的像素矩形高度 description#>
 @return void
 */
glInvalidateSubFramebuffer(GLenum target, GLsizei numAttachments, const GLenum *attachments, GLint x, GLint y, GLsizei width, GLsizei height);
```

# 删除帧缓冲区和渲染缓冲区对象

在应用程序结束渲染缓冲区对象的使用之后，可以删除它们。删除渲染缓冲区和帧缓冲区对象与删除纹理对象非常相似。
渲染缓冲区对象用`glDeleteRenderbuffers`API删除：

``` objc
/**
 删除渲染缓冲区对象

 @param n#> 要删除的渲染缓冲区对象名称的数量 description#>
 @param renderbuffers#> 指向包含n个要删除的渲染x缓冲区对象名称的数组的指针 description#>
 @return void
 */
glDeleteRenderbuffers(GLsizei n, const GLuint *renderbuffers);
```

`glDeleteRenderbuffers`删除`renderbuffers`中指定的渲染缓冲区对象。一旦删除了渲染缓冲区对象，就没有任何与之关联，对象被标记为未用，以后可以被重用，作为新的渲染缓冲区对象。删除当前绑定的渲染缓冲区对象时，该对象被删除，当前渲染缓冲区被重置为0。如果`renderbuffers`中指定的渲染缓冲区对象名称无效或者为0，则它们被忽略（不会生成任何错误）。而且，如果渲染缓冲区连接到当前绑定的帧缓冲区对象，则它首先要与帧缓冲区断开，然后才能被删除。
帧缓冲区对象用`glDeleteFramebuffers`API删除：

``` objc
/**
 删除帧缓冲区对象

 @param n#> 要删除的帧缓冲区对象名称的数量 description#>
 @param framebuffers#> 指向包含n个要删除的帧缓冲区对象名称的数组的指针 description#>
 @return void
 */
glDeleteFramebuffers(GLsizei n, const GLuint *framebuffers);
```

`glDeleteFramebuffers`删除`framebuffers`中指定的帧缓冲区对象。一旦删除了帧缓冲区对象，该对象就不会有任何与之关联的状态，并被标记为未使用，以后可以作为新的帧缓冲区对象重用。删除当前绑定的帧缓冲区对象时，该对象被删除且当前帧缓冲区绑定重置为0。如果`framebuffers`中指定的帧缓冲区对象名称无效或者为0，则它们被忽略，不会生成任何错误。

# 删除用作帧缓冲区附着的渲染缓冲区对象

如果被删除的渲染缓冲区对象作为帧缓冲区对象的一个附着，会发生什么情况？如果要删除的帧缓冲区对象作为当前绑定的帧缓冲区对象中的一个附着，则`glDeleteRenderbuffers`将附着重置为0。如果要删除的渲染缓冲区对象是当前没有绑定的帧缓冲区对象的一个附着，则`glDeleteRenderbuffers`不会将这些附着重置为0。将这些被删除的渲染缓冲区对象与对应的帧缓冲区对象断开是应用程序的责任。

## 读取像素和帧缓冲区对象

`glReadPixels`命令从颜色缓冲区读取像素，并在一个用户分配的缓冲区中返回它们。读取的颜色缓冲区是窗口系统提供的帧缓冲区分配的颜色缓冲区，或者是当前绑定的帧缓冲区对象的颜色附着。当用`glBindBuffer`将一个非零的缓冲区对象绑定到`GL_PIXEL_PACK_BUFFER`时，`glReadPixels`命令可以立即返回，并调用DMA传输从帧缓冲区中读取像素，然后将数据写入像素缓冲区对象。
`glReadPixels`支持多种`format`和`type`参数组合：`format`可以是`GL_RGBA`、`GL_RGBA_INTEGER`或者由查询`GL_IMPLEMENTATION_COLOR_READ_FORMAT`返回的特定于实现的值；`type`可以是`GL_UNSIGNED_BYTE`、`GL_UNSIGNED_INT`、`GL_INT`、`GL_FLOAT`或者由查询`GL_IMPLEMENTATION_COLOR_READ_TYPE`返回的特定于实现的值。返回的特定于实现的格式和类型取决于当前连接的颜色缓冲区的格式和类型。如果当前绑定的帧缓冲区改变，则这些值也可能改变。当前绑定的帧缓冲区对象改变时都必须查询这些值，以确定传递给`glReadPixels`的正确的特定于实现的格式和类型值。

# 性能提示和技巧

下面，我们讨论开发人员在使用帧缓冲区对象时应该认真考虑的性能提示：
- 避免频繁地在渲染到窗口系统提供的帧缓冲区和渲染到帧缓冲区对象之间切换。这对于手持OpenGL ES 3.0实现是一个问题，因为许多这类实现使用块状渲染架构。在块状渲染架构中，使用专用的内部存储器存储帧缓冲区图块（区域）的颜色、深度和模板值。这些内部存储器在电源利用上更为高效，而且与外部内存相比具有更低的内存延迟和更大的带宽。在渲染到图块完成之后，图块被写到设备（系统）内存。每当从一个渲染目标切换到另一个，就需要渲染、保存和恢复对应的纹理和渲染缓冲区附着，这可能带来很大的代价。最佳的方法是首先渲染到场景中合适的帧缓冲区，然后渲染到窗口系统提供的帧缓冲区，最后执行`eglSwapBuffers`命令切换显示缓冲区。
- 不要逐帧创建和删除帧缓冲区和渲染缓冲区对象（或者任何其他大型数据对象）。
- 尝试避免修改用作渲染目标的帧缓冲区对象附着的纹理（使用`glTexImage2D`、`glTexSubImage2D`、`glCopyTexImage2D`等）。
- 如果整个纹理图像将被渲染，则将`glTexImage2D`和`glTexImage3D`中的`pixel`参数设置为`NULL`，因为原始数据不会被使用。如果希望图像不包含任何预先定义的像素值，那么在绘制到纹理之前使用`glInvalidateFramebuffer`清除纹理图像。
- 尽可能共享帧缓冲区对象使用的用作附着的深度和模板渲染缓冲区，以保障内存占用需求最小。

# 总结

这篇文章主要是对用于渲染到屏幕外表面的帧缓冲区对象的使用方法。