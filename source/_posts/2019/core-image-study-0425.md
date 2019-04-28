---
title: "图像处理与分析 -- Core Image"
catalog: true
toc_nav_num: true
date: 2019-04-25 18:40:24
subtitle: "About Core Image"
header-img: "/img/article_header/article_header.png"
busuanzi: true
tags:
- iOS

---

# Introduction

Core Image 是一种图像处理和分析技术，旨在为静态或者视频图片提供近乎实时的处理。它对来自Core Graphics，Core Video，Image I/O框架中的图片数据类型进行操作，使用GPU或CPU渲染路径。Core Image通过提供一个易于使用的API隐藏底层图片处理的细节，我们不需要知道OpenGL，OpenGL ES，Metal的细节来可以利用GPU的能力，也不需要知道任何关于GCD的知识就可以进行多核处理，所有的细节Core Image都会帮你处理。

![Core Image](/img/article/20190425/1.png)

Core Image 提供以下功能：
- 访问内置的图像处理过滤器
- 特征检测能力
- 支持自动图像增强
- 将多个滤镜链接一起形成自定义效果的能力
- 支持创建运行在GPU上的自定义滤镜
- 基于反馈的图像处理能力
MacOS还提供一种打包自定义滤镜以供其他应用使用的方法。

# Processing Images

处理图片意味着应用滤镜 -- 图片滤镜是一种逐像素检测输入图片并通过算法应用一些效果来生成输出图片的软件。图片处理主要依靠描述滤镜及其输入输出的CIFilter和CIImage类，你可以使用Core Image与其他系统框架或者创建通过CIContext创建自己的渲染工作流来应用滤镜、显示或导出结果。

在图片上应用滤镜的基本代码：
``` Swift
let context = CIContext()
let filter = CIFilter(name: "CISepiaTone")
filter?.setValue(0.8, forKey: kCIInputIntensityKey)
let image = CIImage(contentsOf: Bundle.main.url(forResource: "", withExtension: "")!)
filter?.setValue(image, forKey: kCIInputImageKey)
let result = filter?.outputImage
if let result = result {
    let cgImage = context.createCGImage(result, from: result.extent)
}
```

滤镜处理并制作图片，CIImage实例是表示一张图片的不可变对象，这些对象不直接表示图片位图数据，相反，CIImage对象是制作图片的一种方式。一种方式可能要求直接从文件中加载图片，另一种可能是从滤镜或过滤器链的输出，Core Image 仅当你要求将一张图片渲染作为显示或者输出时执行这些方法。

应用滤镜的方式，创建一个或多个CIImage对象表示被滤镜处理的图片同时配置滤镜的输入参数，你可以从以下几种图片数据中创建CIImage对象：
- 被加载图片文件的URL引用或者被加载包含图片文件数据的NSData对象
- Quartz2D，UIKit，或者AppKit的图片代表（CGImageRef，UIImage，NSBitmapImageRep）
- Metal，OpenGL，OpenGl ES纹理
- CoreVideo图片或者像素缓存（CVImageBufferRef，CVPixelBufferRef）
- 进程间分享的图片数据 -- IOSurfaceRef
- 内存中的位图数据（对象指针，按需提供数据的CIImageProvider）

因为CIImage对象描述了如何制作图片（而不是包含图片数据），他同样可以表示滤镜输出。当访问CIFilter对象的outputImage属性时，Core Image只标识和存储执行滤镜需要的步骤，这些步骤只有在请求显示或输出而渲染图片时运行。

## 描述图片处理效果 -- 滤镜

