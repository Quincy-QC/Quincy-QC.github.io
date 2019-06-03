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

因为它操作静态位图，图层绘图与更传统的视图绘图有很大的不同。使用视图绘图，视图本身的修改经常会出发`drawRect:`方法的调用，使用新参数重新绘制内容，但这种方式绘图代价很高因为是在主线程使用CPU完成的。Core Animation尽可能通过在硬件中操作缓存位图来达到相同或类似的效果，以此避免这种开销。

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
- 直接给`contents`属性赋值（适用于层级内容基本不会改变的情况）
- 实现Layer的代理方法完成内容的绘制（适用于层级内容可能周期性改变并且可以有外部对象提供，比如View）
- 定义一个Layer的子类同时复写`drawing`方法来自己提供内容（适用于自定义Layer子类或者想要改变层级的基本会话操作）

### Content赋值图片

因为一个层级只是管理位图的容器，我们可以直接给`Contents`属性赋值图片。层级可以直接使用提供的图片，而不用拷贝一份图片，这个操作可以让图片在不同地方使用时节约内存。赋值的图片类型必须是*CGImageRef*类型，同时图片的分辨率要适配原生设备。允许的话，需要适当调整图片的`ContentsScale`属性。

### 通过代理提供Content

如果需要动态修改层级的Content，我们可以使用提供的代理。在显示时，层级会调用以下代理方法：
- `displayLayer:`方法，该实现负责创建位图并将其分配给`contents`属性
- `drawLayer:inContext:`方法，Core Animation创建一个位图，创建一个图形上下文来绘制位图，然后调用这个方法填充位图。这个方法就是在提供的图形上下文绘图。

代理必须实现`displayLayer:`和`drawLayer:inContext`两种方法之一，如果同时实现，只会调用`displayLayer`方法。

当应用程序是在想要显示的地方加载或创建位图时，复写`displayLayer:`方法更加合适：
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

