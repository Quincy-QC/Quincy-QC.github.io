---
title: "Bitmap - 位的映射"
catalog: true
toc_nav_num: true
date: 2022-01-10 19:00:08
subtitle: "A mapping from some domain (for example, a range of integers) to bits"
header-img: "/img/article_header/article_header.png"
busuanzi: true
tags:
- iOS - Swift

---

# 前言

`BitMap`从字面的意思，很多人认为是位图，其实更准确的来说，是位的映射。下面我们分别从位映射算法和绘图中的绘图这两块内容来介绍`Bitmap`。

# Bitmap 算法

`Bitmap`的基本思想就是用一个`bit`位来标记某个元素对应的`Value`，而`Key`即是该元素。由于采用`bit`为单位来存储数据，所以在存储空间方面，可以大大节约空间。

假设有这样一个需求：在20亿个随机整数中找出某个数是否存在，那么：
- 如果每个数字用`UInt32`存储，占用的空间为`2_000_000_000 * 4 / 1024 / 1024 / 1024 ≈ 7.25G`
- 如果按位存储，20亿个数字就是20亿位，占用空间为`2_000_000_000 / 8 / 1024 / 1024 / 1024 ≈ 0.23G`
所以我们可以看出，按位存储在数据存储中所占有的优势。

## 数据存储

那么我们如果表示一个数呢？
既然我们是按位存储，那么每一位就代表当前位置的数据是否存在，可以用0或者1表示，正好符合二进制。
那么，我们就可以通过二进制表示数字，比如`{1, 3, 5, 7}`：

<font color=red>1</font> | 0 | <font color=red>1</font> | 0 | <font color=red>1</font> | 0 | <font color=red>1</font> | 0
:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:
<font color=red>7</font> | 6 | <font color=red>5</font> | 4 | <font color=red>3</font> | 2 | <font color=red>1</font> | 0

计算机内存分配的最小单位是字节，也就是8位，所以很容易表示小于8的数，那么对于`{10, 11, 14}`的数，可以在另一个8位上表示：

<font color=red>1</font> | 0 | <font color=red>1</font> | 0 | <font color=red>1</font> | 0 | <font color=red>1</font> | 0
:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:
<font color=red>7</font> | 6 | <font color=red>5</font> | 4 | <font color=red>3</font> | 2 | <font color=red>1</font> | 0

0 | <font color=red>1</font> | 0 | 0 | <font color=red>1</font> | <font color=red>1</font> | 0 | 0
:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:
15 | <font color=red>14</font> | 13 | 12 | <font color=red>11</font> | <font color=red>10</font> | 9 | 8

一个`Int32`占32位，那么一个`Int32`可以存储32个数字，那么对于需要存储数字的最大值`N`，可以得到需要申请一个`let tmp: [Int32]`数组长度为`(1 + N / 32)`即可存储：
```swift
tmp[0]：可以表示0~31
tmp[1]：可以表示31~63
tmp[2]：可以表示64~95
。。。
```

## 添加

那我们如何将一个数字`P`放进去呢？
我们就可以通过位运算`|或`，对于是存储在哪一个`Int32`，可以通过`P / 32`计算出。所以在`{1, 3, 5, 7}`中插入数字`6`：
``` swift
0x10101010 | (1 << 6) = 0x10101010 | 0x01000000 = 0x11101010
// 转换为十进制：
170 | 64 = 234
```
所以可以总结为：
```swift
tmp[P/8] |= 1 << (P % 8)
```

## 删除

那么我们删除一个数字`Q`呢？
我们可以通过位运算`&与`，与上同理找到对于存储的位置，然后对应删除数字`6`：
```swift
0x11101010 & ~(1 << 6) = 0x11101010 & 0x10111111 = 0x10101010
// 转换为十进制
234 & ~64 = 170
```
所以可以总结为：
```swift
tmp[Q/8] &= ~(1 << (Q % 8))
```

## 查找

查找数字`R`就更简单了，可以通过位运算`&与`，判断`tmp[R/8] & (1 << (R % 8))`的结果值，如果为`1`则存在，`0`则不存在。

## 用途

