---
title: "创建自己的 GitHub 博客"
catalog: true
toc_nav_num: true
date: 2019-02-12 13:40:24
subtitle: "基于 Hexo 搭建 GitHub Pages"
header-img: "/img/article_header/article_header.png"
busuanzi: true
tags:
- GitHub

---

# Introduction

本人是个懒人，一直想动手写博客，却一拖再拖，总觉得自己的辞藻不够华丽，写出来不够工整漂亮。但是，这次，让我动手这件事情的原因，却是公司把技术分享或者博客作为绩效的考评，相比较技术分享，博客这个不用动嘴的方式却是不二之选。

既然要写博客，当然是选择高大上的基于 GitHub 平台创建的博客啦。GitHub，开源代码库以及版本控制系统，大家都知道的网站。之前我也只是用来找找开源项目，成熟的解决方案，没想到还能作为博客站，真是一个让人愉悦的网站。

这篇文章就是我在创建博客过程中的一些心得（怨念）。

# Build GitHub Pages  

1. 进入 [GitHub](https://github.com/new) 主页创建新仓库，
![GitHub](/img/article/1.png)

2. 进入刚刚创建的仓库，点击 Setting，找到下面的 GitHub Pages，
![GitHub](/img/article/2.png)
任意选择一个主题之后既可以通过 **用户名.github.io** 访问你的博客了。

# Set Up Hexo  

1. 安装 [Hexo](https://github.com/hexojs/hexo)，
``` Shell
$ npm install hexo-cli -g
```

2. Hexo 常用命令
``` Shell
hexo n == hexo new 新建文章
hexo g == hexo generate 生成部署
hexo s == hexo server 本地预览
hexo d == hexo deploy 上传云端
hexo clean 清空部署
```
通过上述命令，可以创建一个默认主题的 blog 页面，即可[生成本地预览](http://localhost:4000/)。

3. 安装主题 [themes](https://hexo.io/themes/)，
在对应的 blog 文件夹下
``` Shell
hexo clean
git clone https://github.com/huweihuang/hexo-theme-huweihuang.git ./hexo-huweihuang （举例）
```
在对应 blog 目录下找到 _config.yml 文件，修改 theme 属性设置为对应的主题名字，即安装主题成功。

4. 部署到 GitHub
继续在上述的 _config.yml 文件下编辑，
![blog](/img/article/3.png)
完成之后就可以通过 **hexo d**上传同步到 GitHub。

# More Questions

每个主题在对应的 _config.yml 都有对应的配置，修改对应配置即可对自己的 blog 页面进行调整。
下面是我想要添加的两个功能点分享：

## Page View statistics - 不蒜子

> “不蒜子”与百度统计谷歌分析等有区别：“不蒜子”可直接将访问次数显示在您在网页上（也可不显示）；对于已经上线一段时间的网站，“不蒜子”允许初始化首次数据。。

1. 安装脚本
``` JavaScript
<script async src="//busuanzi.ibruce.info/busuanzi/2.3/busuanzi.pure.mini.js">
</script>
```
打开对应 *themes/layout/_partial/footer.ejs* 目录下添加上述脚本即可，当然你也可以添加到其他地方。

2. 显示站点总访问量
复制下面代码添加到你需要显示的位置即可。有两种算法可选：
算法a：pv的方式，单个用户连续点击n篇文章，记录n次访问量。
``` CSS
<span id="busuanzi_container_site_pv">
    本站总访问量<span id="busuanzi_value_site_pv"></span>次
</span>
```
算法b：uv的方式，单个用户连续点击n篇文章，只记录1次访客数。
``` CSS
<span id="busuanzi_container_site_uv">
  本站访客数<span id="busuanzi_value_site_uv"></span>人次
</span>
```

3. 显示单页面访问量
算法：pv的方式，单个用户点击1篇文章，本篇文章记录1次阅读量。
``` CSS
<span id="busuanzi_container_page_pv">
  本文总阅读量<span id="busuanzi_value_page_pv"></span>次
</span>
```

4. 自定义样式
'不蒜子'被称为极客的算子，正是因为不蒜子自身只提供标签+数字，至于显示的style和css动画效果，任你发挥。
- busuanzi_value_site_pv 的作用是异步回填访问数，这个id一定要正确。
- busuanzi_container_site_pv的作用是为防止计数服务访问出错或超时（3秒）的情况下，使整个标签自动隐藏显示，带来更好的体验。这个id可以省略。
所以，你也可以使用极简模式：
``` CSS
本站总访问量<span id="busuanzi_value_site_pv"></span>次
本站访客数<span id="busuanzi_value_site_uv"></span>人次
本文总阅读量<span id="busuanzi_value_page_pv"></span>次
```

## Add comments module - Disqus

> 多说已于2017年6月1日正式关停服务，缅怀..

封装好的主题本身已经携带了 多说 与 Disqus 两种评论方式，同样只要在 _config.yml 文件下填写 disqus_username 就可以了，
``` Shell
# Disqus settings
disqus_username: username
```
如果有这么简单，我就不会写分享了，中间坑点满满。

1. 首先，自然是去 [Disqus](https://disqus.com) 官网注册，同时前往右上角的**头像-View Profile**获取到对应的用户信息，回来填写到对应的 disqus_username 上，然后预览发现什么都没有发生。

2. 然后，我们需要前往右上角的**头像-Install on Site**，将你的账号与你的网页关联起来，同时在这时候你会找到你的 Shortname, 他才是最终填写到 disqus_username 位置上的名称，
![Disqus](/img/article/4.png)
然后预览你可能会遇到*We were unable to load Disqus.*，如果没有，恭喜你，你已经成功启动了评论功能，如果没有，那你还得看下一步。

3. 参考 [issue #876](https://github.com/iissnan/hexo-theme-next/issues/876)，可能的原因是我们需要把 disqus_url 变量赋值去掉，有disqus_identifier 就可以。至于页面的链接，Disqus 会自动取 `window.location.href`，Hexo 生成的都是静态的页面，这个值也是正确的。然后，我们预览最终会出现评论页面了。

# End

好了，我的第一篇正规博客结束了，看到自己也有了博客站，心里也是美滋滋。
刚好，这篇博客是在2019年初完成的，预示着一个完美的开始，希望我能把他运营下去吧，加油！

# Reference

> [我是如何利用Github Pages搭建起我的博客，细数一路的坑](https://www.cnblogs.com/jackyroc/p/7681938.html)
> [不蒜子](http://ibruce.info/2015/04/04/busuanzi/)
> [正确的Disqus使用姿势](https://www.jianshu.com/p/7e4453421b8f)