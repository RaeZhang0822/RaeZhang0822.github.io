---
title: Github Pages部署Hexo 踩坑记录
---

## Quick Start

这篇博客主要内容

1. 选择 hexo 的原因
2. 使用 Hexo（无主题） 搭建 Github Pages 的过程并使用 Github Actions 实现自动部署
3. 使用主题后自动部署出错及修复的过程

## 为什么选择 hexo?

> 无它，唯手熟尔。

很早之前我就用 hexo 建过一次 github pages, 当时用了两个仓库 ，一个用来存放 hexo 源码，另一个是 username.github.io 。因为 hexo 可以在本地编辑完文章之后可以直接一键发布，所以我没有及时把源代码 push 到远程的 hexo 源代码仓库。后来某次换电脑的时候忘记把代码拷贝下来，hexo 源代码仓库的代码丢了，补救无果索性换到别的地方写博客。

这次重新建站，调研了 jekyll、hugo、hexo 三个主流方案，客观来说 hugo 在编译速度、社区支持度方面都是最佳选择，但是我三个都试了一下，过程中遇到不少坑，只有 hexo 成功了，并且用 GitHub Actions 实现了一次 push 同时更新源码和展示页面的功能，方便不少，索性先用用着，以后有空折腾试下 hugo。

## Hexo 基本使用和自动部署过程

hexo 的基本使用参考 [官方文档](https://hexo.io/zh-cn/docs/)
使用 GitHub Actions 部署至 GitHub Pages 参考 [在 GitHub Pages 上部署 Hexo](https://hexo.io/zh-cn/docs/github-pages.html)

按照上述两个官方文档，可以顺利讲无主题的 Hexo 部署到 GitHub pages.
说明下，这种方式只用了一个 `username.github.io`仓库，main 分支上是 hexo 的源代码，gh-pages 分支上是 hexo 编译后的文件。
配置的 workflow 可以实现当 main 分支上有 push 时，重新执行 build 命令,将 public 文件下的内容创建一次 commit 然后 push 到 gh-pages 分支。
![image](./img/hello_world/branches.png)
自动部署的任务可以在仓库的 **_Action _** 下看到

## Hexo 使用主题后自动部署失败踩坑记录

### Hexo 主题使用方法

#### 主题安装

以 [clean-blog 主题](https://github.com/klugjo/hexo-theme-clean-blog.git) 为例子
在项目根目录下执行

```javascript
    git clone https://github.com/klugjo/hexo-theme-clean-blog.git themes/clean-blog
```

将主题仓库 clone 到 themes 文件夹下

#### 设置主题

在根目录下的`_config.yml` 文件中设置

```javascript
theme: ”clean-blog“; // 注意与themes文件夹下的主题文件夹同名
```

可以在本地预览一下是没有问题的，但是当我把这一个主题改动 push 到 main 分支后，站点页面空白了，查看 gh-pages 分支发现文件数量明显少了很多，查看 Actions 下的 workflow 记录，发现在执行 Build 任务时有错误

# 参考资料

[静态博客框架 jekyll、hexo 和 hugo 三者之间的区别与差异](https://zhuanlan.zhihu.com/p/368407566)
[官方文档](https://hexo.io/zh-cn/docs/)
[在 GitHub Pages 上部署 Hexo](https://hexo.io/zh-cn/docs/github-pages.html)
