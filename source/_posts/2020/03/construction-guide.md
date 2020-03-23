---
title: 本博客构建指南
date: 2020-03-22 18:39:47
tags:
    - hexo博客
    - 构建指南
---

# hexo博客搭建

### 1 环境搭建

- node.js 去官网下载安装即可

- 默认全局安装

  ` npm install -g hexo-cli`

- Git 安装

### 2 创建项目  eqshen.github.io

创建命令并安装依赖

```
hexo init eqshen.github.io
cd eqshen.github.io
npm install
```

### 3 下载主题

这里我使用的模板是 https://github.com/Shen-Yu/hexo-theme-ayer，也可以参考这里面的安装教程。

在 eqshen.github.io目录下执行命令,把主题下载下来

`git clone https://github.com/Shen-Yu/hexo-theme-ayer.git themes/ayer`

然后更新主题(刚下载下来应该是最新的，感觉这步可有可无)

```
cd themes/ayer
git pull
```

然后修改配置，eqshen.github.io/_config.yml

`theme: ayer`

### 4 安装必须插件

- 搜索插件

`npm install hexo-generator-searchdb --save`

然后在根目录下 _config.yml中添加以下配置

```
# hexo-generator-searchdb
search:
  path: search.xml
  field: post
  format: html
```

- RSS插件

```
npm install hexo-generator-feed --save
```

同样将下面代码复制到你项目根目录下的 _config.yml中

```
feed:
    type: atom
    path: atom.xml
    limit: 20
    hub:
    content:
    content_limit: 140
    content_limit_delim: ' '
    order_by: -date
```

这样项目的基本插件就安装完毕，而且依赖信息都会自动加入到你项目根目下的package.json中，下次直接npm install就可以了（nodejs包管理~~）

### 5 添加菜单

由于使用了主题theme,所以修改菜单应该在 theme/ayer/_config.yml中修改，主页的标题副标题等都是改这里

修改下面这段代码对应的内容

```
menu:
  主页: /
  归档: /archives
  分类: /categories
  标签: /tags
  关于我: /sys/aboutMe
```

/archives是默认的不需要改，/categories应该对应你项目下`/source/categories/` 目录，这时你会发现目录根本不存在，咋整？盘它。

执行命令创建目录,命令行执行前应该要切换到你项目的（根）目录，我这里是 `cd ~/workspace/eqshen.github.io/`

```
hexo new page categories
```

然后修改配置  source/categories/index.md中的内容，替换为以下内容

```
---
title: categories
type: "categories"
layout: "categories"
---
```



然后再同样生成tag目录

```
hexo new page tags
```

同样修改配置  source/tags/index.md内容，替换为下面内容

```
---
title: tags
type: "tags"
layout: "tags"
---
```



这样就大功告成，这里你可能会疑问，上面的命令是什么意思？hexo创建文章、目录、tag等是不用你 ‘右键->新建文件’这种操作的，只需要执行具体的命令，就可以替代上述操作，具体有哪些命令可以看这里 ：[指令](https://hexo.io/zh-cn/docs/commands)

### 6 本地启动

命令`hexo server`

然后就可以到 localhost:4000 查看了