> PS：本来这边有个问题，但是后面苹果也自己概述了，对于UIKit下的视图（层支持），我们直接用默认的`drawRect`方法进行绘制内容。而对于OS X环境下，Layer与View不是直接相关联的，`drawRect`与`drawLayer`方法两者之间的区别。`drawRect`是View的视图渲染方法，`drawLayer`是Layer的代理方法，两者都可以绘图。目前的调用机制是这样：同时实现两个方法会只调用`drawLayer`里的实现；无法单独实现`drawLayer`方法，必须空实现`drawRect`方法；`drawLayer`方法也可以由`setNeedsDisplay`方法调用，同`drawRect`方法一样。两者用法的区别：是否需要获取视图在动画过程中的Layer属性，`drawLayer`方法中实现的内容可以实时获取Layer的属性值，`drawRect`方法只能获取Layer的结果值。[是否对仍待考究](https://stackoverflow.com/questions/4979192/ios-using-uiviews-drawrect-vs-its-layers-delegate-drawlayerincontext/36050120#36050120)

如果没有预先编码的图片或者对象来创建位图，可以使用`drawLayer`方法动态绘制内容：
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

对于带有自定义内容的层支持视图，我们应该继续复写视图方法来进行绘图。层支持的绘图自动使自己成为其层的委托，并实现委托所需的委托方法，我们不应该更改这个配置，所以，对于UIView我们还是实现`drawRect`方法来绘制内容.

### 通过子类提供Content

当实现一个自定义Layer类时，我们可以复写Layer类的绘图方法来进行任何绘图：
- 复写`display`方法，直接设置`contents`属性
- 复写`drawInContext:`方法，在提供的图形上下文绘制

### 调整Content

当我们设置一个图层的`contents`属性时，`contentsGravity`属性决定了如何操作该图片来适应边界。默认情况下，如果一张图片大于当前边界，图层对象会缩放图片以适应可用空间。如果图层边界的宽高比与图像的宽高比不同，就会导致图像的失真。
`contentsGravity`属性可以被分为以下两类：
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

如果设置不透明的背景色，考虑设置层级的`opaque`属性为YES，这样做可以在屏幕上合成图层时提高性能，就不需要使用图层的后备存储来管理alpha通道。但是，如果一个层的角半价不为零，则不能设为不透明。

### Layers支持倒角

我们可以为层级创建一个圆角矩形效果，倒角是一种视觉装饰，它掩盖了层级边界矩形的部分角，允许底层内容显示。因为它涉及到透明层蒙版，除非`maskToBounds`属性设置为YES，否则倒角不会影响图层图像。但是，倒角总是影响图层的背景颜色和边框的绘制。

### Layers支持内置阴影

CALayer类包含几种配置阴影效果的属性。阴影通过增加深度使其看起来像漂浮在底层内容之上，这是另一种类型的视觉装饰。我们可以控制阴影颜色，相对于图层内容的位置，不透明度和形状。

层级阴影的不透明度默认为0，这样可以有效地隐藏阴影。将不透明度修改为非零值会使Core Animation绘制阴影。因为阴影默认直接位于层级的最下方，所以需要更改阴影的偏移量才能看到它。但是务必记住，阴影的偏移量是使用图层的本机坐标系。

![Applying a shadow to a layer](/img/article/20190503/11.png)

当向图层添加阴影时，阴影是图层内容的一部分，但实际上扩展到图层的边界之外，所以，当图层启用`maskToBounds`属性时，阴影效果会被剪切到边缘。如果图层包含透明内容，这是会产生一个奇怪的效果，直接在图层下面的阴影部分仍然是可见的，但是超出部分不可见。所以，这是如果想要一个阴影，但也想用`maskTobounds`属性，我们可以使用两层层级，将该图层嵌入到相同大小应用阴影效果的图层中去。

# Animating Layer Content

Core Animation提供的基础设施可以让我们更加容易的创建层级动画，扩展到拥有层级的所有视图。

## 对图层属性的简单修改动画

简单动画包含隐式动画和显示动画。隐式动画使用默认的时间和动画属性，显式动画需要提供属性的配置。简单动画包含修改层级属性，在时间内完成动画，层级定义了很多影响可见外观的属性，改变其中一个属性会让其产生外观动画。

只需要更新层级属性就能触发隐式动画，层级的视觉外观不会立即改变，相反，Core Animation会根据属性变化作为触发器来创建调度一个或多个隐式动画执行。因此，像如下代码所示的改变会让Core Animation创建一个动画对象，并在下一个更新周期执行动画。

``` Swift
theLayer.opacity = 0.0
```

如果想要显式做出如上改变可以创建*CABasicAnimation*对象同时配置动画参数，可以在添加动画前设置动画的起始与结束值，改变持续时间，或者修改其他动画参数。下面代码展示了如何使用一个对话对象淡化一个层级，创建对象时，指定我们想要动画的属性路径，然后设置动画参数。使用`addAnimation:forKey:`方法调用动画。

``` Swift
let fadeAnimation = CABasicAnimation(keyPath: "opacity")
fadeAnimation.fromValue = 1.0
fadeAnimation.toValue = 0.0
fadeAnimation.duration = 1.0
layer.add(fadeAnimation, forKey: "opacity")

// Change the actual data value in the layer to the final layer
layer.opacity = 0.0
```

> Tip：当创建显式动画时，推荐使用`fromValue`属性。如果没有指定这个属性，Core Animation默认使用层级的当前值作为初始值。如果已经当前值等于初始值，可能不会得到想要的效果。

和隐式动画不同，显式动画不会改变层级的真正属性值，显式动画只提供动画。在动画的最后，Core Animation移除动画对象，在层级上使用当前值重新绘制。如果我们想要显式动画是永久更改，还必须更新层级的属性。

显式和隐式动画在当前runloop结束后开始执行，当前线程必须应用runloop才能让动画执行。如果层级修改多项属性或添加多个动画，所有的动画会在同时执行。

## 使用关键帧动画修改层级属性

基于属性的动画将属性从初始值更改为结束值，*CAKeyframeAnimation*对象允许我们通过一组目标值进行动画处理，其方式可能是线性的，也可能不是线性的。关键帧动画包含一组目标数据和每个值对应的时间组成。在最简单的配置中，使用指定值和时间。对于层位置的改变，我们还可以使用路径作为改变。动画对象获取指定关键帧，并通过给定时间段内从一个值穿插下一个值来构建动画。

``` Swift
let path = CGMutablePath()
path.move(to: CGPoint(x: 0, y: 0))
path.addCurve(to: CGPoint(x: 300, y: 600), control1: CGPoint(x: 200, y: 50), control2: CGPoint(x: 400, y: 350))

let animation = CAKeyframeAnimation(keyPath: "position")
animation.path = path
animation.duration = 2.0
myView.layer.add(animation, forKey: "position")
```

### 指定关键帧数值

关键帧数值在动画中是重要组成部分，这些值定义了动画在执行过程中的行为，指定关键帧的主要方式是将数组指定为包含*CGPoint*数据类型的值，当然也可以用*CGPathRef*代替。当指定一组数值，根据属性需要的数据类型提供数据，可以直接添加一些对象，但是一些对象需要转换成id类型，并且所有标量类型或结构体必须由对象包装：
- 对于*CGRect*属性，将每个矩形转换成*NSValue*对象
- 对于层级的转换属性，将*CATransform3D*矩阵转换成*NSValue*对象。
- 对于*borderColo*属性，每个*CGColorRef*数据类型转换成*id*类型，然后添加到数组中
- 对于*CGFloat*值的属性，将值转换为*NSNumber*对象添加到数组中
- 对于层级的*Contents*属性，指定一组*CGImageRef*类型的数据

对于*CGPoint*类型数据的属性，可以创建*NSValue*对象的数组或者创建*CGPathRef*对象来指定路径。当我们指定一组点，关键帧动画会在每个连续的点之间画一条线并沿着这条路动画。当我们指定一个*CGPathRef*对象，动画将从路径的起始点开始，并遵循其轮廓，包括沿着任何曲面。

### 指定关键帧动画的时间点

关键帧动画的时间和节奏比基本动画更复杂，我们可以使用以下属性控制它：
- *calculationMode*属性定义了用于计算动画计时的算法，这个属性值影响其他与时间相关属性的使用。
    1. 线性和立方动画 -- 当*calculationMode*属性设置为*kCAAnimationLinear*或者*kCAAnimationCubic`时的动画 -- 通过提供的时间信息生成动画，这个模式可以让我们最大程度控制动画时间
    2. 定步动画 -- 当*calculationMode*属性设置为*kCAAnimationPaced*或者*kCAAnimationCubicPaced*时的动画 -- 不依赖*keyTimes*或*timingFunctions*属性提供的外部计时值，相反，计时值是隐式的，以提供恒定速度的动画
    3. 离散动画 -- 当*calculationMode*属性设置为*kCAAnimationDiscrete*时的动画 -- 将动画属性从一个关键帧值直接跳转到另一个，而不需要任何中间值。这种计算模式使用*keyTimes*属性中的值，但忽略*timeingFunctions*属性
- *keyTimes*属性指定应用每个关键帧值的时间标记，仅当计算模式为*kCAAnimationLinear*，*kCAAnimationDiscrete*, *kCAAnimationCubic*时使用，他不是用于定步动画
- *timingFunctions*属性指定每个关键帧段使用的计时曲线

## 停止正在运行的显式动画

动画正常运行到结束，但我们可以使用以下技术提前停止动画：
- 从层级移除一个单独的动画对象，可以调用`removeAnimationForKey:`方法。这个方法使用的键是`addAnimation:forKey`方法传递进去的标识符，不可为空
- 从层级移除所有动画，可以调用`removeAllAnimations`方法。这个方法立即移除所有动画同时使用原始状态重绘层级

## 同时运行多个动画

如果想要在层级同时运行多个动画，我们可以使用*CAAnimationGroup*对象整合他们。使用一个组对象简化了对多个动画的管理，应用于组的时间和持续时间将腐败当个动画中的相同值。

``` Swift
let path = CGMutablePath()
path.move(to: CGPoint(x: 0, y: 0))
path.addCurve(to: CGPoint(x: 300, y: 600), control1: CGPoint(x: 200, y: 50), control2: CGPoint(x: 400, y: 350))

let positionAnimation = CAKeyframeAnimation(keyPath: "position")
positionAnimation.path = path

let colorAnimation = CAKeyframeAnimation(keyPath: "backgroundColor")
let colorValues = [UIColor.red.cgColor, UIColor.yellow.cgColor, UIColor.cyan.cgColor]
colorAnimation.values = colorValues
colorAnimation.calculationMode = .paced

let group = CAAnimationGroup()
group.animations = [positionAnimation, colorAnimation]
group.duration = 5.0
myView.layer.add(group, forKey: "group")
```

使用组动画的进阶方法是使用*CATransaction*对象，后面讨论。

## 监测动画的结束

Core Animation提供动画起始与结束的监测，这些通知是执行与动画相关任务的好时机。有两种方式监听动画的状态：
- 对于*CATransaction*的类方法，设置`setCompletionBlock:`方法，在动画结束后会执行回调
- 对于*CAAnimation*对象，可以实现他的代理方法`animationDidStart:`和`animationDidStop:finished:`

如果想要一个动画接着一个动画执行，不要使用动画监听。可以使用*beginTime*属性开始一个动画，设置另一个动画的开始时间是这个动画的结束时间。

## 如何在Layer-Backed的视图上动画

如果一个层级属于视图，创建动画推荐的方法是使用UIKit提供的基于视图的动画接口。有一些方法可以直接使用Core Animation接口对层进行动画处理。

对于*UIView*类，总有一个Layer与之对应，类本身直接从层级派生出它的大部分数据，因此，对层级做出的更改也会自动由视图显示出来，这就意味着我们可以同时使用Core Animation或者UIView提供的接口来动画。

如果想要使用Core Animation类来初始化动画，必须在基于视图的动画块中调用Core Animation的方法，*UIView*默认禁止图层动画，但在动画块中重新启动他们。一次，在动画块之外所有的任何更改都不是动画。(针对隐式动画，允许在动画块中启动)

``` Swift
UIView.animate(withDuration: 2.0) {
    self.view.layer.opacity = 0.0
}
```

# Advanced Animation Tricks

有很多种方法可以配置基于属性或者关键帧的动画，如果想要同时或顺序运行多种动画时可以使用更高级的方式来同步这些动画的时间将他们链接在一起，我们可以使用其他类型的动画对象来创建视觉转换和一些其他有趣的效果。

## 支持对层级视觉改变的转换动画

如题所示，一个转换动画对象为层级创建视觉动画，转换对象的常见用法是协调一个层级显示和另一个层级消失的动画。不同于修改层级属性的动画，一个转换动画操作层级的缓存图片创建视觉效果，仅通过修改属性是难以做到的。标准类型的转换允许我们只想显示、推送、移动、渐入渐出等动画。

要执行转换动画，需要创建*CATransition*对象并将其添加到与之转换相关的层级上，我们可以使用转换动画指定要执行的转换类型以及动画的起点终点。我们也不需要使用整个转换动画，转换对象可以让我们指定动画时使用的开始与结束进度值，这些值让我们在中点位置开始或者结束动画。

下面代码在两个视图中创建推送转换动画，myView1与myView2在父视图的同一个位置但只有一个视图可见，推送转换会导致一个视图从当前位置往左偏移消失而另一个视图从右边开始偏移到位置显示。更新*isHidden*属性用于视图在动画后的正确显示。

``` Swift
let transition = CATransition()
transition.startProgress = 0.0
transition.endProgress = 1.0
transition.type = .push
transition.subtype = .fromRight
transition.duration = 2.0

myView1.layer.add(transition, forKey: "transition")
myView2.layer.add(transition, forKey: "transition")

myView1.isHidden = false
myView2.isHidden = true
```

这里可以使用同一种转换动画，也可以根据需要使用不同的转换动画。

## 自定义动画的时间

时间控制是动画的重要组成部分，Core Animation可以通过*CAMediaTiming*代理的方法和属性来指定动画的时间信息。*CAAnimation*和*CALayer*都已经遵循了这个代理，但是封装这些动画的隐式转换对象通常提供了优先级默认的时间信息。

在考虑时间与动画时，理解层级与动画的合作关系是关键的，每个层级有自己的用于管理动画计时的本地时间。通常情况下，两个不同层级的本地时间是相近的，我们可以为每个层级指定相同的时间值而用户不会注意到，但是，一个层级的本地时间会由它的父级或它自己的时间参数改变。例如，改变层级的*speed*属性会导致该图层及其子层级的持续时间按比例改变。

为了帮助我们确定给定层级适应的时间值，*CALayer*类定义了`convertTime:fromLayer:`与`convertTime:toLayer:`方法。我们可以使用这些方法将固定时间转换成层级的本地时间或者从一个层级的时间转换成另一个层级的。这个方法描述了可能影响层级本地时间，返回我们可能在其他层级使用的媒体时间属性。下面代码展示了从层级获取当前本地时间的例子，`CACurrentMediaTime`是返回电脑当前时间的方法，用来转换方法获取并转换成层级时间。

``` Swift
let localLayerTime = view.layer.convertTime(CACurrentMediaTime(), from: nil)
```

一旦获取到了层级的本地时间，我们可以使用这个值来更新有关时间属性的动画或层级，比如如下操作：
- *beginTime*属性设置动画的启动时间，通常情况下，动画会在下一轮更新开始，但我们可以用来延迟动画的开始。可以用这个属性来合并两个动画，将一个动画的启动时间设置为另一个时间的结束时间。如果需要延迟动画启动，需要我们将*fillMode*设置为*kCAFillModeBackwards*，这个模式会使动画从初始值启动，如果没有设置，动画会在执行之前跳转到结束位置。
- *autoreverses*属性使动画在指定的持续时间内执行，然后返回到动画的初始值。可以将这个属性与*repeatCount*属性结合，在开始与结束值之间来回动画。设置循环次数是整数时，动画会在初始值的位置停止，设置循环次数为额外的一半时（如1.5），会让动画在结束为止停止。
- *timeOffset*属性适用于组动画启动一些动画在其他动画之后的时间启动

## 暂停、恢复动画

暂停动画，可以利用实现*CAMediaTiming*协议的层级设置动画速度为0.0，直到重新修改动画速度恢复动画。

``` Swift
func pauseLayer(layer: CALayer) {
    let pausedTime = layer.convertTime(CACurrentMediaTime(), from: nil)
    layer.speed = 0.0
    layer.timeOffset = pausedTime   // 动画暂停时的时间偏移量
}

func resumeLayer(layer: CALayer) {
    let pausedTime = layer.timeOffset
    layer.speed = 1.0
    layer.timeOffset = 0.0
    layer.beginTime = 0.0
    let timeSincePause = layer.convertTime(CACurrentMediaTime(), from: nil) - pausedTime
    layer.beginTime = timeSincePause    // 动画延迟执行时间
}
```

## 修改动画参数的显式Transactions

对层做的每一项操作必须是Transactions的一部分，*CATransaction*类管理动画的创建与分组，并在适当的实际执行他们。多数情况下，我们不需要自己创建Transactions，当对层级添加隐式或显式动画时，Core Animation自动创建隐式transaction。当然，我们也可以创建显式transactions更精确的管理动画。

我们可以通过*CATransaction*类创建和管理transactions，调用`begin`方法开始一个新的transaction（隐式），`commit`方法结束transaction。在这些调用之间是我们希望对transaction的一部分改变。

``` Swift
CATransaction.begin()
myView.layer.zPosition = 200.0
myView.layer.opacity = 0.0
CATransaction.commit()
```

使用transactions的主要原因之一是在显式transaction的范围内，可以修改持续时间、计时函数和其他参数。我们也可以为整个transaction安排一个完成回调，在一组动画完成后发出通知。修改动画参数需要使用`setValue:forKey:`方法修改字典中对应的键。

``` Swift
CATransaction.begin()
CATransaction.setValue(10.0, forKey: kCATransactionAnimationDuration)
// perform the animations
CATransaction.commit()
```

我们可以在为不同动画集提供不同的默认值的情况下嵌套transactions，通过再次调用*begin*方法可以嵌套另一个transaction，每个*begin*方法必须对应*commit*方法。只有在提交最外层transaction后，Core Animation才会开始相关动画。

## 为动画添加透视

应用程序可以在三维空间操作图层，但为了简单起见，Core Animation使用并行投影显示图层，实际上试讲场景压成二维平面，这种默认行为导致具有不同*zPosition*值的层以相同的大小出现，尽管他们在z方向上距离很远。我们可以通过修改转换矩阵来更改该行为，以包含透视信息。

修改场景的透视图时，需要修改包含被查看视图的父层级的*sublayerTransform*矩阵，修改父层级将相同的透视信息运用于所有子层级可以简化代码，他还能确保透视图被正确的应用在不同平面上相互重叠的兄弟层级。

```
var perspective = CATransform3DIdentity
perspective.m34 = -1.0/eyePosition
view.layer.sublayerTransform = perspective
```

# Changing a Layer's Default Bahavior

Core Animation使用action对象实现它的隐式层级动画行为。action对象是实现*CAAction*协议的对象，它定义了一些要在层上执行的相关行为。所有*CAAnimation*对象实现了该协议，当一个层级属性变化时，这些对象就被安排来执行修改。

动画属性是action的一种，但我们可以定义大多数我们想要操作的actions，不过要做到这一点，我们必须定义action对象并与应用层级对象关联。

## 自定义实现CAAction协议的Action对象

创建自己的action对象，需要一个类实现*CAAction*协议并实现`runActionForKey:object:arguments:`方法，在这个方法中，使用可用的信息来执行我们想要在该层上执行的任何操作。我们可能用这个方法添加一个动画或者一些其他任务。

当我们定义一个action对象，我们需要觉得触发的条件，这个action的触发是我们用来注册这个action使用的key，也可以在以下情况下被触发：
- 层级的一个属性改变了，可以是层级的任意属性，不仅仅是动画的这个（我们还可以把自定义属性的actions和层级相关联），标识action的键是这个属性的名字
- 层级变得可见或者添加到层级结构中，标识的键为*kCAOnOrderIn*
- 层级从层级结构移除，标识的键为*kCAOnOrderOut*
- 层级涉及转换动画，标识的键为*kCATransition*

## action对象必须在层级上才能产生效果

在action操作之前，层级需要找到相应的action对象，层级相关action的键是正在修改属性的名称或特殊字符。当适当的事件发生在层级，层级调用*actionForKey:*方法查找对应键的action对象，我们可以在搜索过程中插入多个点并为该键提供相关的action对象。

Core Animation查找action对象的顺序如下：
1. 如果层级的代理实现了`actionForLayer:forKey:`方法，层级会调用这个方法。代理必须执行以下一项：
    - 返回给定键的action对象
    - 返回nil，如果不处理action，查找继续
    - 返回NSNull对象，查找结束
2. 层级在actions字典中查找对应的键
3. 层级在[style](https://developer.apple.com/documentation/quartzcore/calayer/1410875-style)字典中包含该键的actions字典（换句话说，包含actions键的style字典也是字典，层级在第二层字典查找对应键）
4. 层级调用`defaultActionForKey:`方法
5. 层级执行Core Animation定义的隐式action

如果我们在任意查询点提供了action对象，层级将停止查询并执行返回的action对象。当查找到一个action对象，层级调用`runActionForKey:object:arguments:`方法执行操作。如果给定键定义的action已经是*CAAnimation*类的实例，则可以使用这个方法的默认实现执行动画。如果是我们自定义实现*CAAction*协议的对象，则必须使用该方法对象实现来执行适当的操作。

在何处安装action对象取决于我们如何修改层级：
- 对于可能只在特定环境应用的actions，或对于已经使用代理的对象，提供代理并实现`actionForLayer:forKey`方法
- 对于层级对象补偿使用代理，可以添加action到层级的actions字典中
- 对于与层对象上定义的自定义属性相关的actions，可以在style字典中包含该action
- 对于层行为的基本actions，继承层级并复写`defaultActionForKey:`方法

``` objectivec
- (id<CAAction>)actionForLayer:(CALayer *)theLayer
                        forKey:(NSString *)theKey {
    CATransition *theAnimation=nil;
 
    if ([theKey isEqualToString:@"contents"]) {
 
        theAnimation = [[CATransition alloc] init];
        theAnimation.duration = 1.0;
        theAnimation.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseIn];
        theAnimation.type = kCATransitionPush;
        theAnimation.subtype = kCATransitionFromRight;
    }
    return theAnimation;
}
```

## 使用CATransaction类临时禁用actions

我们可以使用*CATransaction*类临时禁用actions，当修改一个层级的属性时，Core Animation通常会创建隐式transaction执行修改，如果不想修改，我们可以创建显式transaction并设置*kCATransactionDisableActions*属性为true禁用隐式动画。

``` objectivec
[CATransaction begin];
[CATransaction setValue:(id)kCFBooleanTrue
                 forKey:kCATransactionDisableActions];
[aLayer removeFromSuperlayer];
[CATransaction commit];
```

# Ending

这一章主要是对于Layer动画的内容、概念的学习，毕竟要熟练运用需要到实际场景中反复使用，其实更关键的是数学模型，有了数学模型生成的路径、时间规划，才能有绚丽的动画。

# Reference

> [About Core Animation](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/CoreAnimation_guide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40004514-CH1-SW1)