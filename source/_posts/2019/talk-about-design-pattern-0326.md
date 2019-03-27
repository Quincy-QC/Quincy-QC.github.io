---
title: 设计模式那点事
catalog: true
toc_nav_num: true
date: 2019-03-26 11:19:05
subtitle: "Talk About Design Pattern"
header-img: "/img/article_header/article_header.png"
busuanzi: true
tags:
- iOS - Objective-C

---

> 设计模式 Design Pattern 是一套被反复使用、多数人知晓的、经过分类编目的、代码设计经验的总结，使用设计模式是为了代码可重用性、让代码更容易被理解、保证代码可靠性。

# 为什么要学习设计模式

- 设计模式来源于众多专家的经验和智慧，是从许多优秀的软件系统中总结出的成功的、能够实现可维护性复用的设计方案，使用这些方案将可以避免让我们做一些重复性的工作

- 设计模式提供了一条通用的设计词汇和一种通用的形式来方便开发人员之间的沟通和交流，是的设计方案更加通俗易懂

- 大部分设计模式都兼顾了系统的可重用性和可扩展性，这使得我们可以更好地重用一些已有的设计方案、功能模块甚至一个完整的软件系统，避免我们经常做一系重复的设计、编写一些重复的代码

- 合理使用设计模式并对设计模式的使用情况进行文档化，将有助于别人更快的理解系统

- 将有助于初学者更加深入地理解面向对象思想

# 六大设计原则

## 单一职责原则

一个类只负责一件事。
比如说系统为我们实现的 UIView 与 CALayer 这两个类，UIView 专门负责事件传递和事件响应，而 CALayer 专门负责动画以及视图的显现。

## 依赖倒置原则

抽象不应该依赖于具体实现，具体实现可以依赖于抽象。
换言之，要针对接口编程，而不是针对实现编程。在程序代码中传递参数时或在关联关系时，尽量引用层次高的抽象层类，即使用接口和抽象类进行变量声明、参数类型声明、方法返回类型声明，以及数据类型的转换等，而不要用具体类来做这些事情。

## 开闭原则

对修改关闭，对扩展开放。
当我们定义一个类时，应该尽量考虑到这个类的可扩展性和灵活性，在考虑到当前需求的同时需要考虑后续多个版本迭代需求，需要对类的成员变量以及数据结构尽可能在不改动的情况下通过扩展的方式或者提供接口的方式解决后续的需求问题。

## 里氏替换原则

父类可以被子类无缝替换，且原有功能不受任何影响。
比如 KVO 实现原理，当我们对一个对象进行 addObserver 观察的时候，系统为我们在动态运行时创建一个子类，当我们以为我们观察的是我们的父类，其实已经被悄无声息的替换成对应的子类。

## 接口隔离原则

使用多个专门的协议、而不是一个庞大臃肿的协议。
比如我们经常使用的 UITableViewDelegate 与 UITableViewDataSource 这两个协议，UITableViewDelegate 专门负责 UITableView 的代理回调事件，而 UITableViewDataSource 专门负责数据源的获取，协议中的方法应当尽量少，从而避免一个庞大臃肿的协议。

## 迪米特法则

一个对象应当对其他对象有尽可能少的了解。
高内聚，低耦合。通过降低类与类之间的耦合，减少类与类之间的关联程度，可以让类与类之间的协作更加直接，更容易修改和复用。

# 设计模式分类

## 创建型

创建型模式(Creational Pattern)对类的实例化过程进行了抽象，能够将木块中对象的创建和对象的使用分离。为了是结构更加清晰，外界对于这些对象只需要知道它们的共同接口，而不需要知道具体的实现细节，使整个系统的设计更加符合单一职责原则。

- 简单工厂模式
- 工厂方法模式
- 抽象工厂模式
- 单例模式
- 生成器模式
- 原型模式

### 单例

单例模式需要确保某个类有且仅有一个实例，并提供一个访问它的全局访问点。
注意点主要就是他的唯一性。

```objectivec
#pragma mark ----- SingleInstance.h -----

@interface SingleInstance : NSObject

+ (id)sharedInstance;

@end
```

```objectivec
#pragma mark ----- SingleInstance.m -----

@implementation SingleInstance

+ (id)sharedInstance
{
    // 静态局部变量
    static Mooc *instance = nil;
    
    // 通过dispatch_once方式 确保instance在多线程环境下只被创建一次
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        // 创建实例
        instance = [[super allocWithZone:NULL] init];
    });
    return instance;
}

// 重写方法【必不可少】
+ (id)allocWithZone:(struct _NSZone *)zone{
    return [self sharedInstance];
}

// 重写方法【必不可少】
- (id)copyWithZone:(nullable NSZone *)zone{
    return self;
}

@end
```

