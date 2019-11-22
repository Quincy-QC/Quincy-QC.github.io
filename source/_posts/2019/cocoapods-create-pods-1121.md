---
title: "基于Cocoapods创建私有公有库"
catalog: true
toc_nav_num: true
date: 2019-12-21 18:30:24
subtitle: "Making a Cocoapod"
header-img: "/img/article_header/article_header.png"
busuanzi: true
tags:
- iOS

---

# 前言

之前一直在用大神们的三方库，工作这么久也积累了一些自己的常用类库，考虑想要上传到自己的Cocoapods私有库，方便后期新项目的调用，下面来介绍上传方法。

# 方法

## 首先创建pod模板

可以使用`pod lib create projectName`，如果有自己的模板，可以添加`--template-url=URL`来调用。

创建过程会让你进行一些选项：
``` zsh
What platform do you want to use?? [ iOS / macOS ]
 > iOS

What language do you want to use?? [ Swift / ObjC ]
 > Swift

Would you like to include a demo application with your library? [ Yes / No ]
 > NO

Which testing frameworks will you use? [ Quick / None ]
 > None

Would you like to do view based testing? [ Yes / No ]
 > No
```

## 配置文件

创建完成后会自动打开一个Xcode文件，这边我们打开`Podspec Metadata`文件下的`Project.podspec`配置文件：

![配置文件](/img/article/20191121/1.png)

中间注意几个点：

1. `s.name`的名字必须和项目名相同，不可更改；
2. `s.version`是作为pod的版本号，后面项目上传Github时需对应打上相同的`tag`值；
3. `s.homepage`与`s.source`是项目的地址，按照对应的格式修改即可；
4. `s.framework`对应单数的系统库，`s.frameworks`对应复数的系统库；
5. `s.dependency`对应其他三方库，如果多个三方库只需要多行添加即可，无需逗号；
6. `s.subspec 'Core' do |ss|`可以将项目库文件划分子模块，注意每个模块后对应的`end`。

配置文件结束后，将我们需要上传的类库复制到指定位置即可。

## 验证本地库

在上传之前我们需要验证下本地库是否正确，使用`pod lib lint Project.podsepc --verbose --allow-warnings`进行验证。（`pod spec lint Project.podspec`是用于验证远程库）

## 上传远程仓库并Tag

验证成功后需要将代码上传到远程仓库，代码上传远程仓库之后需要打对应的版本号Tag。

## 私有库

如果不想对外开放，可以使用`pod repo push 仓库地址 --allow-warnings`将podspec放到自己的项目网络地址上，比如公司的gitlab上，创建一个项目专门管理podspec。私有库是`pod search`搜索不到的。

## 公有库

如果需要对外开放，需要创建cocoapods账号：`pod trunk register xxx@email.com userName --verbose`

这边它会发送一封邮件，点击验证即可。使用`pod trunk me`查看账号信息。

然后就可以通过命令进行发布：`pod trunk push Project.podspec --verbose --use-libraries --allow-warnings`

发布成功之后进行`pod repo update`更新，同时还需要进行`rm ~/Library/Caches/CocoaPods/search_index.json`索引缓存删除，然后我们就可以通过`pod search`进行查找对应的库啦。

# 删pod库特定版本

可以删除一个pod的特定版本来纠正意外推送。

`pod trunk delete podName version`

你也可以放弃整个pod和所有版本。

`pod trunk deprecate podName`

# 总结

通过上面步骤就可以简单的创建我们自己的远程类库，更加愉快的开发啦！