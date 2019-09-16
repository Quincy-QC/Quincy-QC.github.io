---
title: "OpenGL ES基础学习"
catalog: true
toc_nav_num: true
date: 2019-05-19 18:20:03
subtitle: "Open Graphics Library for Embedded Systems"
header-img: "/img/article_header/article_header.png"
busuanzi: true
tags: 
- iOS

---

# Introduction

Open Graphics Library(OpenGL)用于可视化2D与3D数据，是多用途的开放标准图形库，支持2D和3D数字内容创建、机械和建筑设计、虚拟原型、飞行仿真、视频游戏等等。我们可以使用OpenGL配置3D图形管线并向其提交数据。顶点被转换点亮，组装成基本类型，并进行栅格化来创建2D图片。OpenGL的设计目的是将函数调用转换为可以发送到底层图形硬件的图形命令，因为底层硬件专门处理图形命令，所以OpenGL的绘图非常快。

OpenGL for Embedded Systems(OpenGL ES)是openGL的简化版，消除冗余功能，为移动图形硬件上提供一个更容易学习和实现的库。

![OpenGL ES](/img/article/20190519/1.png)

OpenGL ES允许应用程序利用底层图形处理器的能力，iOS设备上的GPU可以实现复杂的2D和3D绘图，以及图片上每个像素阴影的复杂计算。如果应用程序的而设计要求需要对GPU硬件有直接全面的访问，那么我们应该使用OpenGL ES。OpenGL ES典型的客户端包括视频游戏和呈现3D图形的模拟器。

# Checklist for Building OpenGL ES Apps for iOS

OpenGL ES为使用GPU硬件渲染图形规范定义了无关平台的API。实现OpenGL ES的平台提供了一个用于执行OpenGL ES命令的渲染上下文，持有渲染结果的帧缓冲区，一个或多个展示帧缓冲区内容的渲染目标。iOS中，*EAGLContext*实现了渲染上下文，iOS只提供了一种帧缓冲区类型，OpenGL ES帧缓冲区对象，和实现渲染地点的*GLKView*、*CAEAGLLayer*。

在iOS中创建OpenGL ES应用需要有以下考虑，一些对于OpenGL ES编程通用的，还有一些是iOS特定的考虑：
1. 决定适用于我们应用的OpenGL ES版本，创建上下文；
2. 在运行时检测设备是否支持我们想要使用的OpenGL ES功能；
3. 选择渲染OpenGL ES内容的位置；
4. 确定应用在iOS中正确运行；
5. 实现渲染引擎；
6. 使用Xcode和Instruments调试OpenGL ES应用，调优以获取最优性能；

## 选择OpenGL ES版本

决定应用需要支持OpenGL ES 3.0，OpenGL ES 2.0，OpenGL ES 1.1还是多个版本：
- OpenGL ES 3.0是在iOS 7的新版本，增加了许多新特性，实现高性能、通用GPU计算技术和之前只能在台式机和游戏机上才能实现的更复杂视觉效果
- OpenGL ES 2.0是iOS设备的基础配置，具有基于可编程着色器的可配置图形管线
- OpenGL ES 1.1仅提供基本的固定函数的图形通道，在iOS中主要用于向后兼容

