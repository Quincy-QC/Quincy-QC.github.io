---
title: "NSTimer循环引用解决方案"
catalog: true
toc_nav_num: true
date: 2020-03-30 20:04:21
subtitle: "The Circular Reference Issue with NSTimer"
header-img: "/img/article_header/article_header.png"
busuanzi: true
tags:
- iOS - Objective-C

---

# 问题

当我们从控制器`View Controller`跳转到一个启用重复调用定时器的控制器`Anohter View Controller`后，进行`pop`操作后，无法销毁该控制器`Another View Controller`。

``` objectivec
self.timer = [NSTimer timerWithTimeInterval:1.0 target:self selector:@selector(timerAction) userInfo:nil repeats:YES];
```

`timer`作为`VC`的属性，被`VC`强引用，创建`timer`对象时`VC`作为`target`被`timer`强引用，即循环引用。

![循环引用](/img/article/20200330/1.png)

<!-- more -->

# weak指针

既然有循环引用，那我们可以用解决`Block`循环引用的方案--`weak`指针来解决吗？答案是不可以的，因为无论是用`weak`还是`strong`修饰，在`NSTimer`的创建过程中都会重新生成一个强引用指针指向`VC`，导致循环引用。

# 解决方案

## invalidate：

> This method is the only way to remove a timer from an NSRunLoop object. The NSRunLoop object removes its strong reference to the timer, either just before the invalidate method returns or at some later point.
If it was configured with target and user info objects, the receiver removes its strong references to those objects as well.

我们可以调用`NSTimer`的`invalidate`方法来手动释放`timer`，当然我们不能在`dealloc`中调用这个方法，因为本身因为循环引用已经导致当前控制器无法释放，自然也不会走`dealloc`方法。
所以我们可以考虑在`viewWillDisappear`这个方法中进行释放，缺点就是可以会忘记，而且并不是很严谨。

## 类方法：

创建一个继承自`NSObject`的子类`TimerWeakTarget`，并创建开启定时器的方法。

`TimerWeakTarget`文件：
``` objectivec
// TimerWeakTarget.h
@interface TimerWeakTarget : NSObject

+ (NSTimer *) scheduledTimerWithTimeInterval:(NSTimeInterval)interval
                                      target:(id)aTarget
                                    selector:(SEL)aSelector
                                    userInfo:(id)userInfo
                                     repeats:(BOOL)repeats;

@end
```

``` objectivec
// TimerWeakTarget.m
@interface TimerWeakTarget ()

@property (weak, nonatomic) id target;
@property (assign, nonatomic) SEL selector;
@property (weak, nonatomic) NSTimer *tempTimer;

@end

@implementation TimerWeakTarget

+ (NSTimer *)scheduledTimerWithTimeInterval:(NSTimeInterval)interval target:(id)aTarget selector:(SEL)aSelector userInfo:(id)userInfo repeats:(BOOL)repeats {
    TimerWeakTarget *timerWeakTarget = [[TimerWeakTarget alloc] init];
    timerWeakTarget.target = aTarget;
    timerWeakTarget.selector = aSelector;
    timerWeakTarget.tempTimer = [NSTimer scheduledTimerWithTimeInterval:interval target:timerWeakTarget selector:@selector(fire:) userInfo:userInfo repeats:repeats];
    return timerWeakTarget.tempTimer;
}


- (void)fire:(NSTimer *)timer {
    if (self.target && [self.target respondsToSelector:self.selector]) {
        [self.target performSelector:self.selector withObject:timer.userInfo];
    } else {
        [self.tempTimer invalidate];
    }
}

@end
```

控制器内调用：
``` objectivec
self.timer = [TimerWeakTarget scheduledTimerWithTimeInterval:1.0 target:self selector:@selector(timerAction) userInfo:nil repeats:YES];
```

`VC`强引用`timer`，`timer`强引用的`target`是`timerWeakTarget`，而`timerWeakTarget`弱引用`VC`，解除循环引用。

![解除循环引用](/img/article/20200330/2.png)

## WeakProxy：

创建一个继承`NSObject`的子类`WeakProxy`，并实现消息转发的相关方法。

`WeakProxy`文件：
``` objectivec
// WeakProxy.h
@interface WeakProxy : NSObject

+ (instancetype)proxyWithTarget:(id)target;

@end
```

``` objectivec
// WeakProxy.m
@interface WeakProxy ()

@property (weak, nonatomic) id weakTarget;

@end

@implementation WeakProxy

+ (instancetype)proxyWithTarget:(id)target {
    return [[WeakProxy alloc] initWithTarget:target];
}


- (instancetype)initWithTarget:(id)target {
    _weakTarget = target;
    return self;
}


- (id)forwardingTargetForSelector:(SEL)aSelector {
    return self.weakTarget;
}


/* OR
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    return [self.weakTarget methodSignatureForSelector:aSelector];
}


- (void)forwardInvocation:(NSInvocation *)anInvocation {
    SEL sel = [anInvocation selector];
    if ([self.weakTarget respondsToSelector:sel]) {
        [anInvocation invokeWithTarget:self.weakTarget];
    }
}
*/

@end
```

控制器内的调用：
``` objectivec
self.timer = [NSTimer scheduledTimerWithTimeInterval:1.0 target:[WeakProxy proxyWithTarget:self] selector:@selector(timerAction) userInfo:nil repeats:YES];
```

这个方法的原理和类方法相似，将`timer`的`target`设置为`WeakProxy`的实例，利用消息转发机制实现执行`VC`中的计时方法，解决循环引用。

## GCD：

控制器内：
``` objectivec
@interface AnotherViewController ()

@property (strong, nonatomic) dispatch_source_t timer;

@end

@implementation AnotherViewController

- (void)viewDidLoad {
    [super viewDidLoad];
   
    self.timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0));
    dispatch_source_set_timer(self.timer, DISPATCH_TIME_NOW, 1*NSEC_PER_SEC, 0);
    dispatch_source_set_event_handler(self.timer, ^{
        dispatch_async(dispatch_get_main_queue(), ^{
            NSLog(@"fire");
        });
    });
    dispatch_resume(self.timer);
}

@end
```

结束计时：
``` objectivec
dispatch_source_cancel(self.timer);
```

## Block:

`iOS 10`之后新出的`API`：

``` objectivec
self.timer = [NSTimer scheduledTimerWithTimeInterval:1.0 repeats:YES block:^(NSTimer * _Nonnull timer) {
    NSLog(@"fire");
}];
```