那么这种`Bitmap`有什么用呢？
可以进行大量数据的快速排序、查找和去重等等，那么我们就抓到了他的关键点，对于大量数据的存储和查找，这就可以引入到我们位图的概念。

# Bitmap 位图

## 概念

**位图**，又称**栅格图(Raster Graphics)**或**点阵图**，是使用像素阵列(Pixel-array/Dot-matrix点阵)来表示的图像。

位图的像素都分配有特定的位置和颜色值，每个像素的颜色信息由RGB组合或者灰度值表示。

根据位深度,可将位图分为1、4、8、16、24及32位图像等。每个像素使用的信息位数越多，可用的颜色就越多，颜色表现就越逼真，相应的数据量越大。例如，位深度为 1 的像素位图只有两个可能的值（黑色和白色），所以又称为二值位图。位深度为 8 的图像有 2^8（即 256）个可能的值。位深度为 8 的灰度模式图像有 256 个可能的灰色值。

RGB图像由三个颜色通道组成。8 位/通道的 RGB 图像中的每个通道有 256 个可能的值，这意味着该图像有 1600 万个以上可能的颜色值。有时将带有 8 位/通道 (bpc) 的 RGB 图像称作 24 位图像（8 位 x 3 通道 = 24 位数据/像素）。通常将使用24位RGB组合数据位表示的的位图称为真彩色位图。

## iOS中的Bitmap

在iOS中，`Bitmap`的数据由`CGImageRef`封装。如果要使用`Bitmap`对图片进行各种处理，就需要先创建位图上下文：
```swift
func createContext(_ image: CGImage) -> CGContext? {
    let w = Int(image.width)
    let h = Int(image.height)
    let bitsPerComponent = 8
    let bytesPerRow = w * 4
    let space = CGColorSpaceCreateDeviceRGB()
    let bitmapInfo = CGImageAlphaInfo.premultipliedLast.rawValue | CGBitmapInfo.byteOrder32Big.rawValue
    let data = UnsafeMutablePointer<UInt32>.allocate(capacity: w * h)
    let context = CGContext(data: data,
                            width: w,
                            height: h,
                            bitsPerComponent: bitsPerComponent,
                            bytesPerRow: bytesPerRow,
                            space: space,
                            bitmapInfo: bitmapInfo)
    return context
}
```

参数说明注释：
```swift
extension CGContext {
    /// - Parameters:
    ///   - data: 用于存放位图的点阵数组指针，当生成上下文并调用上下文的`draw`方法将指定图片或层级绘制到上下文后，data指针就会存储该图片的位图像素信息；我们可以对data里面的数据进行操作，通过创建`CGDataProvider`实例并调用`CGImage`的初始化方法重新生成图片
    ///   - width: 位图宽
    ///   - height: 位图高
    ///   - bitsPerComponent: 每个颜色组件或alpha组件占用的`bite`值，对于32位图像值为8
    ///   - bytesPerRow: 位图每一行占的字节数，对于32位图像一个像素有4`byte`(rgba)，值为w * 4
    ///   - space: 颜色空间，RGBA -> CGColorSpaceCreateDeviceRGB(), CMYK -> CGColorSpaceCreateDeviceCMYK(), 灰度 -> CGColorSpaceCreateDeviceGray()
    ///   - bitmapInfo: 一个常量，通常多枚举做或运算，描述位图上下文所对应的基本信息，具体见`CGBitmapInfo`和`CGImageAlphaInfo`描述
    @available(iOS 2.0, *)
    public /*not inherited*/ init?(data: UnsafeMutableRawPointer?, width: Int, height: Int, bitsPerComponent: Int, bytesPerRow: Int, space: CGColorSpace, bitmapInfo: UInt32)
}
```

### CGBitmapInfo 和 CGImageAlphaInfo

