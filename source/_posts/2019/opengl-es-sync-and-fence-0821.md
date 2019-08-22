---
title: "OpenGL ES学习--同步对象和栅栏"
catalog: true
toc_nav_num: true
date: 2019-08-21 19:10:54
subtitle: "About OpenGL ES"
header-img: "/img/article_header/article_header.png"
busuanzi: true
tags:
- iOS

---

> OpenGL ES 3.0为应用程序提供了一组OpenGL操作在GPU上执行结束的机制。我们可以同步多个图形上下文和线程中的GL操作，这对于许多高级图形应用来说很重要。例如：我们可能希望等待交换反馈的结果，然后在应用程序中使用这些结果。

# 刷新和结束

OpenGL ES 3.0 API继承了OpenGL的客户-服务器模型。应用程序（或客户）发出命令，这些命令由OpenGL ES实现（或者服务器）处理。在OpenGL中，客户和服务器可以存在于网络上的不同机器，OpenGL ES也允许客户和服务器处于不同机器上，但是因为OpenGL ES针对的是手持和嵌入平台，所以客户和服务器通常在同一个设备上。
在客户-服务器模型中，客户发出的命令不一定立刻发送到服务器。如果客户和服务器在一个网络上操作，那么在网络上发送单独命令将非常低效。相反，命令可以缓存在客户端，在稍后的某个时刻发送到服务器。为了支持这种方法，需要一种机制让客户知道服务器合适完成前面提交的命令的执行。考虑另一个例子：多个OpenGL ES上下文（各自为不同线程的当前上下文）共享对象。为了正确地在这些上下文之间同步，很重要的一点是来自上下文A的命令在来自上下文B的命令之前发往服务器，这取决于上下文A修改的OpenGL ES状态。`glFlush`命令用于刷新当前OpenGL ES上下文中未决的命令，并将它们发往服务器。注意，`glFlush`只将命令发往服务器，而不等待它们完成。如果客户端要求这些命令完成，应该使用`glFinish`命令。除非绝对必要，否则我们不建议使用`glFinish`，因为`glFinish`在上下文中所有排队的命令由服务器完全之前不会返回，调用`glFinish`通过强制客户和服务器同步它们的操作可能对性能产生不利的影响。

# 为什么使用同步对象

OpenGL ES 3.0引入了一个称作栅栏（Fence）的新特性，为应用程序提供了通知GPU在一组OpenGL ES操作执行完成之前先等待、然后再将更多执行命令送入队列的手段。我们可以在GL命令流中插入栅栏命令，然后将其与需要等待的同步对象关联。
如果我们将同步对象的使用与`glFinish`命令作比较，则同步对象更高效，因为可以等待GL命令流的部分完成。相比之下，调用`glFinish`命令可能降低应用程序性能，因为该命令会清空图形管线。

# 创建和删除同步对象

为了在GL命令流中插入栅栏命令并创建同步对象，可以调用如下函数：

``` objc
/**
 在GL命令流中插入栅栏命令并创建同步对象

 @param condition#> 指定想同步对象发送信号必须符合的条件；必须为GL_SYNC_GPU_COMMANDS_COMPLETE description#>
 @param flags#> 指定控制同步对象行为的标志按位组合；当前必须为0 description#>
 @return GLsync 同步对象
 */
glFenceSync(GLenum condition, GLbitfield flags);
```

在同步对象首次创建时，其状态为未收到信号（Unsignaled）。在栅栏命令满足指定条件时，其状态变为已收到信号（Signaled）。因为同步对象不能重用，所以必须为每次同步操作创建一个同步对象。
要删除同步对象，可以调用如下函数：

``` objc
/**
 删除同步对象

 @param sync#> 指定要删除的同步对象 description#>
 @return void
 */
glDeleteSync(GLsync sync);
```

删除操作不会立即发生，因为同步对象只有在没有其他操作等待它时才能删除。因此，可以在等待同步对象之后调用`glDeleteSync`命令。

# 等待和向同步对象发送信号

可以用如下调用阻塞客户，并等待一个同步对象收到信号：

``` objc
/**
 阻塞客户，并等待一个同步对象收到信号

 @param sync#> 指定等待其状态的同步对象 description#>
 @param flags#> 指定控制命令刷新行为的位域；可能是GL_SYNC_FLUSH_COMMANDS_BIT description#>
 @param timeout#> 指定等待同步对象获得信号的超时时间（纳秒） description#>
 @return GLenum
 */
glClientWaitSync(GLsync sync, GLbitfield flags, GLuint64 timeout);
```

如果同步对象已经处于“已收到信号”的状态，`glClientWaitSync`命令将立即返回。否则，调用将阻塞，并在最多`timeout`纳秒的时间内等待同步对象收到信号。
`glClientWaitSync`函数可能返回如下值：
- GL_ARRAY_SIGNALED：同步对象在函数调用时已经处于“已收到信号”状态。
- GL_TIMEOUT_EXPIRED：同步对象在`timeout`纳秒之后还没有进入“已收到信号”状态。
- GL_CONDITION_SATISFIED：同步对象在超时之前收到信号。
- GL_WAIT_FAILED：发生错误。

`glWaitSync`函数类似于`glClientWaitSync`函数，但是该函数立即返回且阻塞GPU，直到同步对象收到信号：

``` objc
/**
 阻塞GPU直到同步对象收到信号

 @param sync#> 指定等待其状态的同步对象 description#>
 @param flags#> 指定控制命令刷新行为的位域；必须为0 description#>
 @param timeout#> 指定服务器继续之前等待的超时（纳秒）；必须为GL_TIMEOUT_IGNORED description#>
 @return void
 */
glWaitSync(GLsync sync, GLbitfield flags, GLuint64 timeout);
```

# 总结

这篇文章介绍了OpenGL ES 3.0中宿主应用和GPU执行同步的基础知识。