## 结构型

结构型模式(Structural Pattern)描述如何将类或者对象结合在一起形成更大的结构，可以分为类结构型模式和对象结构性模式：
- 类结构型模式关心类的组合，由多个类可以组合成一个更大的系统，在类机构型模式中一般只存在继承和实现关系。
- 对象结构型模式关心类与对象的组合，通过关联关系使得在一个类中定义另一个类的实例对象，然后通过该对象调用其方法。
结构型模式有以下几类：
- 外观模式
- 适配器模式
- 桥接模式
- 代理模式
- 装饰者模式
- 享元模式

### 桥接

举例：一个关于业务解耦的问题，一个列表需要分别适配几种网络数据类型。
方案：创建两个抽象类 BaseObjectA 与 BaseObjectB，分别解决数据获取和逻辑处理，通过子类(BaseObjectA1..., BaseObjectB1...)完成具体实现，然后将两个类组装，完成业务解耦，同时可以实现多种业务场景需求。
```objectivec
#pragma mark ----- BaseObjectA.h -----

@interface BaseObjectA : NSObject

// 桥接模式的核心实现
@property (nonatomic, strong) BaseObjectB *objB;

// 获取数据
- (void)handle;

@end
```

```objectivec
#pragma mark ----- BaseObjectA.m -----

@implementation BaseObjectA

 /*
    A1 --> B1、B2、B3         3种
    A2 --> B1、B2、B3         3种
    A3 --> B1、B2、B3         3种
  */
- (void)handle
{
    // override to subclass
    
    [self.objB fetchData];
}

@end
```

```objectivec
#pragma mark ----- BaseObjectB.h -----

@interface BaseObjectB : NSObject

- (void)fetchData;

@end
```

```objectivec
#pragma mark ----- BaseObjectB.m -----

@implementation BaseObjectB

- (void)fetchData
{
    // override to subclass
}

@end
```

调用桥接类
```objectivec
#pragma mark ----- BridgeDemo.h -----

@interface BridgeDemo : NSObject

- (void)fetch;

@end
```

```objectivec
#pragma mark ----- BridgeDemo.m -----

@interface BridgeDemo()

@property (nonatomic, strong) BaseObjectA *objA;

@end

@implementation BridgeDemo

/*
 根据实际业务判断使用那套具体数据
 A1 --> B1、B2、B3         3种
 A2 --> B1、B2、B3         3种
 A3 --> B1、B2、B3         3种
 */
- (void)fetch
{
    // 创建一个具体的ClassA
    _objA = [[ObjectA1 alloc] init];
    
    // 创建一个具体的ClassB
    BaseObjectB *b1 = [[ObjectB1 alloc] init];
    // 将一个具体的ClassB1 指定给抽象的ClassB
    _objA.objB = b1;
    
    // 获取数据
    [_objA handle];
}

@end
```

### 适配器



被适配类（不应修改的类）
```objectivec
#pragma mark ----- Target.h -----

@interface Target : NSObject

- (void)operation;

@end
```

```objectivec
#pragma mark ----- Target.m -----

@implementation Target

- (void)operation
{
    // 原有的具体业务逻辑
}

@end
```

适配类
```objectivec
#pragma mark ----- CoolTarget.h -----

// 适配对象
@interface CoolTarget : NSObject

// 被适配对象
@property (nonatomic, strong) Target *target;

// 对原有方法包装
- (void)request;

@end
```

```objectivec
#pragma mark ----- CoolTarget.m -----

@implementation CoolTarget

- (void)request
{
    // 额外处理
    
    [self.target operation];
    
    // 额外处理
}

@end
```

## 行为型

行为型模式(Behavioral Pattern)是对在不同对象之间划分责任和算法的抽象化，更加清晰地划分类与对象的职责，并研究系统在运行时实例对象之间的交互。

- 责任链模式
- 命令模式
- 解释器模式
- 迭代器模式
- 中介者模式
- 备忘录模式
- 观察者模式
- 状态模式
- 策略模式
- 模板方法模式
- 访问者模式

### 责任链

一个类的成员变量中包含这个类（同类型）构成了责任链的基础数据类型。