``` swift
public struct CGBitmapInfo : OptionSet {
    public init(rawValue: UInt32)
    public static var alphaInfoMask: CGBitmapInfo { get }
    public static var floatInfoMask: CGBitmapInfo { get }
    public static var floatComponents: CGBitmapInfo { get }    
    public static var byteOrderMask: CGBitmapInfo { get }
    public static var byteOrder16Little: CGBitmapInfo { get } // 16位小端模式
    public static var byteOrder32Little: CGBitmapInfo { get } // 32位小端模式
    public static var byteOrder16Big: CGBitmapInfo { get } // 16位大端模式
    public static var byteOrder32Big: CGBitmapInfo { get } // 32位大端模式
}

public enum CGImageAlphaInfo : UInt32 {
    case none = 0 /* For example, RGB. */
    case premultipliedLast = 1 /* For example, premultiplied RGBA */
    case premultipliedFirst = 2 /* For example, premultiplied ARGB */
    case last = 3 /* For example, non-premultiplied RGBA */
    case first = 4 /* For example, non-premultiplied ARGB */
    case noneSkipLast = 5 /* For example, RBGX. */
    case noneSkipFirst = 6 /* For example, XRGB. */
    case alphaOnly = 7 /* No color data, alpha data only */
}
```

上面就是对于`CGBitmapInfo`和`CGImageAlphaInfo`的定义。

我们先关注`CGImageAlphaInfo`，像素构成类型：
- `last`和`first`分别代表`alpha`的分量在前置位还是后置位，在数据读取时是按照`RGBA`还是`ARGB`来读取的。
- `premultiplied`代表预乘透明，比如对应`RGBA`为(0.5, 0.5, 0.5, 0.5)的色值，在预乘之后变为(0.25, 0.25, 0.25, 0.5)；所以带有`premultiplied`时，说明在图片解码压缩的时候，就将`alpha`通道的值分别乘到了颜色分量上，我们知道`alpha`就会影响颜色的透明度，我们如果在压缩的时候就将这步做完了，那么渲染的时候就不必再处理`alpha`通道了，这样在显示位图的时候直接显示就行了，这样就提高了性能。
- `noneSkip`就代表了虽然有`alpha`分量，但是忽略`alpha`的值，相当于透明度不起作用；这样的话，我们可以获取到图片的`RGB`原始值。

然后我们看一下`CGBitmapInfo`，像素存储格式：
- `byteOrder16Big`和`byteOrder32Big`分别代表16和32位的大端模式
   - 大端模式(Big-Endian)：高位字节排放在内存的低地址端，低位字节排放在内存的高地址段
   - `0x12 | 0x34 | 0x56 | 0x78`（低到高）
   - 对于`RGBA`在内存中的存放则是为`R | G | B | A`
- `byteOrder16Little`和`byteOrder32Little`分别代表16和32位的小端模式
   - 小端模式(Little-Endian)：低位字节排放在内存的低地址端，高位字节排放在内存的高地址段
   - `0x78 | 0x56 | 0x34 | 0x12`（低到高）
   - 对于`RGBA`在内存中的存放则是为`A | B | G | R`

## 代码实践

### 获取图片中点击位置的颜色

- 获取`ImageView`控件的`bounds`作为像素范围大小
- 通过`CGPoint`参数计算出该点对应与像素索引位置的像素数据
- 逐个字节解析出rgb颜色分量和alpha分量值最后生成UIColor最为结果返回
```swift
extension UIImageView {
    /// 工具坐标点获取颜色
    func color(for point: CGPoint) -> UIColor? {
        guard let pixels = pixels,
              let index = indexForPixel(point)
        else { return nil }
        return color(for: pixels[index])
    }

    /// 解析像素点颜色
    func color(for pixel: UInt32) -> UIColor {
        let a = CGFloat(pixel & 0xff) / 255.0
        let r = CGFloat((pixel >> 8) & 0xff) / 255.0
        let g = CGFloat((pixel >> 16) & 0xff) / 255.0
        let b = CGFloat((pixel >> 24) & 0xff) / 255.0
        return UIColor(red: r, green: g, blue: b, alpha: a)
    }
    
    /// 获取对应像素点索引
    func indexForPixel(_ point: CGPoint) -> Int? {
        guard bounds.contains(point) else { return nil }
        return Int(bounds.width) * Int(point.y) + Int(point.x)
    }
    
    /// 获取图片像素数据
    var pixels: [UInt32]? {
        guard let image = image?.cgImage else { return nil }
        let w = Int(bounds.width)
        let h = Int(bounds.height)
        let bitsPerComponent = 8
        let bytesPerRow = w * 4
        let space = CGColorSpaceCreateDeviceRGB()
        let bitmapInfo = CGImageAlphaInfo.premultipliedFirst.rawValue | CGBitmapInfo.byteOrder32Big.rawValue
        var data = [UInt32](repeating: 0, count: w * h)
        let context = CGContext(data: &data, width: w, height: h, bitsPerComponent: bitsPerComponent, bytesPerRow: bytesPerRow, space: space, bitmapInfo: bitmapInfo)
        context?.draw(image, in: bounds)
        return data
    }
}
```

