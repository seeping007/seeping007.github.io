---
layout: post
title: Github Pages＋Jekyll搭建个人博客
description: 轻松使用Github Pages建立自己的独立博客，并使用Jekyll模版进行美化。
category: blog
---

This is where the story starts......

### 1. 建立/初始化repository

登录Github，新建一个代码库（repository）。假设账户名为user，代码库名为repository。

> Tip：大多网络文章都强调并标明，该代码库的名称必须与账户名相同。其实不然，可以随意取名（例如，笔者将其取名为 personal）。但是，如果名称是用户名时，后续生成网址正好为最简短的形式，即 https://username.github.io；如果采用其他名称，则会在网址后额外添加一个项目名，例如 https://username.github.io/personal。

a. 将代码库拷贝到本地

```
git clone github.com/user/repository.git
```

b. 推送更新

```
git add index.html
git commit -a -m "first pages commit"
git push
```
> Tip：Github要求“User pages must be build from the master branch”，因此以上操作均在master分支上。其他种类的Github pages可能只有gh-pages分支的代码才会被自动生成网站链接。

c. 加载网站

推送更新之后，很快Github就会帮你自动完成网站构建。链接如下：

```
https://<username>.github.io/<repository>
```

### 2. Jekyll + themes 快速丰富网站

#### 工具包安装
1.检查 Ruby 版本号（必须高于ver-2.0.0，如果没有得先安装）

```
ruby --version
```

2.安装 Bundler（管理 Gem 相依性的工具，例如jekyll）

```
gem install bundler
```

3.安装 Jekyll

首先需要在 repository 的根目录下生成 Gemfile 文件，内容为：

```
source 'https://rubygems.org'
gem 'github-pages', group: :jekyll_plugins
```

采用 Bundler 安装 Jekyll

```
bundle install
```

### 3. 使用Jekyll模版

网络上提供了大量适用于 GitHub Pages 的 Jekyll 模版，推荐一个网站：

* [Jekyll Themes][1]

各种风格任你选，去下载一个并解压将文件放到你的代码库（repository）根目录下。当然，适当的整理会略有帮助。

本地服务器预览：

```
$ bundle exec jekyll serve
Configuration file: /Users/octocat/my-site/\_config.yml
            Source: /Users/octocat/my-site
       Destination: /Users/octocat/my-site/\_site
 Incremental build: disabled. Enable with --incremental
    Generating...
                done in 0.309 seconds.
 Auto-regeneration: enabled for '/Users/octocat/my-site'
Configuration file: /Users/octocat/my-site/\_config.yml
    Server address: http://127.0.0.1:4000/
  Server running... press ctrl-c to stop.
```
在本地浏览器访问 http://127.0.0.1:4000/，就可以预览网站效果啦。当然，网页信息都还是模版自带，以后的工作就变成了修改本地文件并使用 GitHub 进行代码托管了。

> Tips: 模版目录中 _site 是由 Jekyll 转换生成，用于保存本地预览页面，因此建议将 _site 目录放进你的 .gitignore 文件中。

### 4. 绑定个人域名
可参考 [GitHub Pages 绑定来自阿里云的域名][2]

### 相关资料
* [GitHub + Jekyll 搭建并美化个人网站][3]

[1]: http://jekyllthemes.org/
[2]: http://quantumman.me/blog/setting-up-a-domain-with-gitHub-pages.html
[3]: http://www.jianshu.com/p/85ca31174488

