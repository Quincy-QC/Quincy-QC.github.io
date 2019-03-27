---
title: "重构 AppDelegate"
catalog: true
toc_nav_num: true
date: 2019-02-15 18:40:24
subtitle: "Refactoring Massive App Delegate"
header-img: "/img/article_header/article_header.png"
busuanzi: true
tags:
- iOS - Swift

---

> 之前曾经阅读过一篇基于 Swift 重构 AppDelegate 的文章， 现在把他整理搬到自己的博客，翻译自 [@V8tr](http://www.vadimbulavin.com/refactoring-massive-app-delegate/)，[Demo](https://github.com/Quincy-QC/MassiveAppDelegate)

# Introduction

AppDelegate 连接着我们的应用与系统，通常被认为是每个 iOS 工程的核心。正常趋势是随着开发逐渐变大，逐渐添加新的功能与职责，然后到处被调用，最终变成一团乱码。
把所有东西都放在 AppDelegate 的代价是巨大的，会影响应用的流畅性，所以，保证这个类简单明了对一个健康的应用是非常重要的。
本文将介绍几种不同的方式帮助我们如何使 AppDelegates 更加简单，可复用，可测试。

# Problem Statement

AppDelegate 是应用的根对象，他保证了应用与正确的与系统或其他应用的对接，所以 AppDelegate 有更加繁重的任务使得它难以修改，扩展，测试。
甚至[苹果](https://developer.apple.com/documentation/uikit/uiapplicationdelegate?language=objc)都推荐你在 AppDelegate 添加三个以上的任务。
通过调查[open-source-ios-apps](https://github.com/dkhamsing/open-source-ios-apps)我制作了一份经常会被放在 AppDelegates 里面任务的列表，我很确定我们会写同样的代码或在项目中有着相同的混乱。
- 初始化三方类库
- 初始化或操作数据存储系统
- 用于单元测试或 UI 测试的应用状态配置
- 配置 UserDefaults：设置首次启动的标识，加载存储数据
- 配置通知：请求权限，存储 token，处理自定义事件，通知到应用的其他部分
- 配置 UIAppearance
- 配置应用角标
- 配置后台任务
- 配置界面：初始化控制器选择与跳转
- 播放音频
- 配置统计分析
- 打印调试日志
- 配置设备旋转方向
- 实现代理方法，尤其是来自三方类库的代理
- 弹窗

我敢肯定这个列表还不够完整。
显然，如此臃肿的 AppDelegate 作为这篇博客的反面教材，难以支持扩展和测试，我们把它称为 Massive App Delegates。

# Solution

在我们意识到 AppDelegate 的臃肿后，我们来看下几种可能的解决方案或者叫他 ’recipes‘。
每个 recipe 必须满足下面条件：
- 遵循 [sigle responsibility principle](https://en.wikipedia.org/wiki/Single_responsibility_principle)
- 易于扩展
- 易于测试

## Recipe #1: Command Design Pattern

命令设计模式（command Design pattern）指多个代表单一事件功能的对象集，这些对象把触发自己事件所需的所有参数封装，因此命令的调用者不知道这个命令的实现和响应者。
为每个 AppDelegate 职责定义一个命令，名字根据功能决定。

``` Swift
protocol Command {
    func execute()
}

struct InitializeThirdPartiesCommand: Command {
    func execute() {
        print("Third parties are initialized")
    }
}

struct InitialViewControllerCommand: Command {
    let keyWindow: UIWindow
    
    func execute() {
        print("Pick root view controller here")
        keyWindow.frame = UIScreen.main.bounds
        keyWindow.backgroundColor = UIColor.white
        keyWindow.makeKeyAndVisible()
        keyWindow.rootViewController = ViewController()
    }
}

struct InitializeAppearanceCommand: Command {
    func execute() {
        print("Setup UIAppearance")
    }
}

struct RegisterToRemoteNotificationsCommand: Command {
    func execute() {
        print("Register for remote notifications here")
    }
}
```
然后我们定义**StartupCommandsBuilder**把命令的创建细节封装，AppDelegate 调用这个类初始化对应的命令然后调用，
``` Swift
// MARK: -------- StartupCommandsBuilder --------

final class StartupCommandsBuilder {
    private var window: UIWindow!
    
    func setKeyWindow(_ window: UIWindow) -> StartupCommandsBuilder {
        self.window = window
        return self
    }
    
    func build() -> [Command] {
        return [InitializeThirdPartiesCommand(),
                InitialViewControllerCommand(keyWindow: window),
                InitializeAppearanceCommand(),
                RegisterToRemoteNotificationsCommand()]
    }
}

// MARK: -------- AppDelegate --------

@UIApplicationMain
class AppDelegate: UIResponder, UIApplicationDelegate {

    var window: UIWindow?


    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        
        StartupCommandsBuilder()
            .setKeyWindow(window!)
            .build()
            .forEach { $0.execute() }
        
        return true
    }
}
```
新命令可以不需要修改 AppDelegate 直接添加到**StartupCommandsBuilder**。
这个解决方案满足定义的条件：
- 每个命令有自己的职责
- 无需修改 AppDelegate 的代码就可以方便添加新命令
- 命令易于测试

## Recipe #2: Composite Design Pattern

复合设计模式（Composite Design Pattern）允许将对象的层级结构视为单例。举个例子就像 iOS 里的**UIView**和它的子类视图。
主体思想是复合类和子 AppDelegate 各自拥有职责，同时复合类将所有方法传递到子类。
``` Swift
typealias AppDelegateType = UIResponder & UIApplicationDelegate

class CompositeAppDelegate: AppDelegateType {
    private let appDelegates: [AppDelegateType]
    
    init(appDelegates: [AppDelegateType]) {
        self.appDelegates = appDelegates
    }
    
    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey : Any]? = nil) -> Bool {
        appDelegates.forEach { _ = $0.application?(application, didFinishLaunchingWithOptions: launchOptions) }
        return true
    }
    
    func application(_ application: UIApplication, didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data) {
        appDelegates.forEach { $0.application?(application, didRegisterForRemoteNotificationsWithDeviceToken: deviceToken) }
    }
}
```
然后，子 AppDelegates 实现真正的任务，
``` Swift
class PushNotificationAppDelegate: AppDelegateType {
    func application(_ application: UIApplication, didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data) {
        print("Registed successfully")
    }
}

class StartupConfiguratorAppDelegate: AppDelegateType {
    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey : Any]? = nil) -> Bool {
        print("Perform startup configurations, e.g. build UI stack, setup UIAppearance")
        return true
    }
}

class ThirdPartiesConfiguratorAppDelegate: AppDelegateType {
    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey : Any]? = nil) -> Bool {
        print("Setup third parties")
        return true
    }
}
```
我们定义**AppDelegateFactory**封装创建逻辑，主 AppDelegate 通过这个工厂类创建组合代理，调用所有的方法，
``` Swift
enum AppDelegateFactory {
    static func makeDefault() -> AppDelegateType {
        return CompositeAppDelegate(appDelegates: [PushNotificationAppDelegate(), StartupConfiguratorAppDelegate(), ThirdPartiesConfiguratorAppDelegate()])
    }
}

@UIApplicationMain
class AppDelegate: UIResponder, UIApplicationDelegate {

    var window: UIWindow?
    let appDelegate = AppDelegateFactory.makeDefault()

    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        
        _ = appDelegate.application?(application, didFinishLaunchingWithOptions: launchOptions)
        
        return true
    }
    
    func application(_ application: UIApplication, didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data) {
        
        appDelegate.application?(application, didRegisterForRemoteNotificationsWithDeviceToken: deviceToken)
    }
}
```
这个方法实现了我们开始定义的条件：
- 每个子 AppDelegate 有个自己的职责
- 无需修改 AppDelegate 的代码就可以方便添加新代理
- 易于测试

## Recipe #3: Mediator Design Pattern

中介者设计模式（Mediator Design Pattern）通过一个隐蔽无约束的方法封装交互策略。对象被中介者无意识操作，无需他们的许可情况下就可以在幕后默默地推行自己的政策。
如果你想要更多学习这个模式，推荐你阅读 [Mediator Pattern Case Study](http://www.vadimbulavin.com/mediator-pattern-case-study/)
定义**AppLifecycleMediator**将**UIApplication**生命周期活动传递到底层监听器，监听器必须遵循必要时可被扩展的**AppLifecycleListener**协议
``` Swift
protocol AppLifecycleListener {
    func onAppWillEnterForeground()
    func onAppDidEnterBackground()
    func onAppDidFinishLaunching()
}

extension AppLifecycleListener {
    func onAppWillEnterForeground() {}
    func onAppDidEnterBackground() {}
    func onAppDidFinishLaunching() {}
}

struct listen1: AppLifecycleListener {
    func onAppDidFinishLaunching() {
        print("app did finish launching1")
    }
    
    func onAppWillEnterForeground() {
        print("app will enter foreground")
    }
}

struct listen2: AppLifecycleListener {
    func onAppDidEnterBackground() {
        print("app did enter background")
    }
}

struct listen3: AppLifecycleListener {
    func onAppDidFinishLaunching() {
        print("app did finish launching2")
    }
}

class AppLifecycleMediator: NSObject {
    private let listeners: [AppLifecycleListener]
    
    init(listeners: [AppLifecycleListener]) {
        self.listeners = listeners
        super.init()
        subscribe()
    }
    
    deinit {
        NotificationCenter.default.removeObserver(self)
    }
    
    private func subscribe() {
        NotificationCenter.default.addObserver(self, selector: #selector(onAppWillEnterForeground), name: UIApplication.willEnterForegroundNotification, object: nil)
        NotificationCenter.default.addObserver(self, selector: #selector(onAppDidEnterBackground), name: UIApplication.didEnterBackgroundNotification, object: nil)
        NotificationCenter.default.addObserver(self, selector: #selector(onAppDidFinishLaunching), name: UIApplication.didFinishLaunchingNotification, object: nil)
    }
    
    @objc func onAppWillEnterForeground() {
        listeners.forEach { $0.onAppWillEnterForeground() }
    }
    
    @objc func onAppDidEnterBackground() {
        listeners.forEach { $0.onAppDidEnterBackground() }
    }
    
    @objc func onAppDidFinishLaunching() {
        listeners.forEach { $0.onAppDidFinishLaunching() }
    }
}

extension AppLifecycleMediator {
    static func makeDefaultMediator() -> AppLifecycleMediator {
        return AppLifecycleMediator(listeners: [listen1(), listen2(), listen3()])
    }
}
```
现在就可以通过一行代码添加到**AppDelegate**
``` Swift
@UIApplicationMain
class AppDelegate: UIResponder, UIApplicationDelegate {

    var window: UIWindow?

    let mediator = AppLifecycleMediator.makeDefaultMediator()

    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {

        return true
    }
}
```
中介者自动订阅所有事件，**AppDelegate**仅需要初始化让他工作。
这种方式满足我们开始定义的条件：
- 每个监听者有自己的职责
- 无需修改 AppDelegate 代码就可以添加新监听者
- 每个模块都易于测试

# Summary
大多数**AppDelegate**都过于庞大，过于复杂，担有太多职责，通过软件设计模式，**AppDelegate**可以被划分为多个类，各自拥有各自的职责，同时方便测试。这样的代码非常容易修改，而且非常灵活，可以在将来提取和复用。

# Reference
[Refactoring Massive App Delegate](http://www.vadimbulavin.com/refactoring-massive-app-delegate/)