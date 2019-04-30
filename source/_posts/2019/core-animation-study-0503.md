---
title: "图形渲染与动画基础 -- Core Animation"
catalog: true
toc_nav_num: true
date: 2019-05-03 15:30:10
subtitle: "About Core Animation"
header-img: "/img/article_header/article_header.png"
busuanzi: true
tags:
- iOS

---

# Introduction

Core Animation是iOS与OS X平台上的图形渲染和动画基础设施，可以在应用中使视图和其他可见元素产生动画。使用Core Animation，绘制动画的每一帧所需的大部分工作都已经实现。我们所需要做的是配置少量动画参数同时启动动画，Core Animation会完成其余部分，将大部分实际绘图工作交给图形硬件实现加速渲染，这种自动图形加速产生高帧率平滑的动画而不会负担CPU和使应用卡顿。
当我们在写iOS代码时，不管是否知道我们都在使用Core Animation，如果是OS X，则可以非常轻松的利用Core Animation。Core Animation位于AppKit和UIKit下，被紧密集成到Cocoa和Cocoa Touch的视图流中。当然，Core Animation也有一些应用视图暴露的扩展功能接口，可以让我们更细致控制应用程序的动画。

![Core Animation](/img/article/20190503/1.png)

# Core Animation Basics

Core Animation提供一个视图和其他可见元素动画的通用系统，它不是View的替代品，相反，他是一种与视图结合，提供更好性能、为视图提供动画的技术。它将视图的内容缓存到可以直接被图形硬件操作的位图中，在一些情况下，这个缓存操作可能会让我们重新思考我们如何呈现和管理我们的应用内容，但是大多数情况我们不需要知道。除了缓存视图内容，Core Animation同样定义了一种方式，指定任意可视内容，结合视图上的内容，让他和其他任意东西一起动起来。

## Layers提供绘图与动画的基础

Layer对象是三维空间中的二维表面，是Core Animation所做事情的核心。与View一样的是，层级管理其表面的几何、内容、视觉属性信息。与View不同的是，层级不定义他们自己的外观。一个层级仅仅管理位图周围的状态信息，位图本身可以是视图绘制本身的结果，也可以是指定的固定图形的结果。因此，应用中的主要层级被被认为是模型对象，因为他们主要管理数据。这个概念很重要，因为他们影响到动画表现。

### 层级绘制模型

多数层级不是真的绘图，相反，一个Layer捕获内容然后缓存到位图，有时也被称为**后备存储器**。当我们修改一个层级的属性，我们所做的只是修改关联层级的状态信息。当一项改变触发动画，Core Animation将层级的位图和状态信息传递给图形硬件，图形硬件使用新的信息渲染位图。在硬件中操作位图比在软件中可以更快的生成动画。

![How Core Animation draws content](/img/article/20190503/2.png)

因为它操作静态位图，图层绘图与更传统的视图绘图有很大的不同。使用视图绘图，视图本身的修改经常会出发**drawRect:**方法的调用，使用新参数重新绘制内容，但这种方式绘图代价很高因为是在主线程使用CPU完成的。Core Animation尽可能通过在硬件中操作缓存位图来达到相同或类似的效果，以此避免这种开销。

尽管Core Animation尽可能地使用缓存内容，但应用任然必须提供初始内容并不时更新。

### 层级动画

层对象的数据和状态信息与改层内容在屏幕上的可视化表现相互解耦，这种解耦为Core Animation提供的一种方法，使它自我干预，并创建新旧状态的动画。例如，修改一个层级的位置属性会使Core Animation移动层级从当前位置到新状态的位置。

![Examples of animations I can perform on layers](/img/article/20190503/3.png)

在动画制作过程中，Core Animation会在硬件上逐帧完成绘图。我们所要做的就是指定动画的起点与终点，让Core Animation完成剩下的工作，如有必要，可以同样指定自定义的时间信息和动画参数，如果不提供，Core Animation有合适的默认值。

## 层级对象定义自身的几何结构

一个层级的工作就是管理内容的视觉几何结构，视觉几何结构由内容的bounds，屏幕上的位置信息，是否旋转、缩放、位移的信息构成。和View一样，层级有frame和bounds来定位位置，同样，层级也有View没有的属性，比如锚点坐标。指定层级几何的某些方面的方式也和视图指定信息的方式不同。

### Layers使用两种坐标系统

Layers同时使用point-based坐标系和unit坐标系来指定内容的位置。使用哪种坐标系统根据传递信息的类型。当指定直接映射到屏幕坐标系的值或者必须相对于另一个层级的指定，就需要用point-based坐标系，比如一个层级的位置属性。当值与屏幕坐标系无关时使用unit坐标系，比如层级的锚点属性，指定相对于层级自身边界的点。

point-based坐标系最常见的使用就是指定曾记得大小（bounds）和位置（position）属性，当然也有frame属性，这个属性是由boudns和position属性中派生出来的值，使用频率较低。bounds和frame的坐标系方向：

![The default layer geometries for iOS and OS X](/img/article/20190503/4.png)

