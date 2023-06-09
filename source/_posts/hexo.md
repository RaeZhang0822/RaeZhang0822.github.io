---
title: Github Pages部署Hexo 踩坑记录
---

## Quick Start

这篇博客主要内容

1. 使用 Hexo（默认主题） 搭建 Github Pages 的过程并使用 Github Actions 实现自动部署
2. 使用主题后自动部署出错及修复的过程

## Hexo 基本使用和自动部署过程

hexo 的基本使用参考 [官方文档](https://hexo.io/zh-cn/docs/)
使用 GitHub Actions 部署至 GitHub Pages 参考 [在 GitHub Pages 上部署 Hexo](https://hexo.io/zh-cn/docs/github-pages.html)

按照上述两个官方文档，可以顺利将默认主题的 Hexo 部署到 GitHub pages.
说明下，这种方式只用了一个 `username.github.io`仓库，main 分支上是 hexo 的源代码，gh-pages 分支上是 hexo 生成的文件。
配置的 workflow 可以实现:当 main 分支上有 push 时，重新执行 build 命令,将 public 文件下的内容创建一次 commit 然后 push 到 gh-pages 分支。
![image](./img/hexo/branches.png)

## Hexo 使用主题后自动部署失败踩坑记录

### 问题场景

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

#### 部署出错

可以在本地预览一下是没有问题的，但是当我把这一个主题改动 push 到 main 分支后，站点页面空白了，查看 gh-pages 分支发现文件数量明显少了很多，想了一下应该是主题作为子仓库在云端时没有拉到相应的 github 仓库的代码，导致无法找到主题文件。

### 解决思路

既然是没有主题文件，我尝试修改 workflow, 在 build 前拉取主题文件仓库

```yml
- name: Install Dependencies
  run: npm install
  # 安装主题文件
- name: Install Theme
  run: git clone git@github.com:klugjo/hexo-theme-clean-blog.git themes/clean-blog
- name: Build
  run: npm run build
- name: Deploy
  uses: peaceiris/actions-gh-pages@v3
  with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
    publish_dir: ./public
```

push 之后触发重新构建报错
![image](./img/hexo/error.png)

#### 错误原因

> 这是因为你的开发机器上的用户通常与特定的 GitHub 帐户关联，并且该帐户具有对私有仓库的访问权限，而 GitHub Actions 用户只能看到它所附属的仓库。

> 如今，与 GitHub 进行身份验证的最佳方式是使用部署密钥（Deploy Keys）。这是一组 SSH 密钥，可以为用户提供对特定仓库的只读访问权限（默认情况下）。

所以需要重新生成一组 SSH 密钥。

#### 解决过程

1. 生成新的 ssh-keys
   ![deployKeys](./img/hexo/sshKeyGen.png)

2. 将公钥添加到 Deploy Keys
   ![deployKeys](./img/hexo/deployKeys.png)

3. 将 私钥 添加到 username.github.io 仓库的 密钥中
   ![deployKeys](./img/hexo/secretKey.png)

4. 在 workflow 中添加授权

```yml
- name: Install Dependencies
  run: npm install
#   授权
- name: Give GitHub Actions access to hexo theme repo
  uses: webfactory/ssh-agent@v0.7.0
  with:
    ssh-private-key: ${{ secrets.SECRET_REPO_DEPLOY_KEY }}
# 安装主题
- name: Install Theme
  run: git clone git@github.com:klugjo/hexo-theme-clean-blog.git themes/clean-blog
- name: Build
  run: npm run build
```

修改完之后重试，即可看到 CI 成功，页面成功部署。

# 参考资料

[官方文档](https://hexo.io/zh-cn/docs/)
[在 GitHub Pages 上部署 Hexo](https://hexo.io/zh-cn/docs/github-pages.html)
[GitHub Actions can't access private repos? Here's how to fix it](https://adventures.michaelfbryan.com/posts/configuring-cargo-auth-in-github-actions/)