```objectivec
#pragma mark ----- BusinessObject.h -----

@class BusinessObject;

typedef void (^CompletionBlock)(BOOL handled);
typedef void (^ResultBlock)(BusinessObject * __nullable handler, BOOL handled);

@interface BusinessObject : NSObject

// 下一个响应者
@property (nonatomic, strong) BusinessObject *nextBusiness;

// 响应者的处理
- (void)handle:(ResultBlock)result;

// 各个业务中的具体处理
- (void)handleBusiness:(CompletionBlock)completion;

@end
```

```objectivec
#pragma mark ----- BusinessObject.m -----

@implementation BusinessObject

- (void)handle:(ResultBlock)result {
    CompletionBlock completion = ^(BOOL handled) {
        if (handled) {
            result(self, handled);
        } else {
            if (self.nextBusiness) {
                [self.nextBusiness handle:result];
            } else {
                result(nil, NO);
            }
        }
    };
    [self handleBusiness:completion];
}


- (void)handleBusiness:(CompletionBlock)completion {
    // 业务逻辑处理
}

@end
```

通过对下一个响应者(nextBusiness)的赋值，实现具体职责的实现顺序。我们可以通过后台服务器配置对应的职责顺序，同时代码中通过类名分别对应相应的业务，实现动态控制相应职责的执行顺序。

### 命令

命令模式可以将行为参数化，降低代码重合度。

声明一个抽象命令类，子类可以继承该类实现具体的业务逻辑行为。
```objectivec
#pragma mark ----- Command.h -----

@class Command;
typedef void(^CommandCompletionCallBack)(Command* cmd);

@interface Command : NSObject
@property (nonatomic, copy) CommandCompletionCallBack completion;

- (void)execute;
- (void)cancel;

- (void)done;

@end
```

```objectivec
#pragma mark ----- Command.m -----

@implementation Command

- (void)execute{
    //override to subclass;
    [self done];
}

- (void)cancel {
    self.completion = nil;
}

- (void)done {
    dispatch_async(dispatch_get_main_queue(), ^{
        if (_completion) {
            _completion(self);
        }
        //释放
        self.completion = nil;
        [[CommandManager sharedInstance].arrayCommands removeObject:self];
    });
}

@end
```

再声明一个管理命令的单例类，允许全局控制命令的实行、取消实行。

```objectivec
#pragma mark ----- CommandManager.h -----

@interface CommandManager : NSObject

// 命令管理容器
@property (nonatomic, strong) NSMutableArray <Command*> *arrayCommands;

// 命令管理者以单例方式呈现
+ (instancetype)sharedInstance;

// 执行命令
+ (void)executeCommand:(Command *)cmd completion:(CommandCompletionCallBack)completion;

// 取消命令
+ (void)cancelCommand:(Command *)cmd;

@end
```

```objectivec
@implementation CommandManager

// 命令管理者以单例方式呈现
+ (instancetype)sharedInstance {
    static CommandManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[super allocWithZone:NULL] init];
    });
    return instance;
}

// 【必不可少】
+ (id)allocWithZone:(struct _NSZone *)zone {
    return [self sharedInstance];
}

// 【必不可少】
- (id)copyWithZone:(nullable NSZone *)zone {
    return self;
}

// 初始化方法
- (id)init {
    self = [super init];
    if (self) {
        // 初始化命令容器
        _arrayCommands = [NSMutableArray array];
    }
    return self;
}

+ (void)executeCommand:(Command *)cmd completion:(CommandCompletionCallBack)completion {
    if (cmd) {
        // 如果命令正在执行不做处理，否则添加并执行命令
        if (![self _isExecutingCommand:cmd]) {
            // 添加到命令容器当中
            [[[self sharedInstance] arrayCommands] addObject:cmd];
            // 设置命令执行完成的回调
            cmd.completion = completion;
            //执行命令
            [cmd execute];
        }
    }
}

// 取消命令
+ (void)cancelCommand:(Command *)cmd {
    if (cmd) {
        // 从命令容器当中移除
        [[[self sharedInstance] arrayCommands] removeObject:cmd];
        // 取消命令执行
        [cmd cancel];
    }
}

// 判断当前命令是否正在执行
+ (BOOL)_isExecutingCommand:(Command *)cmd {
    if (cmd) {
        NSArray *cmds = [[self sharedInstance] arrayCommands];
        for (Command *aCmd in cmds) {
            // 当前命令正在执行
            if (cmd == aCmd) {
                return YES;
            }
        }
    }
    return NO;
}

@end
```

# 结语

设计模式主要就分析这么多，主要需要记住六大设计原则和一些主要的设计模式，能够在遇到问题时灵活运用才是关键。

# 引用

[学习并理解 23 种设计模式](https://github.com/xietao3/Study-Plan/blob/master/DesignPatterns/README.md)