图中的position属性在层级的中点位置，这个属性是根据层级锚点属性中的值更改定义的几个属性之一。锚点是几个使用unit坐标系的属性之一，Core Animation使用unit坐标系表示在层级大小修改时可能受影响的属性，我们可以将unit坐标系看作是总值的百分比，每个unit坐标系中的值在0.0-1.0之间。

![The default unit coordinate systems for iOS and OS X](/img/article/20190503/5.png)

### 锚点坐标影响几何操作

一个层级的相关几何操作的发生与层级的锚点相关，我们可以使用锚点属性访问。当操作层级的位置或位移属性时，锚点的作用会更加可见。下图展示了锚点的改变对位置属性的改变：

![How the anchor point affects the layer's position property](/img/article/20190503/6.png)

下图是锚点对于旋转属性的影响：

![How the anchor point affects layer transformations](/img/article/20190503/7.png)

## 层级树反应出动画状态的不同方面

一个使用Core Animation的应用有三种层级对象：
- Objects in the model layer tree：应用中互动最多的层级，模型对象存储动画的目标值，当我们改变层级的属性时，就使用这个对象
- Objects in the presentation tree：包含运行动画中的值，虽然模型层级树包含动画的目标值，但是呈现层级树反应动画的当前值。我们不应该修改这个值
- Objects in the render tree：真正动画实现的层级，Core Animation的私有对象

每个层级对象被组织成已给层级结构，和应用中的视图一样。事实上，当为所有视图启用层级，每个树的层级结构与视图结构完全匹配。但是，一个应用可以添加额外的层级对象，层级和视图并不是相匹配的。在某些情况下，我们可能会这样这样操作来优化不需要视图而造成的开销。

![The layer trees for a window](/img/article/20190503/8.png)


## Layers与Views的关系

Layers不能代替Views--我们不能基于层级对象创建可视界面。Layers为Views提供基础，具体地说，Layers可以使视图内容的绘制与动画更加简单有效的同时保证高帧率。当然，Layers不能处理事件，绘制内容，参与响应链等等。

# Settign Up Layer Objects

Layer对象是Core Animation的核心，层级管理应用的可见内容和提供修改样式和可见外观的选项。

## 修改View的Layer对象

iOS默认情况下，View会自动创建CALayer类的实例，大多数情况下我们不需要其他类型的层积类。但是，Core Animation提供了不同的层级类，提供可能对我们有用的层级类。[不同的Layer](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/CoreAnimation_guide/SettingUpLayerObjects/SettingUpLayerObjects.html#//apple_ref/doc/uid/TP40004514-CH13-SW25)

修改View的Layer类型：
``` Swift
override var layer: CALayer {
    return CAShapeLayer()
}
```

## 提供Layer的Content

Layers是管理内容的数据对象，一个层级的内容包含我们想要展示可视数据的位图，有以下三种方式提供位图的内容：
- 直接给**contents**属性赋值（适用于层级内容基本不会改变的情况）
- 实现Layer的代理方法完成内容的绘制（适用于层级内容可能周期性改变并且可以有外部对象提供，比如View）
- 定义一个Layer的子类同时复写**drawing**方法来自己提供内容（适用于自定义Layer子类或者想要改变层级的基本会话操作）

### Content赋值图片

因为一个层级只是管理位图的容器，我们可以直接给**Contents**属性赋值图片。层级可以直接使用提供的图片，而不用拷贝一份图片，这个操作可以让图片在不同地方使用时节约内存。赋值的图片类型必须是**CGImageRef**类型，同时图片的分辨率要适配原生设备。允许的话，需要适当调整图片的**ContentsScale**属性。

### 通过代理提供Content

如果需要动态修改层级的Content，我们可以使用提供的代理。在显示时，层级会调用以下代理方法：
- **displayLayer:**方法，该实现负责创建位图并将其分配给**contents**属性
- **drawLayer:inContext:**方法，Core Animation创建一个位图，创建一个图形上下文来绘制位图，然后调用这个方法填充位图。这个方法就是在提供的图形上下文绘图。

代理必须实现**displayLayer:**和**drawLayer:inContext**两种方法之一，如果同时实现，只会调用**displayLayer**方法。

当应用程序是在想要显示的地方加载或创建位图时，复写**displayLayer:**方法更加合适：
``` objectivec
- (void)displayLayer:(CALayer *)theLayer {
    // Check the value of some state property
    if (self.displayYesImage) {
        // Display the Yes image
        theLayer.contents = [someHelperObject loadStateYesImage];
    }
    else {
        // Display the No image
        theLayer.contents = [someHelperObject loadStateNoImage];
    }
}
```

> PS：本来这边有个问题，但是后面苹果也自己概述了，对于UIKit下的视图（层支持），我们直接用默认的**drawRect**方法进行绘制内容。而对于OS X环境下，Layer与View不是直接相关联的，**drawRect**与**drawLayer**方法两者之间的区别。**drawRect**是View的视图渲染方法，**drawLayer**是Layer的代理方法，两者都可以绘图。目前的调用机制是这样：同时实现两个方法会只调用**drawLayer**里的实现；无法单独实现**drawLayer**方法，必须空实现**drawRect**方法；**drawLayer**方法也可以由**setNeedsDisplay**方法调用，同**drawRect**方法一样。两者用法的区别：是否需要获取视图在动画过程中的Layer属性，**drawLayer**方法中实现的内容可以实时获取Layer的属性值，**drawRect**方法只能获取Layer的结果值。[是否对仍待考究](https://stackoverflow.com/questions/4979192/ios-using-uiviews-drawrect-vs-its-layers-delegate-drawlayerincontext/36050120#36050120)

如果没有预先编码的图片或者对象来创建位图，可以使用**drawLayer**方法动态绘制内容：
``` objectivec
- (void)drawLayer:(CALayer *)theLayer inContext:(CGContextRef)theContext {
    CGMutablePathRef thePath = CGPathCreateMutable();
 
    CGPathMoveToPoint(thePath,NULL,15.0f,15.f);
    CGPathAddCurveToPoint(thePath,
                          NULL,
                          15.f,250.0f,
                          295.0f,250.0f,
                          295.0f,15.0f);
 
    CGContextBeginPath(theContext);
    CGContextAddPath(theContext, thePath);
 
    CGContextSetLineWidth(theContext, 5);
    CGContextStrokePath(theContext);
 
    // Release the path
    CFRelease(thePath);
}
```

对于带有自定义内容的层支持视图，我们应该继续复写视图方法来进行绘图。层支持的绘图自动使自己成为其层的委托，并实现委托所需的委托方法，我们不应该更改这个配置，所以，对于UIView我们还是实现**drawRect**方法来绘制内容.

### 通过子类提供Content

当实现一个自定义Layer类时，我们可以复写Layer类的绘图方法来进行任何绘图：
- 复写**display**方法，直接设置**contents**属性
- 复写**drawInContext:**方法，在提供的图形上下文绘制

### 调整Content

当我们设置一个图层的**contents**属性时，**contentsGravity**属性决定了如何操作该图片来适应边界。默认情况下，如果一张图片大于当前边界，图层对象会缩放图片以适应可用空间。如果图层边界的宽高比与图像的宽高比不同，就会导致图像的失真。
**contentsGravity**属性可以被分为以下两类：
- 基于位置的重力常量允许我们将图片固定在边缘或角落上，无需缩放图片
- 基于缩放的重力常量允许我们使用几种选项之一来拉伸图像，可以保持宽高比，也可以拉伸

Position-based gravity constants for layers:
![Position-based gravity constants for layers](/img/article/20190503/9.png)

Scaling-based gravity constants for layers:
![Scaling-based gravity constants for layers](/img/article/20190503/10.png)

## 调整Layer的视觉样式和外观

层级对象内置视觉装饰，比如边框、背景色，我们可以用来补充图层的内容。因为这些视觉装饰不需要我们做渲染，所以在某些情况下可以将层作为独立实体使用。我们需要做的就是设置属性，层级会处理必要的绘画，包括动画。

### Layers有自己的背景和边框

除了基于图像的内容外，层级可以显示填充背景和边框。背景色在层级内容图片背后渲染，边框在图片上面渲染，如果层级包含子层级，他们也在边框下面。因为背景色位于图像后面，所以这种颜色通过图像的任何透明部分发光。

如果设置不透明的背景色，考虑设置层级的**opaque**属性为YES，这样做可以在屏幕上合成图层时提高性能，就不需要使用图层的后备存储来管理alpha通道。但是，如果一个层的角半价不为零，则不能设为不透明。

### Layers支持倒角

我们可以为层级创建一个圆角矩形效果，倒角是一种视觉装饰，它掩盖了层级边界矩形的部分角，允许底层内容显示。因为它涉及到透明层蒙版，除非**maskToBounds**属性设置为YES，否则倒角不会影响图层图像。但是，倒角总是影响图层的背景颜色和边框的绘制。

### Layers支持内置阴影

CALayer类包含几种配置阴影效果的属性。阴影通过增加深度使其看起来像漂浮在底层内容之上，这是另一种类型的视觉装饰。我们可以控制阴影颜色，相对于图层内容的位置，不透明度和形状。

层级阴影的不透明度默认为0，这样可以有效地隐藏阴影。将不透明度修改为非零值会使Core Animation绘制阴影。因为阴影默认直接位于层级的最下方，所以需要更改阴影的偏移量才能看到它。但是务必记住，阴影的偏移量是使用图层的本机坐标系。

![Applying a shadow to a layer](/img/article/20190503/11.png)

当向图层添加阴影时，阴影是图层内容的一部分，但实际上扩展到图层的边界之外，所以，当图层启用**maskToBounds**属性时，阴影效果会被剪切到边缘。如果图层包含透明内容，这是会产生一个奇怪的效果，直接在图层下面的阴影部分仍然是可见的，但是超出部分不可见。所以，这是如果想要一个阴影，但也想用**maskTobounds**属性，我们可以使用两层层级，将该图层嵌入到相同大小应用阴影效果的图层中去。

# Reference

> [About Core Animation](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/CoreAnimation_guide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40004514-CH1-SW1)