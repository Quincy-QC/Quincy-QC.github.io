---
title: "RealityKit - 实景AR"
catalog: true
toc_nav_num: true
date: 2021-08-30 20:19:02
subtitle: "RealityKit AR with the Device's Camera"
header-img: "/img/article_header/article_header.png"
busuanzi: true
tags:
- iOS - Swift

---

# 前言

有了[上一篇](/article/2021/realitykit-0621)对于`RealityKit`的基础概念介绍，现在我们直接对于实景AR水平平面放置物品进行一个简单实现。

# 初始化

## ARView

我们首先需要创建一个通过设备相机创建的现实场景，可以通过创建一个`arView`属性，同时设置`cameraMode`为`ar`（这边是个枚举，如果选择`nonAr`就是一个使用虚拟相机的场景）：
``` swift
private lazy var arView: ARView = {
    let vi = ARView(frame: view.bounds, 
                    cameraMode: .ar, 
                    automaticallyConfigureSession: true)
    return vi
}()
```
同时为了不必要的渲染动作影响性能，可以对`renderOptions`进行配置：
``` swift
...
    vi.renderOptions = [.disableCameraGrain, 
                        .disableHDR, 
                        .disableMotionBlur,
                        .disableDepthOfField,
                        .disableFaceOcclusions,
                        .disablePersonOcclusion, 
                        .disableGroundingShadows, 
                        .disableAREnvironmentLighting]
...
```
同时为了后期调试的方便，可以对应的打开debug模式，一般开一个世界坐标系就可以：
``` swift
...
    vi.debugOptions = [.showWorldOrigin]
...
```
然后将`arView`添加到对应控制器中的视图就可以：
``` swift
override func viewDidLoad() {
    super.viewDidLoad()

    view.addSubview(arView)
}
```

## ARCoachingOverlayView

这边我们使用一个系统提供的AR引导动画，对它进行一个初始化：
``` swift
private lazy var coachingOverlay: ARCoachingOverlayView = {
    let c = ARCoachingOverlayView(frame: arView.bounds)
    c.session = arView.session
    arView.addSubview(c)
}()
```
其实`ARCoachingOverlayView`可以直接对于平面识别后提供一个代理的回调，但是对于反复识别不够灵敏且使用上不够灵活，所以我们在设置识别平面后，会把自动识别到平面后消失的状态关闭，由我们手动控制，所以这边代理可实现可不实现（代理是针对`ARCoachingOverlayView`是否处于活跃状态的回调，由于我们自己手动控制它的状态，所以这些状态不一定要通过这个回调），但我们实现看下他的相关方法：
``` swift
...
    c.delegate = self
    c.goal = .horizontalPlane
    c.activatesAutomatically = false
...
```
相关代理：
``` swift
extension UIViewController: ARCoachingOverlayViewDelegate {

    /// coachingOverlayView即将处于识别活跃状态
    func coachingOverlayViewWillActivate(_ coachingOverlayView: ARCoachingOverlayView) { }
    
    /// coachingOverlayView已经处于取消识别活跃状态
    func coachingOverlayViewDidDeactivate(_ coachingOverlayView: ARCoachingOverlayView) { }

    /// coachingOverlayView点击重置的回调
    func coachingOverlayViewDidRequestSessionReset(_ coachingOverlayView: ARCoachingOverlayView) { }
}
```
这边是简单实现，所以我们在进入页面时就可以开始进行一个识别：
``` swift
...
    c.setActive(true, animated: true)
...
```
然后添加到`arView`上：
``` swift
...
    arView.addSubview(coachingOverlay)
...
```

## 效果图

这时候我们可以看下效果图：
![效果图](/img/article/20210830/1.png)

# 平面检测

现在我们需要对于现实世界进行一个平面检测，主要通过`ARSession`，`ARSession`无需我们自己创建，就是从`ARView`中直接获取，先把他声明下，方便使用：
``` swift
private var session: ARSession {
    arView.session
}
```
然后对`session`进行一些配置，同时识别水平平面；这边我们创建的`ARWorldTrackingConfiguration`，它内部还有很多其他配置（如图片识别、人脸识别等等）：
``` swift
private func setupARSession() {
    let configuration = ARWorldTrackingConfiguration()
    configuration.planeDetection = [.horizontal]
    configuration.environmentTexturing = .automatic
    session.run(configuration, options: [])
}
```
一个关键点，`session`的代理，在代理里面我们就可以在现实世界检测平面并做出回调：
``` swift
...
    session.delegate = self
...
```
相关代理：
``` swift
extension UIViewController: ARSessionDelegate {
    
    /// 每帧调用
    func session(_ session: ARSession, didUpdate frame: ARFrame) { }
    
    /// 新锚点添加时调用(系统会自动识别到平面时添加锚点)
    func session(_ session: ARSession, didAdd anchors: [ARAnchor]) { }
    
    /// 锚点更新时调用
    func session(_ session: ARSession, didUpdate anchors: [ARAnchor]) { }

    /// 锚点移除时调用
    func session(_ session: ARSession, didRemove anchors: [ARAnchor]) { }
}
```
根据上面的回调，其实我们只要用第二个回调方法，在添加水平平面的锚点后，在该锚点位置添加我们的模型就可以了，所以，我们先看下如何创建一个模型（这边只做简单demo，如果想要复杂化，比如检测到平面不立即添加模型，而是有个选择框让用户手动点击添加，可以使用第一个方法，每帧调用的回调，在屏幕上渲染一个选择框，并实时检测是否存在平面）。

# 模型创建与放置

## 自定义Entity类
这边自定义一个`AnchorEntity`类，继承`Entity`和`HasAnchoring`两个协议，同时内部初始化异步加载一个`ModelEntity`，放置卡线程，异步方法内部实现是`Combine`框架（这边的模型直接使用苹果官方推荐的`USDZ`模型格式，其他还有`OBJ`格式的模型，但这种格式的模型和纹理是两个文件，在渲染合成后存在内存问题，虽然有解决方案，但感觉不是很完美）：
``` swift
class ShoeEntity: Entity, HasAnchoring {
    private var cancels: Set<AnyCancellable> = .init()
    
    required init(name: String) {
        super.init()
        
        loadEntity(name: name)
    }
    
    private func loadEntity(name: String) {
        Entity.loadModelAsync(named: name).sink { error in
            debugPrint(error)
        } receiveValue: { [weak self] entity in
            self?.addChild(entity)
        }.store(in: &cancels)
    }
    
    required init() {
        fatalError("init() has not been implemented")
    }
}
```
我们定义一个模型属性：
```
private lazy var shoeEntity: ShoeEntity = .init(name: "shoe.usdz")
```
## 添加模型并展示
模型已经有了，然后就是在检测到锚点时关闭引导动画并展示我们的模型，，基础功能就完成了：
``` swift
func session(_ session: ARSession, didAdd anchors: [ARAnchor]) {
    guard let anchor = anchors.filter({ $0 is ARPlaneAnchor }).first as? ARPlaneAnchor,
          anchor.alignment == .horizontal else {
        return
    }
    coachingOverlay.setActive(false, animated: true)
    shoeEntity.transform.matrix = anchor.transform
    arView.scene.addAnchor(shoeEntity)
}
```

# 效果视频

![效果视频](/img/article/20210830/2.mp4)

# 总结

以上就是一个简单的真实世界识别水平平面并展示模型的一个流程，在整理文章的过程中也让我对`RealityKit`有了更深的理解，把他更为精简的实现。
后续的话，还有更多的细节可以优化，包括光照，手势，位置修正等等，有时间再更。