注意点：
- 这边简单操作，真实过程需要判断`ImageView`的`ContentMode`模型，根据不同的模式判断图片在视图中的位置
- 获取像素数据时机中可以做缓存，没有必要每次都去重新读取
- 索引处理先`Int`转换后再计算，否则会有偏差

### 获取图片中点击位置的颜色(另一种方法)

- 生成只容纳一像素的上下文
- 根据点击位置对上下文进行平移并绘制，然后读取对应颜色
```swift
extension UIImageView {
    func color2(for point: CGPoint) -> UIColor? {
        let w = 1
        let h = 1
        let bitsPerComponent = 8
        let bytesPerRow = w * 4
        let space = CGColorSpaceCreateDeviceRGB()
        let bitmapInfo = CGImageAlphaInfo.premultipliedFirst.rawValue | CGBitmapInfo.byteOrder32Big.rawValue
        var data = [UInt32](repeating: 0, count: w * h)
        guard let context = CGContext(data: &data, width: w, height: h, bitsPerComponent: bitsPerComponent, bytesPerRow: bytesPerRow, space: space, bitmapInfo: bitmapInfo)
        else { return nil }
        context.translateBy(x: -point.x, y: -point.y)
        layer.render(in: context)
        guard let component = data.first else { return nil }
        return color(for: component)
    }
}
```

### 设置图片透明度
```swift
extension UIImage {
    func setAlpha(_ a: CGFloat) -> UIImage? {
        UIGraphicsBeginImageContextWithOptions(size, false, 0)
        defer { UIGraphicsEndImageContext() }
        guard let cgImage = cgImage,
              let context = UIGraphicsGetCurrentContext()
        else { return self }
        context.scaleBy(x: 1, y: -1)
        context.translateBy(x: 0, y: -size.height)
        context.setBlendMode(.multiply)
        context.setAlpha(a)
        context.draw(cgImage, in: .init(origin: .zero, size: size))
        return UIGraphicsGetImageFromCurrentImageContext()
    }
}
```

### 获取图片格式

```swift
extension Data {
    enum ImageFormat {
        case unknown
        case PNG
        case JPEG
        case GIF
        
        struct HeaderData {
            static var PNG: [UInt8] = [0x89, 0x50, 0x4E, 0x47, 0x0D, 0x0A, 0x1A, 0x0A]
            static var JPEG_SOI: [UInt8] = [0xFF, 0xD8]
            static var JPEG_IF: [UInt8] = [0xFF]
            static var GIF: [UInt8] = [0x47, 0x49, 0x46]
        }
    }
    
    var imageFormat: ImageFormat {
        guard count > 8 else { return .unknown }
        
        var buffer = [UInt8](repeating: 0, count: 8)
        copyBytes(to: &buffer, count: 8)
        
        if buffer == ImageFormat.HeaderData.PNG {
            return .PNG
            
        } else if buffer[0] == ImageFormat.HeaderData.JPEG_SOI[0],
                  buffer[1] == ImageFormat.HeaderData.JPEG_SOI[1],
                  buffer[2] == ImageFormat.HeaderData.JPEG_IF[0] {
            return .JPEG
            
        } else if buffer[0] == ImageFormat.HeaderData.GIF[0],
                  buffer[1] == ImageFormat.HeaderData.GIF[1],
                  buffer[2] == ImageFormat.HeaderData.GIF[2] {
            return .GIF
        }
        
        return .unknown
    }
}
```

# 总结

本篇内容就是有关`Bitmap`相关的算法与位图相关的知识，也是作为一个总结归纳和知识梳理，也同时收获到了很多新的思路。
以上。