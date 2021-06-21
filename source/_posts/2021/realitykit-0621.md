---
title: "RealityKit基础概念"
catalog: true
toc_nav_num: true
date: 2021-06-21 19:36:20
subtitle: "RealityKit Experience"
header-img: "/img/article_header/article_header.png"
busuanzi: true
tags:
- iOS - Swift

---

# 前言

最近Apple刚好在`WWDC21`上发布了[Object Capture](https://developer.apple.com/documentation/realitykit/capturing_photographs_for_realitykit_object_capture/)，基于这个契机，我们就想用苹果提供生成的模型加上最新的`RealityKit`框架进行一次实景AR开发，这篇文章就简单介绍`RealityKit`的基础概念。

# 介绍

`RealityKit`框架专门为增强现实量身定制，能够提供逼真的图像渲染、相机特效、动画、物理特效等等。借助原生`ARKit`整合、基于物理的超逼真渲染、变换和骨骼动画、空间音频和刚体物理，`RealityKit`可以比以往更加快速轻松地进行增强现实开发。

# 基础概念

如下图`ARView`的结构层次：
![RealityKit](/img/article/20210621/1.png)

首先`ARView`就是用于展示AR体验的视图，可以将它添加到父视图中，通过初始化`cameraMode`的不同，可以确定是否需要使用相机生成真实的AR场景还是虚拟场景，这个比较简单。看下API：
``` swift
public init(frame frameRect: CGRect, cameraMode: ARView.CameraMode, automaticallyConfigureSession: Bool)
```

其次是`Scene`就是保存AR视图呈现的容器，我们不需要直接创建，可以通过`ARView·`获取到`scene`这个熟悉，后续我们的模型都是直接或者间接添加到这个`scene`上。

然后就是`Entity`了，这里有`Anchor Entity`，`Model Entity`，其实本质都是继承与`Entity`，同时，如果我们想要自定义一些`Entity`，我们可以通过继承`Entity`，同时实现相应的协议，同时加载相应的`Component`，就可以创建具有不同外观和行为的实体，如下图：
![RealityKit](/img/article/20210621/2.png)

这边又有两个概念，一个是协议，一个是`Component`，其实概括来讲就是协议让`Entity`拥有了某项能力，而`Component`则是用来真正装备该能力的组件修饰，所以，如果我想要一个`Entity`拥有外观属性（大小、颜色），那我们我们就需要继承`HasModel`这个协议，同时配置`Component`这个属性赋予这个实体真正的外观属性，后面再深入介绍。

苹果默认提供了两个基础`Entity`，一个是`Anchor Entity`，一个是`Model Entity`，现在我们就来具体介绍这两个`Entity`。

# Entity

一个基类`Entity`默认实现了`HasTransform`和`HasSynchronization`协议，拥有`Transform component`与`Synchronization component`两个组件，分别用于空间转换（大小、位移，旋转）与联网时的同步操作，同时还实现了`HasHierarchy`协议，允许所有的`Entity`拥有层级关系（父实体，子实体）。

## AnchorEntity

`Anchor Entity`默认实现了`HasAnchoring`协议，实现该协议之后就是允许该实体能够将该虚拟内容渲染到真实世界，我们一般用来把他作为基础实体，然后后续具体的实体添加到这个`Anchor Entity`上，后续添加到`ARView`上继续渲染，可以把他当成一个载体，本身并没有形状或者外观。

## ModelEntity

另一个默认提供的实体是`Model Entity`，实现了`HasModel`, `HasPhysics`两个协议。
- `HasModel`这个协议本身就继承于`HasTransform`协议，他针对与`HasTransform`协议，又增加了一个`Model component`组件，允许设置该实体的外观大小包括纹理等信息。
- `HasPhysics`这个协议是`HasPhysicsBody`与`HasPhysicsMotion`协议的总和。
    - `HasPhysicsBody`又继承于`HasCollision`，`HasCollision`又继承于`HasTransform`。`HasCollision`增加的组件特性就是碰撞体积与控制模式（是否需要记录细节碰撞信息）；`HasPhysicsBody`新增的组件特性是允许对该物体施加外力并进行运动。
    - `HasPhysicsMotion`新增的组件特性是基础运动模式，能够在创建后无需外力就保持一个持续的运动，同时判断是否允许外界改变行为；

## 自定义Entity

所以，基于上面的介绍，我们自定义`Entity`也就很简单了，只要针对于我们的场景，选择不同的协议与添加对于的`Component`就可以得到我们定制的实体，用于展示，非常Swift。

# 简单实现

那么我们就来简单实现一个非AR场景下的Box实体，并增加一个自旋动作。
首选我们就自定义一个实体，有以下需求：
- 直接添加到`Scene`上，需要有`Anchor Entity`的锚点添加特性，所以继承`HasAnchoring`协议
- 需要自定义一个形状box和颜色，所以继承`HasModel`协议
- 需要增加一个自旋匀速运动，所以继承`HasPhysics`协议

``` swift
class Box: Entity, HasAnchoring, HasModel, HasPhysics {
    required init(world: SIMD3<Float>) {
        super.init()
        
        position = world

        // 设置大小，颜色，形状
        model = ModelComponent(
            mesh: .generateBox(size: 0.5),
            materials: [SimpleMaterial(color: .cyan, roughness: 0.5, isMetallic: true)]
        )

        // 设置碰撞体积
        collision = CollisionComponent(
            shapes: [.generateBox(size: [0.5, 0.5, 0.5])],
            mode: .default,
            filter: .default
        )

        // 设置物理模型大小，密度，运动模式(非动态)
        physicsBody = {
            var c = PhysicsBodyComponent(
                shapes: [ShapeResource.generateBox(size: [0.5, 0.5, 0.5])],
                density: 0.95,
                material: .generate(),
                mode: .kinematic
            )
            return c
        }()
        
        // 给定y方向恒定角速度转动
        physicsMotion = PhysicsMotionComponent(linearVelocity: [0, 0, 0], angularVelocity: [0, 1, 0])
    }
    
    required init() {
        fatalError("init() has not been implemented")
    }
}
```
这样，一个简单的自定义实体就已经定制完了。

然后创建我们的`ARView`，添加到主视图，并通过`ARView`获取的`Scene`添加`Box`实例，然后就能肯定一个沿着y轴恒定速度旋转的盒子了。
代码如下：

``` swift
class ViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad() 

        let arView = ARView(frame: view.bounds, cameraMode: .nonAR, automaticallyConfigureSession: false)
        view.addSubview(arView)

        let box = Box(world: [0, 0, 0])
        arView.scene.addAnchor(box)
    }
}
```

效果图：
![RealityKit](/img/article/20210621/3.png)

# 结语

一个简单的旋转模型就实现了，是不是很简单，可以自己尝试下。