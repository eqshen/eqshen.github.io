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

### 7 部署

本次的博客包括评论（gitalk）都上放在github仓库里的（免费的），部署方式可以采用hexo官方推荐的travis。不过我采用了 GitHub Action的方式，博客源码放在仓库[eqshen.github.io](https://github.com/eqshen/eqshen.github.io)的develop分支上，编译后的前端代码放在该仓库的master分支。Github Action的脚本监听的是dep分支，即每次更新完博客，把源码推送到dep分支，博客就会自动编译并部署。配置步骤如下

1. 前提要创建好自己的仓库 xxx.github.io，其中xxx就是github用户名，如果还没了解过，先去百度吧，关键字：Github Page。

2. 在仓库的Actions标签页下点击`New workflow`创建自动部署脚本，然后点击右上角 `skip this:Set up a workflow yourself`，然后就上写脚本了，写好脚本点右上角 `Start Commit`按钮就会在你项目的根目录下生成.github文件夹和相关文件。

3. 上面的脚本当然不用自己写，站在巨人肩膀上，我们直接使用的是  [renzhaosy/hexo-deploy-action](renzhaosy/hexo-deploy-action)。当然，如果你看了还是不懂，你可以直接复制我的脚本（我勉强让你踩一下我的肩膀~~）, 如果你有想法，比如监听的分支不是`dep`你可以自己改。

   ```
   name: Build and Deploy
   on:
     push:
       branches:
         - dep
   jobs:
     build-and-deploy:
       runs-on: ubuntu-latest
       steps:
       - name: Checkout
         uses: actions/checkout@master
   
       - name: Build and Deploy
         uses: renzhaosy/hexo-deploy-action@master
         env:
           PERSONAL_TOKEN: ${{ secrets.ACCESS_TOKEN }}
           PUBLISH_REPOSITORY: eqshen/eqshen.github.io # The repository the action should deploy to.
           BRANCH: master  # The branch the action should deploy to.
           PUBLISH_DIR: ./public # The folder the action should deploy.
   ```

4. 到这你就快成功了，接下来给上面脚本 创建 access_token（不然你的脚本可没有权限动你仓库的代码），看到上面脚本里的 `${{ secrets.ACCESS_TOKEN }}`了吗？像不像占位符，在执行的时候等待被替换呢！

   - 点击你github主页右上角的头像，再点击Setting，进入设置。
   - 点击左侧菜单栏最后一项 `Develop Settings`
   - 然后点击 `Personal access token`
   - 再点击 `Generate new token`，然后按需求勾选赋予权限（理论上只要勾repo就行了，反正我除了delete的都勾了）
   - 最后应该是点击最下面的`Generate Token`, 然后你就会拿到一个 access_token，复制这个token,页面先别关，关了这个token就找不到了,拿到这个token之后并不是去替换上面脚本里的`secrets.ACCESS_TOKEN`
   - 浏览器新打开一个标签页，进入你仓库的主页  xxx.github.io。点击仓库的的Settings，然后再点击`Secrets` -> `Add a new secret`, Name填 `ACCESS_TOKEN`（跟脚本里的要一致，懂了吧？你仔细品），Value填上面生成的让你复制的那个 `access_token`，然后点击确定。
   - 最后你往你脚本监听的分支推送源代码，然后点击 `Actions` 查看构建日志吧，可能要一分钟才能构建好。

5. 到此部署结束，还没弄图床，有空把上面几个关键步骤的图配了~~，有问题欢迎下方留言交流。