CIFilter对象是代表图片处理效果和控制效果行为系列参数的可变对象，使用滤镜的方式：创建CIFilter对象，设置输入参数，然后访问它的输出图像。[各类滤镜](https://developer.apple.com/library/archive/documentation/GraphicsImaging/Reference/CoreImageFilterReference/index.html#//apple_ref/doc/uid/TP40004346)

### 复杂效果 -- 链式滤镜

每个Core Image提供一个CIImage输出对象，可以把这个输出对象作为另一个滤镜的输入。

!(Construct a filter chain by connecting filter inputs and outputs)[/img/20190425/2.png]

Core Image 优化了滤镜链的应用，以快速高效的方式渲染结果，链中的每个CIImage对象不是完全渲染过的图片，而是渲染的方式，Core Image不需要单独执行每个滤镜操作，这样会非常浪费时间和内存，相反，Core Image将滤镜组合成一个单独的操作，甚至以不同的顺序重新组织滤镜，从而更有效的生成相同的结果。

!(Core Image optimizes a filter chain to a single operation)[/img/20190425/2.png]

我们可以看出，Crop滤镜操作从最后移动到优先执行，这个滤镜用于输出裁剪后的图片，因此不需要对裁剪外的像素使用颜色和锐化过滤器，所以Core Image首先执行裁剪，确保消耗巨大的图形处理操作只适用于最终输出可见的像素内。

``` Swift
func applyFilterChain(to image: CIImage) -> CIImage? {
    let colorFilter = CIFilter(name: "CIPhotoEffectProcess", parameters: [kCIInputImageKey: image])
    guard let image = colorFilter?.outputImage else {
        return nil
    }
    let bloomImage = image.applyingFilter("CIBloom", parameters: [kCIInputRadiusKey: 10.0, kCIInputIntensityKey: 1.0])
    let cropRect = CGRect(x: 0, y: 0, width: 150, height: 150)  // 左下角为坐标原点
    let croppedImage = bloomImage.cropped(to: cropRect) // 用于常用的筛选操作，比如croped, clamped
    return croppedImage
}
```

### 特殊类型的滤镜

[生成二维码](https://developer.apple.com/library/archive/documentation/GraphicsImaging/Reference/CoreImageFilterReference/index.html#//apple_ref/doc/filter/ci/CIQRCodeGenerator)
[生成条形码](https://developer.apple.com/library/archive/documentation/GraphicsImaging/Reference/CoreImageFilterReference/index.html#//apple_ref/doc/filter/ci/CICode128BarcodeGenerator)
[其他](https://developer.apple.com/library/archive/documentation/GraphicsImaging/Conceptual/CoreImaging/ci_tasks/ci_tasks.html#//apple_ref/doc/uid/TP30001185-CH3-DontLinkElementID_6)

## 用Core Image上下文构建自己的工作流

主要涉及Metal，OpenGL，OpenGL ES等实时渲染，后面学到Metal的时候再来看。

# Detecting Faces in an Image

Core Image可以在图片中解析找到人脸。它执行的是人脸检测而不是人脸识别，人脸检测是包含人脸特征的矩形的识别，而人脸识别是对特定人脸的识别。Core Image检测人脸后，可以提供人脸特征信息，例如眼睛或鼻子的位置。它也可以跟踪视频中人脸的位置。

``` Swift
DispatchQueue.global().async {
    let image = CIImage(contentsOf: Bundle.main.url(forResource: "7", withExtension: "jpg")!)
    let context = CIContext()
    let opts = [CIDetectorAccuracy: CIDetectorAccuracyHigh]
    let detector = CIDetector(ofType: CIDetectorTypeFace, context: context, options: opts)
    let features = detector?.features(in: image!) as? [CIFaceFeature]
    if let features = features {
        for feature in features {
            print(feature.bounds)
            
            if feature.hasLeftEyePosition {
                print("Left eye \(feature.leftEyePosition.x) \(feature.leftEyePosition.y)")
            }
            
            if feature.hasRightEyePosition {
                print("Right eye \(feature.rightEyePosition.x) \(feature.rightEyePosition.y)")
            }
            
            if feature.hasMouthPosition {
                print("Mouth \(feature.mouthPosition)")
            }
        }
    }
}
```

# Auto Enhancing Images

Core Image的自动优化特征用于分析图片的直方图，人脸区域内容，元数据属性，然后它返回一个CIFilter对象数组，这些对象已经被设置为改进后图片的输入参数。

下面展示了Core Image用于自动优化图片的滤镜：
- CIRedEyeCorrection：修复由于相机闪光灯造成的红眼、琥珀眼、白眼
- CIFaceBalance：调整脸部颜色，使皮肤更适应
- CIVibrance：增强图片饱和度而不扭曲肤色
- CIToneCurve：调整图像对比度
- CIHighlightShadowAdjust：调整阴影细节

``` Swift
let adjustments = image.autoAdjustmentFilters()
for filter in adjustments {
    filter.setValue(image, forKey: kCIInputImageKey)
    image = filter.outputImage!
}
```

# Subclassing CIFilter: Reciptes for Custom Effects

我们可以通过图片的输出作为另一个的输入合并多个你喜欢的滤镜来创建自定义的效果，当你多次通过这个方式创建效果时，可以考虑继承CIFilter来封装一个滤镜。

## Subclassing CIFilter to Create the CIColorInvert Filter

当我们继承CIFilter后，我们可以通过重新编写默认值或合并滤镜来修改已有的滤镜。Core Image用这项技术实现一些内置滤镜。
继承一个滤镜我们需要实现以下内容：
- 定义滤镜的输入参数属性，必须以**input**作为参数名开头，比如*inputImage*
- 如有必要复写**setDefaults**方法
- 如些**outputImage**方法

Core Image提供的CIColorInvert滤镜是CIColorMatrix滤镜的改进，顾名思义，CIColorInvert为CIColorMatrix提供向量，将输入图片的颜色反转。

``` Swift
class CIColorInvert: CIFilter {
    public var inputImage: CIImage?
    
    override var outputImage: CIImage? {
        get {
            guard let inputImage = inputImage else {
                return nil
            }
            let filter = CIFilter(name: "CIColorMatrix",
                                  parameters: [kCIInputImageKey: inputImage,
                                               "inputRVector": CIVector(x: -1, y: 0, z: 0),
                                               "inputGVector": CIVector(x: 0, y: -1, z: 0),
                                               "inputBVector": CIVector(x: 0, y: 0, z: -1),
                                               "inputBiasVector": CIVector(x: 1, y: 1, z: 1)])
            return filter?.outputImage
        }
    }
}
```

## Chrome Key Filter Recipe

从资源图移除背景颜然后与新背景融合。
创建色度滤镜：
- 创建一个我们想要移除颜色数据的立方体映射，使他们透明
- 使用CIColorCube滤镜从图片中移除对应色度的颜色
- 使用CISourceOverCompositing滤镜将源图与背景图混合

### 创建立方体映射

一个颜色立方体是一个3D颜色查询表。CIColorCube滤镜把色值作为输入，在表中查询这些值，默认的查询表是一个标识矩阵--意味着对提供的数据不做任何操作。但是，这个方法要求我们删除一种颜色绿色。
我们需要从图片中移除所有的绿色，把绿色的可见度设为0。”绿色“包含系列颜色，最直接的方法是把颜色从RGBA转换成HSV类型值，在HSV中，H表示为围绕圆柱体中轴的一个角，在这种表示中，我们可以把颜色作为一个饼状切片，然后简单的删除对应颜色块。
移除绿色，我们需要定义包含绿色最大最小角度值，然后，所有范围内的值的可见度都设为0，纯绿色对应120°，最大最小角度应该围绕这个值。
立方体映射数据必须提前乘以alpha，所以创建数据的最后一步是将RGB值乘以计算出来的alpha值，对于绿色，alpha为0，其他为1.0.

``` Swift
let size = 64
let count = size * size * size * 4
let cubeData = UnsafeMutablePointer<Float>.allocate(capacity: count)
var rgb: [Double] = [0, 0, 0], hsv: [Double] = [0, 0, 0]
for z in 0..<size {
    rgb[2] = (Double(z)) / Double(size - 1)
    for y in 0..<size {
        rgb[1] = Double(y) / Double(size - 1)
        for x in 0..<size {
            rgb[0] = Double(x) / Double(size - 1)
            // rgbToHSV(rgb, hsv)
            let alpha = (hsv[0] > minHugAngle && hsv[0] < maxHueAngle) ? 0.0f : 1.0f
            let offset = x * 4 + y * size * 4 + z * size * size * 4
            cubeData[offset] = Float(rgb[0] * alpha)
            cubeData[offset + 1] = Float(rgb[1] * alpha)
            cubeData[offset + 2] = Float(rgb[2] * alpha)
            cubeData[offset + 3] = Float(alpha)
            cubeData.moveAssign(from: cubeData + 3, count: 4)
        }
    }
}
let data = Data(bytesNoCopy: cubeData, count: count, deallocator: .free)
let colorCube = CIFilter(name: "CIColorCube")
colorCube?.setValue(NSNumber(value: size), forKey: "inputCubeDimension")
colorCube?.setValue(data, forKey: "inputCubeData")
```

### 从源图移除绿色

``` Swift
colorCube?.setValue(inputImage, forKey: kCIInputImageKey)
let result = colorCube?.outputImage
```

### 融合处理后源图与背景图

运用CISourceOverCompositing滤镜融合图片：
- 设置inputImage为处理后的源图
- 设置inputBackgroundImage为新的背景图

# Getting the Best Performance

Core Image提供多种选项创建图片，上下文，渲染上下文。看你如何选择完成一项任务：
- 应用多久执行一次任务
- 应用使用静态还是视频图片
- 是否需要支持实时处理和分析
- 颜色保真度对用户的重要性

## 性能最佳实践

- 不要在每次渲染时创建CIContext对象；上下文存储非常多状态信息；复用更加有效
- 评估应用是否需要颜色管理；如非必要不要使用
- 避免在GPU渲染CIImage对象时使用Core Animation动画；如果想要同时使用，则用CPU
- 确保图片不超过CPU与GPU的限制；CIContext对象的图片大小限制取决于使用的是CPU还是GPU
- 尽可能使用小图；性能随输出像素改变
- UIImageView与静态图片更适配
- 避免CPU与GPU之间非必要的纹理传输
- 考虑使用简单的滤镜类似于算法滤镜产生的结果
- 利用iOS 6.0及更高版本对YUV图片的支持；相机像素缓存是天生的YUV，但大多数图像处理算法期望RGBA数据，两者之间的转换是有成本的；Core Image支持从CVPixelBuffer对象读取YUB并应用适当的颜色转换

# Creating Custom Filters

[What You Need to Know Before Writing a Custom Filter](https://developer.apple.com/library/archive/documentation/GraphicsImaging/Conceptual/CoreImaging/ci_advanced_concepts/ci.advanced_concepts.html#//apple_ref/doc/uid/TP30001185-CH9-SW1)

[Creating Custom Filters](https://developer.apple.com/library/archive/documentation/GraphicsImaging/Conceptual/CoreImaging/ci_custom_filters/ci_custom_filters.html#//apple_ref/doc/uid/TP30001185-CH6-TPXREF101)

# Reference

> [About Core Image](https://developer.apple.com/library/archive/documentation/GraphicsImaging/Conceptual/CoreImaging/ci_intro/ci_intro.html#//apple_ref/doc/uid/TP30001185-CH1-TPXREF101)