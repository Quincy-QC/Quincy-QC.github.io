---
title: "使用Swift Package Manager管理依赖项"
catalog: true
toc_nav_num: true
date: 2019-11-22 19:23:09
subtitle: "Swift Package Manager"
header-img: "/img/article_header/article_header.png"
busuanzi: true
tags:
- iOS

---

# 前言

上一篇文章讲解了使用Cocoapods进行包管理的一些基本操作，讲到这里就不得不说刚刚集成于Xcode 11中支持iOS平台等多平台的包管理工具--Swift Package Manager。

# 在项目中使用加载

进入添加包依赖界面有两种方式，如下图：

![添加包依赖1](/img/article/20191122/1.png)
![添加包依赖2](/img/article/20191122/2.png)

然后可以进行包搜索，需要登录Git账号，可以在`Accounts`中进行设置。

搜索完成后可以进行版本选择（版本号、分支、Commit）：

![添加包依赖3](/img/article/20191122/3.png)

安装完成后就可以在项目中使用，非常方便。

# 上传自己的SPM库

先进行包初始化，创建并进入文件夹`mkdir ProjName && cd ProjName`，使用`swift package init`命令进行包初始化。

初始化后会生成一系列文件：
- README.md：不用解释，用于展示并介绍包的文件；
- Package.swift：包配置文件，后面会详细讲解；
- Sources文件夹：存放具体代码的文件夹；
- Tests文件夹：存放各测试用例的文件夹。

包配置文件：
``` swift
// swift-tools-version:5.1
// The swift-tools-version declares the minimum version of Swift required to build this package.

import PackageDescription

let package = Package(
    name: "MyCategory",
    platforms: [
        .macOS(.v10_13), .iOS(.v11), .tvOS(.v11),
    ],
    products: [
        // Products define the executables and libraries produced by a package, and make them visible to other packages.
        .library(
            name: "MyCategory",
            targets: ["MyCategory"]),
    ],
    dependencies: [
        // Dependencies declare other packages that this package depends on.
        // .package(url: /* package url */, from: "1.0.0"),
        .package(url: "https://github.com/onevcat/Kingfisher.git", from: "5.8.3")
    ],
    targets: [
        // Targets are the basic building blocks of a package. A target can define a module or a test suite.
        // Targets can depend on other targets in this package, and on products in packages which this package depends on.
        .target(
            name: "MyCategory",
            dependencies: ["Kingfisher"]),
        .testTarget(
            name: "MyCategoryTests",
            dependencies: ["MyCategory"]),
    ]
)
```
中间的很多内容与Cocoapods配置文件相同，也有对应的参数，非常容易理解。
添加外部依赖在`dependencies`添加对应的Git路径，并在`targets`中声明。

对于`Sources`文件夹下的代码文件，有一点要说一下。因为SPM是跨平台包管理器，所以在引入系统库时需要进行平台判断，否则会在编译时无法通过：
``` swift
#if os(Linux)

// Code specific to Linux

#elseif os(macOS)

// Code specific to macOS

#endif

#if canImport(UIKit)

// Code specific to platforms where UIKit is available

#endif
```

上述文件配置结束后，和Cocoapods一样上传到Git并打Tag生成最终的Release包，然后就可以在SPM中搜索到我们的包啦。

# 对于已有项目

对于已有项目，或者说是Cocoapods生成的包管理文件，我们可以同样创建SPM，在目录中添加`Package.swift`和`Sources`文件即可。

# 总结

总的来说，Swift Package Manager还是非常方便的，就这样。