我们应该针对最相关的特性和应用程序的设备选择一个或多个版本，关于iOS设备功能更多可见[iOS Device Compatibility Reference](https://developer.apple.com/library/archive/documentation/DeviceInformation/Reference/iOSDeviceCompatibility/Introduction/Introduction.html#//apple_ref/doc/uid/TP40013599)

## 验证OpenGL ES功能

[iOS Device Compatibility Reference](https://developer.apple.com/library/archive/documentation/DeviceInformation/Reference/iOSDeviceCompatibility/Introduction/Introduction.html#//apple_ref/doc/uid/TP40013599)总结了在已发布iOS设备中可用的功能和扩展，但是，为了让应用程序能够在尽可能多的设备和iOS版本上运行，我们应该始终在运行时查询OpenGL ES可以实现的功能。

要确定实现的特定限制，比如最大纹理大小或顶点属性的最大数量，使用相应的`glGet`方法查找对应的值（如在*gl.h*文件查找*MAX_TEXTURE_SIZE*或*MAX_VERTEX_ATTRIBS*）。

使用`glGetIntegerv`和`glGetStringi`方法检测OpenGL ES 3.0的扩展：
``` objectivec
/// #import <OpenGLES/ES3/gl.h>
BOOL CheckForExtensions(NSString *searchName) {
    int max = 0;
    glGetIntegerv(GL_NUM_EXTENSIONS, &max);
    NSMutableSet *extensions = [NSMutableSet set];
    for (int i = 0; i < max; i++) {
        [extensions addObject:@((char *)glGetStringi(GL_EXTENSIONS, i))];
    }
    return [extensions containsObject:searchName];
}
/// 对于OpenGL ES 2.0和1.1的扩展，使用glGetString(GL_EXTENSIONS)获取以空格分隔的所有扩展名
```

## 选择渲染位置

iOS中，帧缓冲区对象存储了绘图命令的结果，我们可以通过以下多种方式使用帧缓冲区对象的内容：
- *GLKit*框架提供了绘制OpenGL ES内容和管理自身帧缓冲区的view，和支持动画的控制器。使用这些类创建全屏视图或将OpenGL ES内容放入UIKit视图层次结构中。[Drawing with OpenGL ES and GLKit](#drawing-with-opengl-es-and-glkit)
- *CAEAGLLayer*类提供绘制OpenGL ES内容作为Core Animation层级组合一部分的方法，但我们必须使用这个类创建自己的帧缓冲区
- 与任意OpenGL ES实现一样，我们也可以使用帧缓冲区进行离屏图形处理或渲染图像管线其他地方使用的纹理。在OpenGL ES中，离屏缓冲区可以用于使用多个渲染目标的渲染算法

更多关于离屏缓冲区、纹理、Core Animation层级的渲染，前往[Drawing to Other Rendering Destinatons](#drawing-to-other-rendering-destinations)

## iOS集成

iOS应用默认支持多任务，但在OpenGL ES应用内处理这个特性需要额外考虑，不正确使用OpenGL ES会导致应用在后台被系统杀死。

许多iOS设备包含高分辨率显示，所以我们需要支持多种显示尺寸和分辨率。

了解如何支持这些和其他iOS特性，前往[Multitasking, High Resolution, and Other iOS Features](#multitasking,-high-resolution,-and-other-ios-features)。

## 实现渲染引擎

设计OpenGL ES绘图代码有许多可能的策略，其详细信息超出了文本的范围。渲染引擎设计的许多方面对于OpenGL和OpenGL ES的所有实现都是通用的。

更多iOS设备设计考虑前往[OpenGL ES Design Guidelines](#opengl-es-design-guidelines)和[Concurrency and OpenGL ES](#concurrency-and-opengl-es)。

## 调试与性能分析

Xcode和Instruments提供了许多工具来跟踪渲染问题并分析OpenGL ES的性能，前往[Debugging and Profiling](#debugging-and-profiling)。

# Configuring OpenGL ES Contexts

每个OpenGL ES的实现提供了创建管理OpenGL ES规范所需要状态的渲染上下文的方法，通过在上下文的状态，多个应用可以轻松地共享图形硬件而不会干扰其他应用程序的状态。

在我们使用OpenGL ES方法前，必须初始化*EAGLContext*对象，*EAGLContext*类还提供将OpenGL ES内容与Core Animation集成的方法。

## 当前上下文是OpenGL ES函数调用的目标

iOS应用每个线程都有一个当前上下文，当我们调用OpenGL ES的方法时，这个就是状态要被更改的上下文。

设置当前线程上下文，在对应线程调用`setCurrentContext:`方法：
``` objectivec
[EAGLContext setCurrentContext:context];
```

调用*EAGLContext*类方法`currentContext`来检索当前线程的上下文。

> 如果在相同线程切换两种或以上上下文，在设置新的当前上下文之前调用`glFlush`方法确保之前提交的命令能及时交付给图形硬件。

OpenGL ES对当前上下文*EAGLContext*对象持有强引用，当调用`setCurrentContext:`方法切换上下文则不会对之前的对象强持有，为了防止*EAGLContext*对象在切换时被释放，我们应该对之强引用。

## 每个上下文都针对OpenGL ES的特定版本

一个*EAGLContext*对象仅支持OpenGL ES的一个版本。例如：OpenGL ES 1.1版本下的代码不兼容2.0或3.0版本；使用OpenGL ES 2.0版本特性的版本兼容3.0版本，同时2.0版本的扩展经常可以在3.0版本中少量修改后使用；OpenGL ES 3.0的特性和新增的硬件性能需要3.0版本。

我们在创建和初始化*EAGLContext*对象时选择OpenGL ES的版本。如果设备不支持对应的版本，会返回nil，我们必须保证正确初始化之后使用它。
``` objectivec
EAGLContext* CreateBestEAGLContext() {
    EAGLContext *context = [[EAGLContext alloc] initWithAPI:kEAGLRenderingAPIOpenGLES3];
    if (context == nil) {
        context = [[EAGLContext alloc] initWithAPI:kEAGLRenderingAPIOpenGLES2];
    }
    return context;
}
```

## EAGL Sharegroup

尽管上下文持有OpenGL ES的状态，但它不直接管理OpenGL ES的对象。相反，可以通过*EAGLSharegroup*对象创建和持有OpenGL ES对象，每个上下文都包含一个被委托创建对象的*ESGLSharegroup*对象。

在两个或多个上下文引用一个共享组时优势很明显，这时创建OpenGL ES的对象在所有上下文都可用，如果绑定到另一个上下文与创建它的上下文有相同的标识符，则引用相同的OpenGL ES对象。移动设备的资源非常稀缺，在上下文创建多个相同内容的拷贝是浪费资源的，公共资源可以更好的利用设备的图形资源。

sharegroup是一个不透明对象，没有可以调用的属性或方法，可以使用sharegroup对象的上下文对之进行强引用。

![OpenGL ES Sharegroup](/img/article/20190519/2.png)

sharegroup在以下两种情况下最有用：
- 上下文之间的大多数共享资源不会改变
- 当我们想要在其他线程创建OpenGL ES对象而主线程用于渲染。例如：第二个上下文在单独的线程上用于获取数据和创建资源，加载资源后，第一个上下文可以绑定到对象并立即使用它。*GLKtextureLoader*类就是使用这个模式来提供异步纹理加载。

想要创建引用相同sharegroup的上下文，第一个上下文使用`initAPI:`方法初始化后会自动创建sharegroup，第二个及后续上下文调用`initAPI:sharegroup:`使用第一个上下文的sharegroup进行初始化。

``` objectivec
EAGLContext* firstContext = CreateBestEAGLContext();
EAGLContext* secondContext = [[EAGLContext alloc] initWithAPI:[firstContext API] sharegroup: [firstContext sharegroup]];
```

当sharegroup由多个上下文共享时，我们有责任管理OpenGL ES对象的状态更改：
- 应用程序可以同时跨多个上下文访问未修改的对象
- 当对象被发送到上下文的命令修改时，不能在其他地方读写该对象
- 在对象修改后，所有上下文必须重新绑定才能看到更改。如果上下文在绑定之前引用该对象，则该对象的内容是未定义的

下面是更新OpenGL ES对象的步骤：
- 在可能使用对象的每个上下文调用`glFlush`方法
- 在想要修改对象的上下文，调用一种或多种OpenGL ES方法修改对象
- 在接收状态修改命令的上下文中调用`glFlush`方法
- 在每个其他上下文，重新绑定标识符

> 另一种共享对象的方法是使用一个渲染上下文，多个目标帧缓冲区。在渲染时，应用程序绑定合适的帧缓冲区并根据需要渲染帧，因为所有的OpenGL ES对象都是从一个上下文应用，所有会看到相同的OpengGL ES数据。这种模式使用的资源较少，但只适用于单线程应用程序，但单线程程序中可以仔细控制上下文状态。

# Drawing with OpenGL ES and GLKit

GLKit框架提供了视图和控制器类，消除了绘画和动画OpenGL ES内容所需的设置和维护代码，*GLKView*类管理OpenGL ES的基础结构，为绘图代码提供空间，*GLKViewController*类提供一个渲染循环用于GLKit视图上平滑的展示OpenGL ES内容的动画。这些类扩展了用于绘制视图内容和管理视图呈现的标准UIKit设计模式。因此，我们可以将主要精力用于OpenGL ES渲染代码与应用流畅度上。GLKit框架同样提供简化OpenGL ES 2.0与3.0开发的其他特性。

## GLKit视图根据需求绘制OpenGL ES内容

*GLKView*类提供了一个基于OpenGL ES等价于标准UIView的绘图周期。*UIView*对象自动配置图形上下文，所以`drawRect:`实现只需要执行Quartz 2D的绘图命令，*GLKView*对象自动配置自己所以我们的绘图方法只需要执行OpenGL ES的绘图命令。*GLKView*类通过维护持有OpenGL ES绘图命令结果的帧缓冲区来提供此功能，然后在绘图方法返回时自动将他们呈现给Core Animation。

与标准UIKit视图一样，GLKit视图按需渲染。当视图第一次显示，它调用我们的绘图方法--Core Animation缓冲区渲染输出并在视图显示时显示它。当我们想要修改视图内容，调用`setNeedsDisplay`方法，视图会重新调用我们的绘图方法，缓冲区结果图，在屏幕显示它。这个方法在渲染图像的数据修改很少或只响应用户操作时有用，只有在需要的时候渲染新视图内容，这样可以节约设备电量并为其他操作留出更多时间。

![Rending OpenGL ES content with a GLKit view](/img/article/20190519/3.png)

### 创建与配置GLKit View

我们可以以编程的方式创建配置一个GLKView对象，也可以用故事板，在绘图钱，我们需要与之关联*EAGLContext*对象。

- 当以编程方式创建一个视图时，先创建一个上下文然后将它用于*initWithFrame:context:*方法
- 当从故事板加载一个视图，创建一个上下文然后设置为这个视图的*context*属性

一个GLKit视图自动创建和配置他自己的OpenGL ES帧缓冲区对象和渲染缓冲区。我们使用视图的绘图属性控制这些对象属性，当我们改变尺寸大小、比例系数、或者某个绘画属性，它会在下次内容绘制时自动删除和重新创建相应的帧缓冲区和渲染缓冲区对象。

``` objectivec
- (void)viewDidLoad {
    [super viewDidLoad];
    
    EAGLContext *context = CreateBestEAGLContext();
    GLKView *glkView = [[GLKView alloc] initWithFrame:self.view.bounds context:context];
    // Configure renderbuffers
    glkView.drawableColorFormat = GLKViewDrawableColorFormatSRGBA8888;
    glkView.drawableDepthFormat = GLKViewDrawableDepthFormat24;
    glkView.drawableStencilFormat = GLKViewDrawableStencilFormat8;
    // enable multisampling
    // multisampling是一种消除锯齿边缘的反锯齿，在大多数3D应用程序中提高图形质量，代价是使用更多的内存和片段处理时间
    glkView.drawableMultisample = GLKViewDrawableMultisample4X;
    [self.view addSubview:glkView];
}
```

### 使用GLKit View绘图

绘制OpenGL ES内容分为三步：准备OpenGL ES基础设施，执行绘图命令，将渲染内容提交给Core Animation显示。其中*GLKView*实现了第一步与第三步，下面代码实现了绘图的第二步：

``` objectivec
- (void)drawRect:(CGRect)rect {
    // clear the framebuffer
    glClearColor(0.0, 0.0, 0.1, 1.0);
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
    // Draw using previously configured texture, shader, uniforms, and vertex array
    glBindTexture(GL_TEXTURE_2D, _planetTexture);
    glUseProgram(_diffuseShading);
    glUniformMatrix4fv(_uniformModelViewProjectionMatrix, 1, 0, _modelViewProjectionMatrix.m);
    glBindVertexArray(_planetMesh);
    glDrawElements(GL_TRIANGLE_STRIP, 256, GL_UNSIGNED_SHORT, NULL);
}
```

> *glClear*方法提示OpenGL ES任何现有的帧缓冲区内容都可以被丢弃，避免加载以前的内容到内存中造成高消耗内存操作。为了确保最佳性能，我们应该在每次绘图前总是调用这个函数

*GLKView*可以提供一些简单的接口给OpenGL ES绘图，因为它管理了OpenGL ES渲染过程中的标准部分：

- 在调用绘图方法前，视图需要：
    + 确保*EAGLContext*为当前上下文
    + 基于现有尺寸、比例系数、绘图属性创建帧缓冲区对象和渲染缓冲区（如有必要）
    + 将帧缓冲区对象绑定为绘制命令的当前目标
    + 设置OpenGL ES视图端口以匹配帧缓冲区大小
- 在绘图方法返回后，视图需要：
    + 解析multisampling缓冲区（如果multisampling已设置）
    + 丢弃不再需要的渲染缓冲区
    + 将渲染缓冲区内容提交给Core Animation缓冲区和展示

## 使用代理渲染

许多OpenGL ES应用程序使用自定义类实现渲染代码，这个方法的优势在于它能为多个渲染器分别定义不同的渲染类，从而轻松地支持多个渲染算法，共享公共功能的渲染算法可以从父类继承。例如，我们可以使用不同的渲染类同时支持OpenGL ES 2.0和3.0，或者使用它们在具有更强大硬件的设备上定制渲染，以获得更优质的图片。

GLKit非常适合这种方法--我们可以使用渲染器对象成为标准GLKView实例的代理，没有使用继承GLKView和实现`drawInRect:`方法，而是使用渲染器类实现*GLKViewDelegate*代理并实现`glkView:drawInRect:`方法。下面代码演示了应用程序在启动时根据硬件特性选择渲染器类：

``` objectivec
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    // Create a context
    EAGLContext *context = [[EAGLContext alloc] initWithAPI:kEAGLRenderingAPIOpenGLES2];
    [EAGLContext setCurrentContext:context];
    
    // Choose a rendering class based on device features
    GLint maxTextureSize;
    glGetIntegerv(GL_MAX_TEXTURE_SIZE, &maxTextureSize);
    if (maxTextureSize > 2048) {
        self.renderer = [[MyBigTextureRenderer alloc] initWithContext:context];
    } else {
        self.renderer = [[MyRenderer alloc] initWithContext:context];
    }
    GLKView *view = (GLKView *)self.window.rootViewController.view;
    view.delegate = self.renderer;
    view.context = context;
    return YES;
}
```

## GLKit视图控制器动画OpenGL ES内容

默认情况下，一个*GLKView*对象按需渲染内容。也就是说，使用OpenGL ES绘图的一个关键优势是他能使用图形处理硬件对复杂场景进行连续动画--游戏和模拟器等应用程序很少使用静态图片。对于这些情况，*GLKit*框架提供一个视图控制器类，为它管理的*GLKView*对象维护一个动画循环，这个循环遵循游戏和模拟器中场景中常见的设计模式，分为两个阶段：更新与显示。下图展示了一个简单的动画循环示例：

![The animation loop](/img/article/20190519/4.png)

### 动画循环的理解

更新阶段，视图控制器调用它自身的*update*方法（或者当未继承GLKViewController时可以使用代理的`glkViewControllerUpdate:`方法），这个方法中，我们应该为下一帧绘图做准备。例如，一个游戏可能使用这个方法根据上一帧以来接受到的输入事件来决定玩家和敌对角色的位置，科学的可视化可以使用这种方法运行模拟的一个步骤。如果我们需要时间信息来决定应用程序下一帧的状态，使用这个视图控制器的其中一个时间属性比如`timeSinceLastUpdate`属性。

展示阶段，试图控制器调用视图的`display`方法，这个方法会触发绘图方法。在绘图方法中，我们对GPU提交OpenGL ES的绘图指令来渲染我们的内容。为了最佳性能。我们的应用程序应该在渲染最新一帧时修改OpenGL ES对象，然后提交绘图指令。

动画循环以视图控制器的`framesPerSecond`属性所指示的速率在这两个阶段切换，我们可以使用`preferredFramesPerSecond`属性来设置所需的帧速率--为了当前显示硬件的最优性能，视图控制器自动渲染接近于我们设置值的最优帧速率。

> 为了获得最佳效果，选择一个应用程序可以达到的帧率。平滑、一致的帧率比经常变化的帧率能有更好的用户体验

### 使用GLKit视图控制器

``` objectivec
@implementation PlanetViewController // subclass of GLKViewController
 
- (void)viewDidLoad {
    [super viewDidLoad];
 
    // Create an OpenGL ES context and assign it to the view loaded from storyboard
    GLKView *view = (GLKView *)self.view;
    view.context = [[EAGLContext alloc] initWithAPI:kEAGLRenderingAPIOpenGLES2];
 
    // Set animation frame rate
    self.preferredFramesPerSecond = 60;
 
    // Not shown: load shaders, textures and vertex arrays, set up projection matrix
    [self setupGL];
}
 
- (void)update {
    _rotation += self.timeSinceLastUpdate * M_PI_2; // one quarter rotation per second
 
    // Set up transform matrices for the rotating planet
    GLKMatrix4 modelViewMatrix = GLKMatrix4MakeRotation(_rotation, 0.0f, 1.0f, 0.0f);
    _normalMatrix = GLKMatrix3InvertAndTranspose(GLKMatrix4GetMatrix3(modelViewMatrix), NULL);
    _modelViewProjectionMatrix = GLKMatrix4Multiply(_projectionMatrix, modelViewMatrix);
}
 
- (void)glkView:(GLKView *)view drawInRect:(CGRect)rect {
    // Clear the framebuffer
    glClearColor(0.0f, 0.0f, 0.1f, 1.0f);
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
 
    // Set shader uniforms to values calculated in -update
    glUseProgram(_diffuseShading);
    glUniformMatrix4fv(_uniformModelViewProjectionMatrix, 1, 0, _modelViewProjectionMatrix.m);
    glUniformMatrix3fv(_uniformNormalMatrix, 1, 0, _normalMatrix.m);
 
    // Draw using previously configured texture and vertex array
    glBindTexture(GL_TEXTURE_2D, _planetTexture);
    glBindVertexArrayOES(_planetMesh);
    glDrawElements(GL_TRIANGLE_STRIP, 256, GL_UNSIGNED_SHORT, 0);
}
@end
```

`viewDidLoad`方法创建了OpenGL ES上下文并提供给视图，同时设置动画循环的帧率。视图控制器自动成为视图的代理，同时实现动画循环的更新与显示阶段。在`update`方法中，它计算显示旋转行星所需的变换矩阵。在`glkView:drawInRect:`方法，它将这些矩阵提供给着色器程序，并提交绘制命令来渲染行星的几何形状。

# Drawing to Other Rendering Destinations

帧缓冲区对象是渲染命令的目标。当创建一个帧缓冲区对象，我们可以精确控制它存储的颜色、深度和模板数据。我们通过将图片附加到帧缓冲区来提供这种存储，如下图所示。最常见的图片关联是渲染缓冲区对象。我们也可以将OpenGL ES的纹理附加到帧缓冲区的颜色连接点，这意味着所有的绘图命令都将渲染到纹理中。稍后，纹理可以作为未来渲染命令的输入。我们还可以在一个渲染上下文创建多个帧缓冲区对象，这么做可以在多个帧缓冲区之间共享相同的渲染管线和OpenGL ES资源。

![Famebuffer with color and depth renderbuffers](/img/article/20190519/5.png)

所有这些方法都需要手动创建帧缓冲区和渲染缓冲区对象来存储OpenGL ES上下文的渲染结果，还需要编写额外的代码来将内容呈现到屏幕，如有必要还要运行一个动画循环。

## 创建帧缓冲区对象

根据应用程序打算执行的任务，将配置不同的对象附加到帧缓冲区对象。在多数情况下，配置帧缓冲区的区别在于对象被附加到对象的颜色连接点上：
- 使用帧缓冲区用于离屏图像处理，附加一个渲染缓冲区。见[创建离屏缓冲区对象](#创建离屏缓冲区对象)
- 使用帧缓冲区图片作为后续渲染步骤的输入，附加一个纹理。见[使用帧缓冲区对象渲染纹理](#使用帧缓冲区对象渲染纹理)
- 在Core Animation层级组合中使用帧缓冲区，使用特定Core Animation-aware渲染缓冲区。见[渲染到Core Animation层级](#渲染到Core-Animation层级)

### 创建离屏缓冲区对象

用于离屏渲染的帧缓冲区分配内存给它所有的关联作为OpenGL ES的渲染缓冲区。下面代码分配内存给一个带有颜色和深度关联的帧缓冲区对象：

1. 创建帧缓冲区并绑定：
``` objectivec
GLuint framebuffer;
glGenFramebuffers(1, &framebuffer);
glBindFramebuffer(GL_FRAMEBUFFER, framebuffer);
```

2. 创建颜色渲染缓冲区，分配内存，与帧缓冲区的颜色连接点相关联：
``` objectivec
GLuint colorRenderbuffer;
glGenRenderbuffers(1, &colorRenderbuffer);
glBindRenderbuffer(GL_RENDERBUFFER, colorRenderbuffer);
glRenderbufferStorage(GL_RENDERBUFFER, GL_RGBA, width, height);
glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_RENDERBUFFER, colorRenderbuffer);
```

3. 创建一个深度或者深度/模板渲染缓冲区，分配内存，与帧缓冲区深度关联点相关联：
``` objectivec
GLuint depthRenderbuffer;
glGenRenderbuffers(1, &depthRenderbuffer);
glBindRenderbuffer(GL_RENDERBUFFER, depthRenderbuffer);
glRenderbufferStorage(GL_RENDERBUFFER, GL_DEPTH_COMPONENT16, width, height);
glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, GL_RENDERBUFFER, depthRenderbuffer);
```

4. 测试帧缓冲区的完整性，这个测试方法只需要在帧缓冲区配置改动时执行：
``` objectivec
GLenum status = glCheckFramebufferStatus(GL_FRAMEBUFFER);
if (status != GL_FRAMEBUFFER_COMPLETE) {
    NSLog(@"failed to make complete framebuffer object %x", status);
}
```

在绘图到离屏渲染缓冲区后，使用`glReadPixels`方法返回它的内容到CPU做进一步处理。

### 使用帧缓冲区对象渲染纹理

创建帧缓冲区的代码几乎与离屏实例一样，但现在是纹理分配内存并与颜色关联点相关联：
1. 创建帧缓存对象（与上例相似）；
2. 创建目标纹理，与帧缓冲区的颜色关联点相关联：
``` objectivec
GLuint texture;
glGenTextures(1, &texture);
glBindTexture(GL_TEXTURE_2D, texture);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);   // 纹理过滤
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA8, width, height, 0, GL_RGBA, GL_UNSIGNED_BYTE, NULL);
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, texture, 0);
```
3. 分配缓存并关联深度缓冲区（与上例类似）；
4. 验证帧缓冲区的完整性；

虽然这个例子假设我们正在渲染到颜色纹理，但也可以使用其他选项。例如，使用*OES_depth_texture*扩展，可以将纹理关联到深度关联点，以将场景中的深度信息存储到纹理。我们可以使用这个深度信息计算最终渲染场景中的阴影。

### 渲染到Core Animation层级

Core Animation是iOS上图形渲染和动画的核心基础设施，我们可以使用不同iOS子系统渲染的层级（如UIKit、Quartz 2D、OpenGL ES）来组合应用程序的用户界面或其他可视化显示。OpenGL ES通过*CAEAGLLayer*类关联Core Animation，这个类是一种特殊类的Core Animation层级，其内容来自OpenGL ES渲染缓冲区。Core Animation将渲染缓冲区的内容和其他层级组合在一起，并将结果图像显示在屏幕上。

![Core Animation shares the renderbuffer width OpenGL ES](/img/article/20190519/6.png)

*CAEAGLLayer*通过提供了两项关键功能为OpenGL ES提供支持。首先，它为渲染缓冲区分配内存共享存储，其次，它将渲染缓冲区呈现到Core Animation，用渲染缓冲区中的数据替换层级之前的内容。该模型的优点在于Core Animation层级不需要每一帧都绘制，只需要在渲染图像发生改变时绘制。

> GLKView类自动执行以下步骤，所以当我们想要在视图的内容层中使用OpenGL ES绘图时，应该使用它。

使用Core Animation层级的OpenGL ES渲染：
1. 创建*CAEAGLLayer*对象并配置它的属性。想要最佳性能，设置层级的*opaue*属性为YES。可选，通过为*CAEAGLLayer*对象的`drawableProperties`属性分配一个新的字典来配置渲染表面的表面属性；可以指定渲染缓冲区的像素格式，并指定渲染缓冲区的内容在发送到Core Animation后是否被丢弃。有关允许的键列表，见[EAGLDrawable Protocol Reference](https://developer.apple.com/documentation/opengles/eagldrawable?language=objc)；
2. 创建OpenGL ES上下文，并设置为当前上下文；
3. 创建帧缓冲区对象；
4. 创建颜色渲染缓冲区，调用`renderbufferStorage:fromDrawable:`方法分配存储内存并将层级对象作为参数传入。宽度、高度、像素格式都是从层级获取并用于渲染缓冲区的分配存储内存：
``` objectivec
GLuint colorRenderbuffer;
glGenRenderbuffers(1, &colorRenderbuffer);
glBindRenderbuffer(GL_RENDERBUFFER, colorRenderbuffer);
[myContext renderbufferStorage:GL_RENDERBUFFER fromDrawable:myEAGLLayer];
glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_RENDERBUFFER, colorRenderbuffer);
```
> 当Core Animation的层级`bounds`或属性改变时，必须重新分配渲染缓冲区存储内存。如果不重新分配内存，渲染缓冲区的大小可能不匹配层级的大小。
5. 检索颜色渲染缓冲区的高度和宽度：
```
GLuint width, height;
glGetRenderbufferParameteriv(GL_RENDERBUFFER, GL_RENDERBUFFER_WIDTH, &width);
glGetRenderbufferParameteriv(GL_RENDERBUFFER, GL_RENDERBUFFER_HEIGHT, &height);
```
在前面的示例中，显示地提供了渲染缓冲区的宽度和高度以便分配内存。这里，代码在分配存储内存后检索颜色渲染缓冲区的宽度和高度。应用程序之所以这样做，是因为颜色缓冲区的实际尺寸是根据层级的边界大小和缩放因子计算的。关联到帧缓冲区的渲染缓冲区必须具有相同的纬度。除了使用高度和宽度分配深度缓冲区内存，还可以用来分配OpenGL ES视图端口，并帮助确定应用程序的纹理和模型中所需的细节级别；
6. 分配缓存并关联深度缓冲区；
7. 验证帧缓冲区的完整性；
8. 通过将*CAEAGLLayer*对象传递给`addSublayer:`一个可见层级的方法添加到Core Animation的层级结构中。

## 绘制到帧缓冲区对象

现在我们已经拥有了一个帧缓冲区对象，然后就是填充。下面介绍了渲染新帧并呈现给用户所必要的步骤，渲染到纹理或离屏帧缓冲区的行为类似，只是应用程序使用最终帧的方式不同。

### 按需或使用动画循环渲染

我们在渲染到一个Core Animation层级时必须选择何时绘制我们的OpenGL ES内容，就像使用GLKit视图和控制器绘图一样。如果渲染到离屏帧缓冲区或纹理，则在适合使用这些帧缓冲区类型的情况下随时绘制。

对于on-demand绘制，实现我们自己的方法绘制并呈现渲染缓冲区，并在需要显示新内容的时候调用它。

对于动画循环绘制，使用`CADisplayLink`对象。`displayLink`是由Core Animation提供的一种定时器，允许我们将绘图同步到屏幕的刷新率。下面代码展示了我们如何检索显示视图的屏幕，使用这个屏幕创建一个新的`displayLink`对象并添加到运行环中。

> *GLKViewController*类自动使用`CADisplayLink`对象来动画`GLKView`的内容，只有当我们需要超出*GLKit*框架所提供的行为时，才直接使用`CADisplayLink`类

```
CADisplayLink *displayLink = [self.view.window.screen displayLinkWithTarget:self selector:@selector(drawFrame)];
[displayLink addToRunLoop:[NSRunLoop currentRunLoop] forMode:NSDefaultRunLoopMode];
```

在`drawFrame`方法的实现中，读取`displayLink`的`timestamp`属性来获取渲染下一帧的时间戳，可以用来计算下一帧对象的位置。

通常，每次屏幕刷新都会触发`displayLink`对象；该值通常为60Hz，但在不同的设备上有所不同。大多数应用程序不需要每秒更新屏幕60次。我们可以设置`displayLink`的`frameInterval`属性设置为调用该方法之前经过的实际帧数。例如，如果帧间隔设置为3，应用程序则每隔3帧调用一次，或者每秒大约调用20帧（原帧率的三分之一）。

> 为了最佳效果，渲染一个应用程序可以始终实现的帧率。

### 渲染帧

下图展示了OpenGL ES应用程序渲染和呈现帧的步骤，这些步骤包括很多提示以提高应用程序的性能。

![iOS OpenGL Rendering Steps](/img/article/20190519/7.png)

#### 清空缓冲区

在每一帧开始前，删除所有帧缓冲区附加的下一帧不需要使用的内容。调用`glClear`方法，传递一个位掩码清除所有缓冲区：

``` objectivec
glBindFramebuffer(GL_FRAMEBUFFER, framebuffer);
glClear(GL_DEPTH_BUFFER_BIT | GL_COLOR_BUFFER_BIT);
```

使用`glClear`提示OpenGL ES可以丢弃渲染缓冲区或纹理的现有内容，从而避免将先前内容加载到内存中的昂贵内存。

#### 准备资源并执行绘图命令

这两个步骤包含了设计应用程序架构时所做的大多数关键决策。首先，决定我们想要展示给用户的内容并配置相应的OpenGL ES对象上传到GPU（如顶点缓冲对象，纹理，着色器程序及其输入变量）。下一步，提交绘图命令告诉GPU如何使用这些资源渲染帧。

渲染器设计的更多细节见[OpenGL ES Design Guidelines](#opengl-es-design-guidelines)。目前，最需要注意的最重要的性能优化是，如果只在渲染新帧时修改OpenGL ES对象应用程序运行的更快。虽然应用程序可以在修改对象和提交绘图命令中间进行切换，但如果每帧只执行一个步骤会运行地更快。

#### 执行绘图命令

这一步获取我们上一步准备的对象并提交绘图命令使用它们。设计渲染代码这一部分以高效运行的更多细节见[OpenGL ES Design Guidelines](#opengl-es-design-guidelines)。目前，最需要注意的最重要的性能优化是，如果只在渲染新帧时修改OpenGL ES对象应用程序运行的更快。虽然应用程序可以在修改对象和提交绘图命令中间进行切换，但如果每帧只执行一个步骤会运行地更快。

#### 解决多重采样

如果应用程序使用反锯齿提高图形质量，需要在呈现给用户之前解析像素。更多细节见[使用多重采样提高图片质量](#使用多重采样提高图片质量)。

#### 丢弃不需要的渲染帧缓冲区

丢弃操作是一个性能提示，告诉OpenGL ES一个或多个渲染缓冲区的内容不再需要。通过提示OpenGL ES我们不需要一个渲染缓冲区的内容，缓冲区里的数据被丢弃，可以避免缓冲区更新的复杂任务。

在渲染循环的这个阶段，应用程序已经提交了所有的绘图命令。虽然应用程序需要颜色渲染缓冲区来显示在屏幕上，但可能不需要深度缓冲区的内容。

``` objectivec
const GLenum discards = {GL_DEPTH_ATTACHMENT};
glBindFramebuffer(GL_FRAMEBUFFER, framebuffer);
glDiscardFramebufferEXT(GL_FRAMEBUFFER, 1, discards)
```

> `glDiscardFramebufferEXT`方法由OpengGL ES1.0和2.0的`EXT_discard_framebuffer`扩展提供。在OpenGL ES 3.0上下文中，使用`glInvalidateFramebuffer`方法。

#### 呈现结果到Core Animation

在这个步骤，颜色渲染缓冲区持有完成帧，所以我们需要做的就是呈现给用户。下面代码将renderbuffer绑定到上下文并呈现。这使得完成帧被交到Core Animation。

``` objectivec
glBindRenderbuffer(GL_RENDERBUFFER, colorRenderbuffer);
[context presentRenderbuffer: GL_RENDERBUFFER];
```

默认情况下，我们必须保证应用程序呈现完渲染缓冲区后内容被丢弃。这意味着每次呈现帧时，当渲染新帧时必须完整重新创建帧内容，由于这个原因，上述的代码总是擦除颜色缓冲区。

如果应用程序想要在帧之间保存颜色缓冲区的内容，那么在`CAEAGLLayer`对象的`drawableProperties`属性字典添加`kEAGLDrawablePropertyRetainedBacking`键为`YES`，同时在`glClear`方法调用中移除`GL_COLOR_BUFFER_BIT`常量。保留备份可能需要iOS分配额外的内存来存储缓冲区内容，可能会降低应用程序的性能。

## 使用多重采样提高图片质量

多重采样是反锯齿的一种形式，平滑锯齿边缘，提高大多数3D应用的图像质量。OpenGL ES 3.0将多重采样作为核心规范的一部分，OpenGL ES 1.0和2.0合一通过`APPLE_framebuffer_multisample`扩展提供。多重采样使用更多的内存和片段处理时间来渲染图片，但相比其他方法使用更低的性能成本提高图像质量。

下图展示了多重采样的工作原理。应用程序不是创建一个帧缓冲区，而是两个。多重采样缓冲区包含所有必要的渲染内容关联（通常是颜色和深度缓冲区），解析缓冲区只包含必要的展示渲染图片给用户的关联（通常是颜色渲染缓冲区，但可能是纹理）。多重采样渲染缓冲器使用和解析渲染缓冲器相同的维度分配内存，但每个维度都包含一个指定每个像素存储的样本数量的额外参数。应用程序将所有的渲染都执行到多重采样缓冲区，然后将这些样本解析到解析缓冲器中生成最终反锯齿图像。

![How multisampling](/img/article/20190519/8.png)

下面代码展示了多重采样缓冲区的创建，使用之前创建缓冲器的宽高。通过调用`glRenderbufferStorageMultisampleAPPLE`方法创建渲染缓冲器的多重采样存储；

``` objectivec
GLuint sampleFramebuffer;
glGenFramebuffers(1, &sampleFramebuffer);
glBindFramebuffer(GL_FRAMEBUFFER, sampleFramebuffer);

GLuint sampleColorRenderbuffer;
glGenRenderbuffers(1, &sampleColorRenderbuffer);
glBindRenderbuffer(GL_RENDERBUFFER, sampleColorRenderbuffer);
glRenderbufferStorageMultisample(GL_RENDERBUFFER, 4, GL_RGBA8_OES, width, height);
glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_RENDERBUFFER, sampleColorRenderbuffer);

GLuint sampleDepthRenderbuffer;
glGenRenderbuffers(1, &sampleDepthRenderbuffer);
glBindRenderbuffer(GL_RENDERBUFFER, sampleDepthRenderbuffer);
glRenderbufferStorageMultisample(GL_RENDERBUFFER, 4, GL_DEPTH_COMPONENT16, width, height);
glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, GL_RENDERBUFFER, sampleDepthRenderbuffer);

if (glCheckFramebufferStatus(GL_FRAMEBUFFER) != GL_FRAMEBUFFER_COMPLETE) {
    NSLog(@"Failed to make complete framebuffer object %x", glCheckFramebufferStatus(GL_FRAMEBUFFER));
}
```

下面是一些基于多重采样的修改之后的渲染代码：
1. 在清除缓冲区步骤，需要同时清除多重采样帧缓冲区内容：

``` obejctivec
glBindFramebuffer(GL_FRAMEBUFFER, sampleFramebuffer);
glViewport(0, 0, framebufferWidth, framebufferHeight);  // 选取绘图区域
glClear(GL_COLOR_BUFFER_BIT, GL_DEPTH_BUFFER_BIT);
```

2. 在提交绘图命令后，需要将内容从多重缓冲区解析到解析缓冲区。每像素存储的样本将合并到解析缓冲区的单个样本中：

``` objectivec
glBindFramebuffer(GL_DRAW_FRAMEBUFFER_APPLE, resolveFramebuffer);
glBindFramebuffer(GL_READ_FRAMEBUFFER_APPLE, smapleFramebuffer);
glResolveMultisampleFramebufferAPPLE();
```

3. 在丢弃步骤，我们可以丢弃多重采样帧缓冲区关联的两个渲染缓冲区，这是因为预计呈现的内容以及存储到解析帧缓冲区内：

``` objectivec
const GLenum discards[] = {GL_COLOR_ATTACHMENT0, GL_DEPTH_ATTACHMENT};
glDiscardFramebufferEXT(GL_READ_FRAMEBUFFER_APPLE, 2, discards);
```

4. 在呈现结果步骤，呈现关联到解析帧缓冲区的颜色渲染缓冲区：

``` objectivec
glBindRenderBuffer(GL_RENDERBUFFER, colorRenderbuffer);
[context presentRenderbuffer: GL_RENDERBUFFER];
```

多重采样并不是免费的；存储额外样本的内存消耗，解析样本到解析帧缓冲区的时耗。如果我们想要添加多重采样到应用中，必须多测试性能以确保它可接受。

> 上述代码基于OpenGL ES 1.0与2.0，多重采样位于OpenGL ES 3.0的核心API，使用不同的方法。

# Multitasking, High Resolution, and Other iOS Features

使用OpenGL ES的许多方面都是平台中立，但是在iOS上使用有一些细节需要特别考虑。特别是，iOS应用使用OpenGL ES正确处理多任务或进入后台有被终止的风险。在iOS设备开发OpenGL ES内容时，我们应该考虑现实分辨率和其他设备的特性。

## 实现一个多任务感知的OpenGL ES应用程序

应用程序在用户切换到其他程序时可以继续运行。更多多任务讨论见[App States and Multitasking](https://developer.apple.com/library/archive/documentation/iPhone/Conceptual/iPhoneOSProgrammingGuide/BackgroundExecution/BackgroundExecution.html#//apple_ref/doc/uid/TP40007072-CH4)

使用OpenGL ES的应用程序在切入后台时必须执行更多的工作，如果多任务处理不当，那应用可能会崩溃，同样，一个应用程序可能想要释放OpenGL ES资源这样就只适用于前台工作。

### 后台应用程序可能无法在图像硬件上执行命令

如果OpenGL ES应用程序视图在图像硬件上执行OpenGL ES命令可能会崩溃。iOS阻止后台应用程序访问图像处理器，因此最前端的应用程序总是能给用户提供更好的体验。应用程序崩溃不止在后台调用OpenGL ES，还会在进入后台时之前提交的命令刷新到GPU。所以我们必须保证所有之前提交的命令在进入后台前都执行完毕。

如果使用GLKit视图和控制器，只在绘图方法中提交OpenGL ES命令，应用程序会自动在进入后台时正确运行。默认情况下，在应用程序处于非活动时`GLKViewController`类暂停其动画计时器，确保绘图方法不被调用。

如果不适用GLKit视图或者控制器或者在GLKView绘图方法之外提交OpenGL ES命令，我们必须采用以下步骤保证程序在后台不崩溃：
1. 在`applicationWillResignActive:`方法中，需要停止动画计时器，将自己置于一个已知的良好状态，并调用`glFinish`方法；
2. 在`applicationDidEnterBackground:`方法中，应用程序可能想要删除一些OpenGL ES对象使内存和资源对于前台应用程序可。调用`glFinish`方法确保立即删除资源；
3. 在`applicationDidEnterBackground:`方法中，确保没有OpenGL ES的调用，如果有任何调用就会导致崩溃；
4. 在`applicationWillEnterForeground:`方法中，重新创建对象和启动动画计时器。

总而言之，需要调用`glFinish`方法保证所有之前提交的命令都从命令缓存中取出并被OpenGL ES执行。在进入后台后，在进入前台之前必须避免使用OpenGL ES。

### 在进入后台前删除容易重新创建的资源

在应用程序进入后台时从来不需要释放OpenGL ES对象，通常情况，我们应该避免处理它的内容。考虑两种情况：
- 应用正在玩游戏并短暂切出去检查日历，当用户返回游戏，游戏的资源仍然在内存中并可以直接继续游戏；
- 当用户启动另一个OpenGL ES应用程序，如果需要更多的资源那么系统会自动终止后台的OpenGL ES应用程序让它不执行任何额外的工作。

我们的目标应该是把应用程序设计成一个良好公民：这意味着移动到前台的时间尽可能短，同时减少他在后台的内存占用。

下面是我们应该处理的两种情况：
- 应用程序应该保证纹理，模型和其他资源在内存中；当应用进入后台时，需要长时间重新创建的资源不应该被处理掉；
- 应用程序应该处理可以快速易创建的对象，寻找消耗巨大内存的对象。

简单的目标是应用程序分配内存持有渲染结果的帧缓冲区，当应用程序在后台时，它对用户是不可见的并可能不会使用OpenGL ES渲染任何新内容。这意味着应用程序的帧缓冲区分配了大量内存但并没有用。同样，帧缓冲区的内容是临时的，大多数应用程序在每次渲染新帧时都会重新创建帧缓冲区的内容。这是用渲染缓冲区成为一个内存密集型资源，可以很容易创建，成为移动到后台时可以处理的候选对象。

如果我们使用GLKit视图和控制器，`GLKViewController`类自动在进入后台时处理关联的视图帧缓冲区。如果我们为其他用途手动创建了帧缓冲区，那在进入后台时应该处理他们。在这两种情况下，我们应该考虑应用程序当时可以处理哪些其他临时资源。

## 支持高分辨率显示

默认情况下，GLKit视图的`contentScaleFactor`属性和包含它屏幕的比例相匹配，因此将其关联的帧缓冲区配置以渲染的全分辨率展现。 更多关于UIKit支持的高分辨率展现的内容见[Supporting High-Resolution Screens In Views](https://developer.apple.com/library/archive/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/SupportingHiResScreensInViews/SupportingHiResScreensInViews.html#//apple_ref/doc/uid/TP40010156-CH15)。

如果使用Core Animation层级出现OpenGL ES的内容，它默认的缩放因子为1.0。为了在Retina显示器的全分辨率下绘图，我们应该修改`CAEAGLayer`对象的缩放因子以匹配屏幕的缩放因子。

当支持高分辨率显示器的设备，我们应该相应地跳转应用程序的模型和纹理资源。当高分辨率的设备下运行，我们可能想要渲染更详细的模型和纹理来渲染更好的图像，相反，在标准分辨率设备下，我们可能使用更小的模型和纹理。

> 许多OpenGL ES API调用都用屏幕像素表示维度，如果使用的缩放因子大于1.0，那么在使用`glScissor`、`glBlitFramebuffer`、`glLineWidth`、`glPointSize`函数或`gl_PointSize`着色器变量时，我们需要相应调整维度。

决定如何支持高分辨率显示器的一个重要因素是性能。Retina显示器上的翻倍比例因子使像素的数量变为四倍，这使得GPU处理的碎片数量变为原来的四倍。如果应用程序对每个片段执行多个计算，像素的增加可能会导致帧率降低。如果我们发现应用程序在较高的比例因子下运行速度明显较慢，考虑以下选项之一：
- 使用性能调优指南优化片段着色器的性能，见[Tuning Your OpenGL ES App](tuning-your-openGL-es-app)；
- 在片段着色器实现一个更简单的算法，这样做可以降低单个像素的质量，以更高的分辨率渲染整个图像；
- 使用在1.0到屏幕比例因子之间的比例因子，比例因子为1.5比1.0提供更高的质量，但需要填充填充的像素比比例为2.0的图像更少；
- 使用较低精度格式的GLKView对象的`drawableColorFormat`和`drawableDepthFormat`属性。这样做可以减少操作底层渲染缓冲区所需的内存带宽；
- 使用较低的比例因子，启用多重采样。另一个优点是，多重采样同样可以在不知道高分辨率显示器的设备上提供更高的质量。要为GLKView对象启用多重采样，要更改它的`drawableMultisample`属性值。如果不是渲染到GLKit视图上，则必须手动甚至多重采样缓冲区并在最终图形呈现前解析他们，见[使用多重采样提高图片质量](#使用多重采样提高图片质量)。

## 支持多种屏幕旋转

与任意应用程序一样，OpenGL ES应用程序应该支持与其内容相适应的用户界面方向。可以在应用程序的信息属性列表定义支持的屏幕方向或者拥有OpenGL ES内容的控制器使用`supportedInterfaceOrientations`方法。

默认情况下，`GLKViewController`和`GLKView`类自动处理屏幕转向：当用户渲染设备到支持的方向，系统将产生动画旋转屏幕并更改视图控制器视图的大小，当大小改变时，`GLKView`对象相应调整帧缓冲区和绘图窗口的大小。如果需要响应这个更改，在`GLKViewController`的子类中实现`viewWillLayoutSubviews`或`viewDidLayoutSubviews`方法，如果使用自定义的`GLKView`子类则实现`layoutSubviews`。

如果使用Core Animation层级绘制OpenGL ES的内容，应用程序也应该包含一个视图控制器来管理用户界面方向。

## 在外部显示器上显示OpenGL ES内容

一个iOS设备可以被连接到外部显示器，外部显示器的分辨率及其内容缩放因子可能与主屏幕的不同，渲染帧的代码应该调整以匹配。

在外部显示器绘图的过程基本和主屏幕上的相同：
1. 按照[Multiple Display Programming Guide for iOS](https://developer.apple.com/library/archive/documentation/WindowsViews/Conceptual/WindowAndScreenGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40012555)的步骤在外部显示器创建一个窗口。
2. 为渲染策略向窗口添加何时的视图或控制器；
    - 如果使用GLKit渲染，设置`GLKViewController`和`GLKView`（或自定义子类）的实例并用`rootViewController`属性添加到窗口；
    - 如果渲染到Core Animation层级，添加包含层级的视图作为窗口的子视图。使用动画循环来渲染，通过检索窗口的`screen`属性并调用`dispalyLinkWithTarget:slector:`方法创建一个外部显示器优化的显示链接对象。

# OpenGL ES Design Guidelines

现在我们已经掌握了在iOS应用中使用OpenGL ES的基础，使用这章下的信息帮助我们设计更优性能的渲染引擎。这一章节介绍了渲染器设计的关键概念；后面的内容将用特定的最佳实践和性能技术对这些信息扩展。

## 如何可视化OpenGL ES

本节描述了可视化OpenGL ES设计的两个方面：作为客户端-服务器架构和作为管线。这两方面在计划和评估应用程序的体系结构都非常有用。

### OpenGL ES作为客户端-服务器架构

应用程序与OpenGL ES客户端通信状态改变、纹理和顶点数据、渲染命令。客户端将这些数据转换成图形硬件理解的格式，并发送到GPU，这些进程会增加应用程序图形性能的开销。

![OpenGL ES client-server architecture](/img/article/20190519/9.png)

要获得出色的性能需要小心管理这些开销。一个优秀设计的应用程序减少对OpenGL ES的调用频率，使用硬件合适的数据格式来降低转换成本，并小心管理和OpenGL ES之间的数据流。

### OpenGL ES作为图形管线

应用程序配置图形管线，然后执行绘图命令将顶点数据发送到管线中。管线的后续阶段运行顶点着色器处理顶点数据，将顶点数据组装成基本类型，将基本类型光栅化为片段，运行片段着色器来计算每个片段的颜色和深度值，并将片段混合到帧缓冲区显示。

![OpenGL ES graphics pipline](/img/article/20190519/10.png)

使用管线作为一个心理模型来识别应用程序执行什么工作来生成一个新帧。我们的渲染器设计包含编写着色器程序处理管线的顶点和片段阶段，组织提供给这些程序的顶点和纹理数据，以及配置OpenGL ES状态机来驱动管线的固定功能的阶段。

图形管线的各个阶段可以同时计算它们的结果--例如，应用程序可能准备了新的基础类型而图形硬件的各个部分还在对之前提交的几何图形执行顶点和片段计算，但是，后期阶段取决于先前阶段的输出，所以如果任何管道阶段执行过多任务或运行太慢，其他管线阶段将处于空闲状态知道最慢的完成工作。一个优秀设计的应用程序应可以根据图形硬件功能平衡每个管道阶段的工作。

> 所以当我们调整应用程序性能时，第一步通常确定它处于哪个位置，以及会遇到哪些瓶颈。

## OpenGL ES版本和渲染器架构

iOS支持3中OpenGL ES版本。最新版本提供更多的灵活，允许我们自主实现渲染包含高质量视觉效果而不影响性能的算法。

### OpenGL ES 3.0

OpenGL ES 3.0是iOS 7之后的新版本。应用程序可以使用OpenGL ES 3.0中引入的特性来实现高级的图形编程技术（以前只能用于桌面级硬件和游戏控制台）从而获得更快的图形性能和引人注目的视觉效果。更多见[OpenGL ES API Registry](http://www.khronos.org/registry/OpenGL/index_es.php)。
[OpenGL ES 3.0](https://developer.apple.com/library/archive/documentation/3DDrawing/Conceptual/OpenGLES_ProgrammingGuide/OpenGLESApplicationDesign/OpenGLESApplicationDesign.html#//apple_ref/doc/uid/TP40008793-CH6-SW13)
### OpenGL ES 2.0
### OpenGL ES 1.0

## 设计高性能的OpenGL ES应用程序

总结下，一个优秀设计的OpenGL ES应用程序需要：
- 利用OpenGL ES管线中的并行性
- 管理应用程序和图形硬件之间的数据流

下图展示了一个使用OpenGL ES执行动画显示的应用程序流程：

![App model for managing resources](/img/article/20190519/11.png)

当应用程序启动时，要做的第一件事是初始化不打算在生命周期中更改的资源。理想情况下，应用程序将这些资源封装到OpenGL ES的对象中，目标是创建任何在应用程序运行期间保持不变的对象（甚至是应用程序生命周期的一部分，如游戏关卡的持续时间），以增加的初始化时间交换更好的渲染性能。复杂的命令或状态改变应该替换为能与单个函数调用一起使用的OpenGLES对象，比如，配置固定函数管线可能需要几十个函数调用。相反，在初始化阶段编译一个图形着色器，并在运行时通过一个函数调用切换到它。创建或者修改开销较大的OpenGL ES对象应该总是作为静态对象创建。

渲染循环处理我们打算渲染给OpenGL ES上下文的所有项，然后将结果呈现给显示器。在动画场景中，一些数据每一帧都在更新，在上图所示的内部渲染循环中，应用程序在更新渲染资源（在进程中创建或修改OpenGL ES对象）和提交这些资源的绘图命令之间切换。这个内部循环的目标是平衡工作负载让CPU和GPU并行工作，防止应用程序和OpenGL ES同时使用相同的资源。在iOS中，如果不在帧的起始或结束时执行修改OpenGL ES对象，那么修改的代价会非常高。

这个内部循环的一个重要目标是避免将数据从OpenGL ES拷贝回应用程序，将结果从GPU拷贝到CPU会非常慢。如果拷贝的数据同样用于后续渲染当前帧过程的一部分（如上图的渲染循环中展示的），应用程序会被阻塞知道所有之前提交的绘图命令完成。

在应用程序提交帧所需要的所有绘图命令，然后将结果呈现到屏幕上。非交互式哟哟欧诺个程序会将最终图片拷贝到应用程序内存中进行下一步处理。

最终，当应用程序准备退出或结束主要工作，将释放OpenGL ES对象获取更多的可用资源。

总结本设计的重要特点：
- 在任何可能情况下创建静态资源；
- 内部渲染循环在修改动态资源和提交渲染命令之间切换，尝试避免在帧的起始或结束之外的时间修改动态资源；
- 避免将中间渲染结果读取回应用程序；

本节后面的部分提供了游泳的变成技术来实现这个渲染循环的特性。

### 避免同步和Flush操作

OpenGL ES规范不要求实现立即执行命令。通常情况，命令被排队到命令缓冲区，后面由硬件执行。OpenGL ES会等到应用程序许多命令在队列中后发送到硬件--批处理更加有效，但是，一些OpenGL ES函数必须立即刷新命令缓冲区，其他函数不仅刷新命令缓冲区，还会阻塞直到之前的命令完成才返回对应用和程序的控制。只有在必要时才使用刷新和同步命令，过度使用刷新或同步命令会导致应用程序等待硬件结束渲染时停止。

下面这些情况需要OpenGL ES将命令缓冲区提交给硬件执行：
- `glFlush`函数将命令缓冲区提交到图形硬件，它会阻塞到命令被提交到硬件但不用等到命令执行结束；
- `glFinish`函数刷新命令缓冲区，然后等待所有之前提交的命令在图像硬件上执行结束；
- 检索帧缓冲区内容的函数（如`glReadPixels`）同样要等待提交的命令完成；
- 命令缓冲区已满；

#### 有效利用glFlush

在一些桌面OpenGL的实现中，定期调用`glFlush`函数可以有效平衡CPU和GPU的工作，但在iOS中并非如此。iOS图像硬件实现的延迟算法依赖于一次性缓存所有场景中的顶点数据，因此可以对其进行最有处理，去除隐藏表面。通常，OpenGL ES应用程序只有两种情况下调用`glFlush`或`glFinish`函数：
- 应用程序在进入后台时需要刷新命令缓冲区，因为应用程序在后台时执行OpenGL ES命令会导致iOS崩溃；
- 如果应用程序在多个上下文共享OpenGL ES对象（如顶点缓存或纹理），我们需要调用`glFlush`函数同步使用这些资源。例如，在一个上下文加载顶点数据之后调用`glFlush`函数，以确保内容准备被其他上下文检索。当与其他iOS APIs（如Core Image）共用OpenGL ES对象时也可以使用这个建议；

#### 避免查询OpenGL ES状态

调用`glGet*()`，包括`glGetError()`，可能需要OpenGL ES在检索任何状态变量之前执行之前的命令。这种同步会强迫图形硬件和CPU同步运行，减少了并行性的机会。为了避免这种情况，维护自己需要查询任意状态的副本，直接访问它而不是调用OpenGL ES。

当错误发生时，OpenGL ES设置一个错误标识。这些错误和其他错误出现在的Xcode或OpenGL ES分析器的帧调试器中。应该使用这些工具而不是使用`glGetError`函数，如果频繁调用会降低性能。其他查询（例如`glCheckFramebufferStatus()`，`glGetProgramInfoLog()`，`glValidateProgram()`）通常也只在开发和调试中有用。

### 使用OpenGL ES管理资源

许多OpenGL数据可以直接存储到OpenGL ES渲染上下文和与之关联的共享组中。OpenGL ES的实现可以自由将数据转换为适合图形硬件的格式，这可以显著提高性能，特别对于不经常更改的数据。应用程序还可以向OpenGL ES提供如何使用数据的提示，OpenGL ES的实现可以更有效使用这些提示处理数据。例如，静态数据可能被放置在图像处理器可以轻松获取的内存中，甚至可以放置在专用的图形内存中。

### 使用双重缓冲避免资源冲突

当应用程序和OpenGL ES同时访问一个OpenGL ES对象时会发生资源冲突。当一个参与者尝试修改对象而另一个参与者正在使用，他们可能会阻塞直到该对象不再被使用。一旦他们开始修改对象，其他参与者可能在修改完成之前都无法使用。或者，OpenGL ES可以隐式复制对象，以便两个参与者都可以继续执行命令。任何一个选项都是安全的，但是每个都有可能成为应用程序的瓶颈。下图展示了这个问题，在这个例子中，只有一个纹理对象，OpenGL ES和应用程序都想使用它，当应用程序尝试修改纹理，它必须等待之前提交的绘图命令完成--CPU和GPU同步。

![Single-buffered texture data](/img/article/20190519/12.png)

为了解决这个问题，应用程序可以在修改和绘制该对象之间执行额外的工作。但是，如果应用程序没有额外的工作可以执行，它应该显示创建两个大小相同的对象；一个参与者读取对象，另一个参与者修改另一个。下图展示了双缓冲方法，当GPU处理一个纹理时，CPU修改另一个。在初始化启动之后，CPU和GPU都不会处于空闲状态。虽然显示了纹理，这个解决方案几乎适用于任何类型的OpenGL ES对象。

![Double-buffered texture data](/img/article/20190519/13.png)

对于大多数应用程序双缓冲区已经足够，但它要求两个参与者几乎同时完成处理命令。为了避免阻塞，可以添加更多的缓冲区；这实现了传统的生产者-消费者模型。如果生产者在消费者之前完成处理命令，它将接受一个空闲缓冲区并继续处理命令。在这种情况下，只有消费者严重落后，生产者才会停止生产。

双缓冲区和三缓冲区需要消耗额外的内存以防止管道停滞，额外使用内存可能会对应用程序的其他部分造成压力。在iOS设备中，内存稀缺；所以设计必须和其他应用程序优化平衡以使用更多的内存。

### 注意OpenGL ES的状态

OpenGL ES的实现维护了一份复杂的状态数据，包括使用`glEnable`或`glDisable`函数设置的开关、当前着色器程序以及其统一变量、当前绑定的纹理图元、当前绑定的顶点缓存及其启用的顶点属性。硬件有一个当前状态，它被缓存并懒加载。切换状态是昂贵的，所以最好减少状态切换。

不要设置以及设置的状态。一旦启用某个特性就不用再启用。比如，如果多次调用相同参数的`glUniform`函数，OpenGL ES可能不会检查已经设置的相同的状态，它只是简单的更新状态值，即使值相同。

避免使用专用的设置或关闭过程设置不必要的状态，而不是将这类调用放入绘图循环。设置和关闭过程对于开启关闭实现特定视觉效果的特性很有用--例如，当绘制线框轮廓线围绕有纹理的多边形时。

#### 使用OpenGL ES对象封装状态

要减少状态更改，创建一个对象，将多个OpenGL ES状态更改收集到一个对象，该对象可以使用一个函数绑定。例如，顶点数组对象将多个顶点属性的配置存储到一个对象中。见[使用顶点数组对象合并顶点数组状态更改](使用顶点数组对象合并顶点数组状态更改)。

#### 组织绘图调用最小化状态更改

OpenGL ES的状态更改的效果没有立即生成。相反，当我们发出绘图指令时，OpenGL ES将执行绘制状态值所需的工作。通过减少状态更改，我们可以减少CPU用于配置图形管线的时间。例如，在应用程序中保留一个状态向量，并且只有在绘制调用之间的状态发生时才设置相应的OpenGL ES状态。另一个有用的算法是状态排序--跟踪我们需要做的绘图操作和每个操作所需的状态更改量，然后对它们排序，以便连续使用相同的状态执行操作。

OpenGL ES的iOS实现可以缓存它在状态之间高效切换所需的一些配置数据，但每个唯一状态集的初始配置需要更长的时间。为了保持一致的性能，我们可以在配置过程中“预热”计划使用的每个状态：
- 启用计划使用的状态配置或着色器；
- 使用状态配置绘制少量顶点；
- 刷新OpenGL ES上下文，以便在预热阶段不显示绘图；

# Best Practices for Working with Vertex Data

要使用OpenGL ES渲染帧，应用程序需要配置图形管线并提交要绘制的图形图元。在一些应用中，所有的图元都是使用同样的管线配置绘制；其他应用可能使用不同的技术渲染不同的帧元素。但无论应用程序使用的哪个图元，管线如何配置，应用程序都需要向OpenGL ES提供顶点。本章提供了一个关于顶点的课程并就如何有效处理顶点数据提供了针对性的建议。

顶点由一个或多个属性组成，例如位置、颜色、法线或纹理坐标。OpenGL ES 2.0或3.0应用程序可以自由定义自己的属性；顶点数据的每个属性对应于作为顶点着色器的属性变量。OpenGL ES 1.1的应用程序使用固定管线定义好的属性。

将一个属性定义为由一到四个组件构成的向量，属性中的所有组件共享一个公共数据类型。例如，一个颜色可能被定义为四个`GLubyte`组件（r,g,b,a）。当一个属性被加载到着色器变量中时，OpenGL ES使用默认值填充所有应用程序未提供的组件数据。最后一个组件被填充为1，其他未指定组件为0，如下图所示：

![Conversion of attribute data to shader variables](/img/article/20190519/14.png)

应用程序可能将一个属性配置为常量，这意味着对于作为绘图命令部分提交的所有顶点使用相同的值，又意味着每个顶点都是该属性的值。当应用程序调用OpenGL ES的函数绘制一组顶点时，顶点数据被拷贝到图形硬件。处理顶点数据的图形硬件，在着色器处理每个顶点，组装图元并将它们光栅化到帧缓冲区。OpenGL ES的一个优点在于将提交到OpenGL ES顶点数据的一组函数标准化，移除OpenGL ES提供的陈旧和低效的机制。

应用程序必须提交大量图元来渲染一帧，需要小心管理它们的顶点数据和如何提交到OpenGL ES。本章所述的做法可以概括为以下几个基本原则：
- 减少顶点数据的大小；
- 减少OpenGL ES将顶点数据传输到图形硬件之前的预处理；
- 减少拷贝顶点数据到图形硬件花费的时间；
- 减少对每个顶点的计算；

## 简化模型

基于iOS设备的图形硬件很强大，但它显示的图片一般很小。我们不需要非常复杂的模型在iOS上呈现引人注目的图形。减少用于绘制模型的顶点数量可以直接减少顶点的数据和对顶点数据执行的计算。

我们可以使用以下技术降低模型的复杂度：

- 在不同细节级别提供不同版本的模型，并在运行时基于到摄像机的距离和显示的尺寸渲染合适的模型；
- 使用纹理来消除对某些顶点信息的需要。例如，凹凸贴图可以用来在不添加更多顶点数据的情况下向模型添加细节；
- 一些模型添加顶点改进光照细节或渲染质量。这通常是在计算每个顶点的值并在光栅化阶段对三角形进行插值完成的。例如，如果我们将聚光灯指向三角形的中心，它的效果可能被忽略，因为聚光灯醉了的部分不是指向一个顶点。通过添加顶点，我们可以额外提供插值点，代价是增加顶点数据的大小和模型上执行的计算。如果不想要添加额外的顶点，考虑将计算移动到管线的片段阶段：
    + 如果应用程序使用OpenGL ES 2.0或之后的版本，应用程序在顶点着色器执行计算并将其分配给一个可变变量。变化的值由图形硬件插值并作为输入传递到片段着色器。相反，将计算的输入分配给变量并在片段着色器执行计算。这样做会将执行计算的成本从每个顶点的成本到每个片段的成本，从而减少顶点阶段的压力并增加管线片段阶段的压力。当应用程序在顶点处理时被阻塞，这样做，计算是相对廉价的并可以通过更改显著减少顶点数。
    + 如果应用程序使用OpenGL ES 1.1，我们可以使用DOT3照明来执行各个片段的光照。我们可以通过添加凹凸贴图纹理来保存正常信息，并使用`GL_DOT3_RGB`模式的纹理组合操作应用凹凸贴图。

## 避免在属性数组中存储常量

如果模型中包含在这个模型中使用保持不变的数据的属性，则不要为每个顶点复制该数据。OpenGL ES 2.0和3.0应用程序可以设置一个常量顶点属性或使用一个统一的着色器值来保存该值。OpenGL ES 1.1应用程序应该使用每个顶点的属性函数，如`glColor4ub`或`glTexCoord2f`。

> 待更新...

# Reference

> [About OpenGL ES](https://developer.apple.com/library/archive/documentation/3DDrawing/Conceptual/OpenGLES_ProgrammingGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008793-CH1-SW1)