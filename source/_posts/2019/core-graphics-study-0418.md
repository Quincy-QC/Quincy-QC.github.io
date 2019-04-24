---
title: "绘图引擎 -- Quartz 2D"
catalog: true
toc_nav_num: true
date: 2019-04-18 17:40:24
subtitle: "Quartz 2D Programming Guide"
header-img: "/img/article_header/article_header.png"
busuanzi: true
tags:
- iOS

---

> 最近在网上看文章的时候，看到了一篇[在 iOS 中使用 GLSL 实现抖音特效](http://www.lymanli.com/2019/04/05/ios-opengles-filter/)的文章，忽然兴趣就来了，然后就看到了一系列图片处理专用名词，这反而更加勾起了我的兴趣，主要是之前也看到一些关于 OpenGL 的字眼的文章，但是都没有花很多精力去研究，刚好眼下又看到了这篇文章，刚好这个博主也有对 OpenGL 的基础介绍，所以本是打算边看边学，把这个 OpenGL 搞懂。
这个博主对于 OpenGL 的研究文章有3篇，本想着应该问题不大，在完成第一篇的简单使用并绘制图片，到第二篇的对图片进行伸缩，我才发现这里面涵盖的东西非常之广，并不是我想象中那么简单的一个框架，这里面已经涉及到了一个大的领域技术，但是这并没有把我劝退，反而我的内心浮现了一个想法。说实话，作为 iOS 开发者这么多年，我还没有一个可以作为自己专业技术领域的特长，在未来的互联网斗争中如何扩大自己的优势呢，我感觉这个图形处理领域可能会是我进军的领域了。
所以，我打算从最简单的绘图开始 -- iOS 绘图框架 Core Graphics。

# Introduction

Quartz 2D是一个二维绘图引擎，可以在 iOS 环境和内核之外的所有 Mac OS X 应用程序环境下使用。我们可以使用 Quartz 2D API 来使用功能，比如基于路径的绘图，具有透明度的绘制，阴影，绘制阴影，透明层，颜色管理，抗锯齿渲染，PDF 文档生成，PDF 元数据访问等。在 iOS 中，Quartz 2D 可以使用所有可用的图形和动画技术，比如 Core Animation, OpenGL ES, UIKit 类等。（只讨论 iOS 范畴）

# Concepts

## 绘图点：The Graphics Context

图形上下文是一种不透明的数据类型（CGContextRef），他封装了 Quartz 用于将图像绘制到输出设备（如 PDF 文件，位图，显示窗口）的信息。图像上下文中的信息包括图像绘制参数和页面上绘制的特定于设备的表示。Quartz 中所有的对象被绘制到或者包含在图像上下文中。
图像上下文包含以下几种：

1. A bitmap graphics context：位图图形上下文，允许你在位图中绘制 RGB 颜色，CMYK 颜色，灰度模式。位图是像素的矩阵阵列（或光栅），每个像素表示图像中的一个点。位图图形也称为采样图像。

2. A PDF graphics context：PDF 图形上下文，允许你创建 PDF 文件。在 PDF 文件中，绘图被保存为一系列命令。PDF 文件与位图之前有以下区别：
- PDF 文件不同于位图，可以包含超过一页内容
- 当你在不同的设备从 PDF 文件中绘制页面时，生成的图像将根据该设备的显示特性进行优化
- PDF 文件本质上是独立于分辨率--在不牺牲图像细节的情况下，他们被绘制的大小可以无限增加或减小；位图图像的用户感知质量与位图的显示分辨率有关

3. A window graphics context：可以让你在窗口绘图。（适用于 Mac OS X）

4. A layer context：图层上下文，与另一个图形上下文关联的离屏绘图位置。当将层绘制到创建它的图形上下文时，它的设计是为了获得最佳性能。对于屏幕外绘制，图层上下文比位图图形上下文是更好的选择。

## 图形状态：Graphics States

Quartz 根据当前图形状态下的参数修改绘图结果。图形上下文包含一组图形状态，当 Quartz 创建上下文，状态是空的，只有当你保存图形状态时，才会将图形状态保存到堆栈。当恢复图形状态时，会将当前图形状态从堆栈顶部弹出，弹出状态变为上次保存的状态。
``` objectivec
CGContextSaveGState();  // 存储状态
CGContextRestoreGState();   // 恢复状态
```

## 坐标系：Quartz 2D Coordinate Systems

Quartz 2D坐标系：
![Quartz 2D坐标系](/img/article/20190418/1.png "Quartz 2D坐标系")

由于不同设备具有不同的底层成像功能，所以必须以与设备无关的方式定义设备的位置和大小。Quartz 通过单独的坐标系统--用户空间--将其映射到输出设备--设备空间--使用当前变换矩阵（CTM）来实现设备的独立性。当前的变换矩阵是一种特殊类型的矩阵，成为仿射变换，他通过应用平移、旋转、缩放（移动、旋转、调整坐标系大小）等操作将点从一个坐标系空间映射到另一个坐标系空间。

有些技术使用不同于 Quartz 使用的默认坐标系来设置他们的图形上下文，相对于 Quartz，这样的坐标系是经过修改的坐标系，在执行一些 Quartz 绘图操作时必须对其进行补偿。最常见的修改坐标系就是讲原点放在上下文的左上角，并将y轴指向页面底部。比如：
- Mac OS X，继承 NSView 并重写 *isFlipped* 返回 YES
- iOS，UIView 返回的绘图上下文
- iOS，*UIGraphicsBeginImageContextWithOptions* 调用返回的绘图上下文

因为 UIKit 本身使用了不同于 Quartz 的默认坐标系约定，所以修改了绘图上下文坐标系；它将转换应用于它创建的 Quartz 上下文以匹配他们的约定。这些变换会导致一系列问题，比如路径绘制过程中，圆弧在默认坐标系中是顺时针绘制，但如果修改了坐标系，则会变成逆时针绘制，就像镜像反射。

# Paths

路径定义了一个或多个形状或子路径。

## 一些基本绘图操作
``` objectivec
/// 点 Points
CGContextMoveToPoint(context, x, y);

/// 线 Lines
CGContextAddLineToPoint(context, x, y);
CGContextAddLines(context, [points], count)

/// 圆弧
CGContextAddArc(context, x, y, radius, startAngle, endAngle, clockwise);
CGContextAddArcToPoint(context, x1, y1, x2, y2, radius);    // 前两个点表示切点

/// 曲线 Curves
CGContextAddCurveToPoint(context, x1, y1, x2, y2, x, y);    // 前两个点控制切线方向，最后一个点是结点
CGContextAddQuadCurveToPoint(context, x1, y1, x, y)         // 第一个点是切点，最后一个点是结点

/// 结束一段子路径
CGContextClosePath(context);

/// 椭圆 Ellipses
CGContextAddEllipseInRect(context, rect);

/// 矩形 Rectangles
CGContextAddRect(context, rect);
CGContextAddRects(context, [rects], count);

/// 开始绘制路径
CGContextBeginPath(context);

```

## 创建路径
```objectivec
/// 获取路径对象
CGMutablePathRef mutablePath = CGPathCreateMutable();
CGPathRef path = CGContextCopyPath(context);

/// 移动绘图起始点
CGPathMoveToPoint(path, NULL, x, y);    // 第二个参数，可以修改坐标系（CGContextTranslateCTM, CGContextScaleCTM, or CGContextRotateCTM）

/// 其余与上述类似 ...
```

## 闭合路径

- 影响闭合路径的参数：
``` objectivec
/// 线条粗细
CGContextSetLineWidth(context, width);

/// 线条转折处样式
CGContextSetLineJoin(context, join);        // kCGLineJoinMiter (the default), kCGLineJoinRound 圆角, or kCGLineJoinBevel 折角

/// 线条转折样式为 Miter 时，
CGContextSetMiterLimit(context, limit);     // 斜接的长度除以线的宽度，如果结果大于斜接极限，则样式转换为Bevel。

/// 线条终起点样式
CGContextSetLineCap(context, cap);          // kCGLineCapButt (the default), kCGLineCapRound 圆角, or kCGLineCapSquare 折角

/// 线条 虚线样式
CGContextSetLineDash(context, phase, [lengths], count);     // phase: 起点处，指定从虚线的哪个位置开始绘制

/// 线条颜色
CGContextSetStrokeColorWithColor(context, color);
```

- 闭合路径的几种方式：
``` objectivec
/// 闭合
CGContextStrokePath(context);

/// 矩形闭合
CGContextStrokeRect(context, rect);

/// 特定线粗细矩形闭合
CGContextStrokeRectWithWidth(context, rect, width);      // width: 线粗细

/// 椭圆闭合
CGContextStrokeEllipseInRect(context, rect);

/// 多条线闭合
CGContextStrokeLineSegments(context, [points], count);      // points: 每两个点定义点一条线

/// 闭合
CGContextDrawPath(context, kCGPathFillStroke || kCGPathEOFillStroke);
```

## 填充路径

填充路径的几种方式：
```objectivec
/// 填充 奇偶规则 even-odd rule
CGContextEOFillPath(context);   // 射线穿过图形，穿过路径+1，奇数为路径内部，偶数为路径外部

/// 填充 非零圈数规则 nonzero winding number rule
CGContextFillPath(context);     // 射线穿过图形，路径从左到右穿过射线+1，从右到左-1，0位路径外部，非0为路径内部

/// 矩形填充
CGContextFillRect(context, rect);

/// 椭圆填充
CGContextFillEllipseInRect(context, rect);

/// 填充
CGContextDrawPath(context, kCGPathFill || kCGPathEOFill);
```

## 修剪路径

``` objectivec
/// 修剪 非零圈数规则 nonzero winding number rule
CGContextClip(context);

/// 修剪 奇偶规则 even-odd rule
CGContextEOClip(context);

/// 矩形修剪
CGContextClipToRect(context, rect);

/// 矩形修剪图形
CGContextClipToMask(context, rect, mask);
```

# Transforms

## 修改用户坐标系

``` objectivec
/// 平移
CGContextTranslateCTM(context, x, y);
    
/// 旋转
CGContextRotateCTM(context, angle);
    
/// 缩放
CGContextScaleCTM(context, x, y);

/// 根据特定矩阵转换用户坐标系
CGContextConcatCTM(context, transform);
```

## 创建仿射变换

``` objectivec
/// 平移
CGAffineTransformMakeTranslation(x, y);

/// 以transform基础进行平移
CGAffineTransformTranslate(transform, x, y);

/// 旋转
CGAffineTransformMakeRotation(angle);

/// 以transform基础进行旋转
CGAffineTransformRotate(transform, angle);

/// 缩放
CGAffineTransformMakeScale(x, y);

/// 以transform基础进行缩放
CGAffineTransformScale(transform, x, y);

/// 对point进行矩阵变换
CGPointApplyAffineTransform(point, transform);

/// 对size进行矩阵变换
CGSizeApplyAffineTransform(size, transform);

/// 对rect进行矩阵变换
CGRectApplyAffineTransform(rect, transform);
```

## 评估仿射变换

``` objectivec
/// 仿射变换是否相同
CGAffineTransformEqualToTransform(transform1, transform2);

/// 仿射变换是否为初始状态
CGAffineTransformIsIdentity(transform);

/// 初始状态
CGAffineTransformIdentity;
```

## 用户空间与设备空间转换

``` objectivec
/// point转换
CGContextConvertPointToDeviceSpace(context, point);
CGContextConvertPointToUserSpace(context, point);

/// size转换
CGContextConvertSizeToDeviceSpace(context, size);
CGContextConvertRectToUserSpace(context, size);

/// rect转换
CGContextConvertRectToDeviceSpace(context, rect);
CGContextConvertRectToUserSpace(context, rect);
```

# Patterns

Pattern 是一组绘图操作序列，这些操作被反复操作到图形上下文中。可以像使用颜色一样使用 Pattern，Quartz 会把页面划分为一组 Pattern 单元格，每个单元格大小与 Pattern 图形相似，并使用提供的回调函数绘制每个单元格。
![A Pattern drawn to a window](/img/article/20190418/2.png)

## 编写一个回调函数来绘制一个着色图案单元格

``` objectivec
/**
 绘制着色图案单元格的绘图回调

 @param info 指向与图案关联的私有数据的泛型指针
 @param myContext 绘图上下文
 */
void MyDrawColoredPattern(void *info, CGContextRef myContext) {
    CGFloat subunit = 5;
    
    CGRect myRect1 = {{0, 0}, {subunit, subunit}},
    myRect2 = {{subunit, subunit}, {subunit, subunit}},
    myRect3 = {{0, subunit}, {subunit, subunit}},
    myRect4 = {{subunit, 0}, {subunit, subunit}};
    
    CGContextSetRGBFillColor(myContext, 0, 0, 1, .5);
    CGContextFillRect(myContext, myRect1);
    CGContextSetRGBFillColor(myContext, 1, 0, 0, .5);
    CGContextFillRect(myContext, myRect2);
    CGContextSetRGBFillColor(myContext, 0, 1, 0, .5);
    CGContextFillRect(myContext, myRect3);
    CGContextSetRGBFillColor(myContext, .5, 0, .5, .5);
    CGContextFillRect(myContext, myRect4);
}
```

## 设置着色图案颜色空间

允许*MyDrawColoredPattern*代码中使用颜色绘制图案单元格的前提是，将基本图案颜色空间设置为NULL。

``` objectivec
CGColorSpaceRef patternSpace;
patternSpace = CGColorSpaceCreatePattern(NULL);
CGContextSetFillColorSpace(context, patternSpace);
CGColorSpaceRelease(patternSpace);
```

## 建立着色图案剖析

``` objectivec
/**
 创建图案

 @param NULL 传递到*MyDrawColoredPattern*绘图回调中的info
 @param patternRect 图案rect
 @param CGAffineTransformIdentity 图形变换
 @param 16 单元格之间的水平位移（可以理解为单元格宽）
 @param 18 单元格之间的竖直位移（可以理解为单元格高）
 @param kCGPatternTilingConstantSpacing 拼接方式
 @param YES 是否为着色模式（否则为模板模式，应该是印刷模式）
 @param callbacks 绘图回调函数
 @return 图案
 */
CGPatternRef myPattern = CGPatternCreate(NULL, patternRect, CGAffineTransformIdentity, 16, 18, kCGPatternTilingConstantSpacing, YES, &callbacks);
```

## 将着色图案填充、描边

``` objectivec
CGFloat alpha = 1;
CGContextSetFillPattern(context, myPattern, &alpha);
```

## 完整的彩绘封装

``` objectivec
void MyColorPatternPainting(CGContextRef myContext, CGRect rect) {
    CGPatternRef pattern;
    CGColorSpaceRef patternSpace;
    CGFloat alpha = 1;
    static const CGPatternCallbacks callbacks = {0, &MyDrawColoredPattern, NULL};
    CGContextSaveGState(myContext);
    patternSpace = CGColorSpaceCreatePattern(NULL);
    CGContextSetFillColorSpace(myContext, patternSpace);
    CGColorSpaceRelease(patternSpace);
    
    pattern = CGPatternCreate(NULL, CGRectMake(0, 0, 16, 18), CGAffineTransformMake(1, 0, 0, 1, 0, 0), 16, 18, kCGPatternTilingConstantSpacing, true, &callbacks);
    
    CGContextSetFillPattern(myContext, pattern, &alpha);
    CGPatternRelease(pattern);
    CGContextFillRect(myContext, rect);
    CGContextRestoreGState(myContext);
}

void MyDrawColoredPattern(void *info, CGContextRef myContext) {
    CGFloat subunit = 5;
    
    CGRect myRect1 = {{0, 0}, {subunit, subunit}},
    myRect2 = {{subunit, subunit}, {subunit, subunit}},
    myRect3 = {{0, subunit}, {subunit, subunit}},
    myRect4 = {{subunit, 0}, {subunit, subunit}};
    
    CGContextSetRGBFillColor(myContext, 0, 0, 1, .5);
    CGContextFillRect(myContext, myRect1);
    CGContextSetRGBFillColor(myContext, 1, 0, 0, .5);
    CGContextFillRect(myContext, myRect2);
    CGContextSetRGBFillColor(myContext, 0, 1, 0, .5);
    CGContextFillRect(myContext, myRect3);
    CGContextSetRGBFillColor(myContext, .5, 0, .5, .5);
    CGContextFillRect(myContext, myRect4);
}
```

# Shadows

## 阴影的使用

``` objectivec
CGContextSaveGState(context);
CGContextSetShadow(context, offsetsize, blur);  // offset：偏移量（UIKit下：x向右为正，y向下为正），blur：阴影量，颜色默认为（0, 0, 0, 1.0/3.0）
CGContextSetShadowWithColor(context, offsetsize, blur, color);
/* 其他绘图操作 */
CGContextRestoreGState(context);
```

# Gradients

Quartz 提供两种数据类型创建渐变色 -- *CGShadingRef* 和 *CGGradientRef*。同时，这两种数据类型又可以分别创建轴向(axial)或者径向(radial)的渐变线。

An axial gradient(a linear gradient) -- 在定义的两个端点之间沿轴向变化，所有垂直于该轴向的直线上的点具有相同的色值。
A radial gradient -- 在定义的两个端点（通常都是圆）之间沿轴向径向变化的填充，所有圆心落在轴线上圆的圆周上的点具有相同的色值，梯度的圆截面半径由两端半径确定，半径从一端到另一端呈线性变化。

## CGShading 与 CGGradient 比较

CGShadingRef -- 可以自己控制渐变中每个点颜色值的计算，非常细节。在创建CGShading对象之前，必须先创建CGFunction对象来定义梯度颜色计算的函数，自定义的函数可以更自由的创建更流畅的颜色梯度。

CGRadientRef -- 是CGShadingRef的子类，使用更加方便。Quartz为我们计算渐变中每个点的颜色，不需要我们提供计算函数，只需要提供作用域和颜色值，Quartz会自动计算每个连续位置的颜色梯度。

## CGGradient

### 轴向渐变线

``` objectivec
CGGradientRef myGradient;
CGColorSpaceRef myColorspace;
size_t num_locations = 2;
CGFloat locations[2] = { 0.0, 1.0 };
CGFloat components[8] = {
    1.0, 0.5, 0.4, 1.0,  // Start color
    0.8, 0.8, 0.3, 1.0   // End color
};
CGPoint myStartPoint, myEndPoint;
myColorspace = CGColorSpaceCreateWithName(kCGColorSpaceGenericRGB);
myGradient = CGGradientCreateWithColorComponents(myColorspace, components, locations, num_locations);
CGContextDrawLinearGradient(context, myGradient, myStartPoint, myEndPoint, kCGGradientDrawsBeforeStartLocation | kCGGradientDrawsAfterEndLocation);
```

### 径向渐变线

``` objectivec
CGGradientRef myGradient;
CGColorSpaceRef myColorspace;
size_t num_locations = 2;
CGFloat locations[2] = { 0.0, 1.0 };
CGFloat components[8] = {
    1.0, 0.5, 0.4, 1.0,  // Start color
    0.8, 0.8, 0.3, 1.0   // End color
};
CGFloat myStartRadius, myEndRadius;
CGPoint myStartPoint, myEndPoint;
myColorspace = CGColorSpaceCreateWithName(kCGColorSpaceGenericRGB);
myGradient = CGGradientCreateWithColorComponents(myColorspace, components, locations, num_locations);
CGContextDrawRadialGradient(context, myGradient, myStartPoint, myStartRadius, myEndPoint, myEndRadius, 0);
```

## CGShading

暂不予考虑

# Transparency Layers

当多个对象组合成复合图形，如何将复合图形视为单一对象，形成如同单一对象般的应用效果，这就要用到透明层这个概念。

![Transparency Layers](/img/article/20190418/3.png)

``` objectivec
CGContextSetShadow(context, CGSizeMake(10, 20), 10);
CGContextBeginTransparencyLayer(context, NULL);
// your drawing code
CGContextEndTransparencyLayer(context);
```

# Ending

当然，Quartz 2D的内容不止这么多，我现在整理的也只是一些简单易用的皮毛，针对于 iOS 应用的一些简易图形绘制应该是没有问题了，总的说来，学无止境，后面可能要继续对Core Image的学习。

# Reference

> [Quartz 2D Programming Guide](https://developer.apple.com/library/archive/documentation/GraphicsImaging/Conceptual/drawingwithquartz2d/dq_overview/dq_overview.html#//apple_ref/doc/uid/TP30001066-CH202-TPXREF101)