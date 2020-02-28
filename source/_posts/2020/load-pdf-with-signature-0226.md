---
title: "加载带有签章的PDF文件"
catalog: true
toc_nav_num: true
date: 2020-01-19 20:01:29
subtitle: "Load PDF File with Signature"
header-img: "/img/article_header/article_header.png"
busuanzi: true
tags:
- iOS - Objective-C

---

# 前言

常规的PDF文件可以直接通过`UIWebView`或`WKWebView`直接进行本地文件或远程链接加载，但是带有签章的PDF文件，需要一些处理。

<!-- more -->

# 对于iOS 12以上的系统

可以直接使用`WKWebView`进行加载本地PDF文件或URL，就会直接显示电子签章。（`UIWebView`并不会直接显示电子签章）。

# 对于iOS 12以下的系统

即使是使用`WKWebView`也无法加载显示电子签章，这时候就需要[pdf.js](https://github.com/mozilla/pdf.js)。`pfd.js`可以实现在`html`下直接浏览PDF文档，是一款开源的PDF文档读取解析插件。

`pdf.js`主要包含两个库文件：
- `pdf.js`负责API解析；
- `pdf.worker.js`负责核心解析；

## 下载

`pdf.js`是火狐浏览器的开源项目，[下载地址](http://mozilla.github.io/pdf.js/getting_started/)。

1. 我们可以直接下载`Stable`包，包含需要的`PDF.js`和`viewer`文件：

![pdf.js](/img/article/20200226/1.png)

2. 也可以从[github](https://github.com/mozilla/pdf.js)下载代码，
- 并进行`gulp generic`编译，最终需要的代码文件`pdf.js`和`pdf.worker.js`会在`build/generic/`路径下；
![编译后](/img/article/20200226/3.png)
- 或使用`gulp minified`编译，可以一定程度对文件进行压缩，最终需要的代码文件`pdf.js`和`pdf.worker.js`会在`build/minified/`路径下；
![编译后](/img/article/20200226/4.png)

如果还觉得文件够大，我们还可以对之进行一些删减，最终文件如下：
![文件](/img/article/20200226/5.png)

## 使用

直接拖入到项目中，文件夹添加方式选择`Create folder references`，选择`Create groups`会丢失样式。

PDF文件只能使用本地文件，所以对于网络资源需要先进行下载再展示。

代码：

``` objectivec
 - (void)loadPDFFile:(NSString*)filePath {  
    NSString *viwerPath = [[NSBundle mainBundle] pathForResource:@"viewer" ofType:@"html" inDirectory:@"minified/web"];
    NSString *urlStr = [NSString stringWithFormat:@"%@?file=%@#page=1", viwerPath, filePath];
    urlStr = [urlStr stringByAddingPercentEncodingWithAllowedCharacters:[NSCharacterSet URLQueryAllowedCharacterSet]];
    NSURLRequest *request = [NSURLRequest requestWithURL:[NSURL URLWithString:urlStr]];
    [self loadRequest:request];
}
```

至此，`pdf.js`加载的功能也只是和使用原生的方式一样加载无电子签章的文件，需要显示电子签章，需要将一下代码注释，可以全局搜索代码`data.fieldType === 'Sig'`查找：

``` objectivec
//    if (data.fieldType === 'Sig') {
//      data.fieldValue = null;
//
//      _this3.setFlags(_util.AnnotationFlag.HIDDEN);
//    }
```

## 在线使用

我们可以使用`mozilla`部署在`github pages`上的`Viewer`就行PDF加载，和本地`viewer.html`加载PDF文件类型，使用如下路径加载：

``` objectivec
NSString *urlStr = [NSString stringWithFormat:@"http://mozilla.github.io/pdf.js/web/viewer.html?file=%@#page=1", filePath];
```

但是源码的本身是默认不显示签章，所以如果想使用在线预览方式，需要我们自定义`HTML`修改部分代码并部署到网页，就可以实现在线预览。

## 最终效果

![效果图](/img/article/20200226/2.png)
