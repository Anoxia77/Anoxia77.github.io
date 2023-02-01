---
title: Hexo搭建&Github部署
---
## hexo 下载与构建
> [https://hexo.io/zh-cn/docs/setup](https://hexo.io/zh-cn/docs/setup) 安装&搭建教程

基本步骤
```shell
# 下载
npm install -g hexo-cli

# 添加环境变量 把hexo 安装目录下的 node_moudle 目录添加到环境变量，这样就可以直接使用 hexo 命令了

# 随便找一个目录,初始化，进入，下载依赖
hexo init my-blog
cd my-blog
npm install

# 然后修改 _config.yml 配置文件 跟着 配置介绍修改https://hexo.io/zh-cn/docs/configuration
```
到这里，在本地的hexo 就基本构建成功了。
这个时候可以使用 `hexo server`启动试一下，看能不能启动访问。
## 部署到github或gitee
操作是一样，只记录 部署到 github 的操作
首先，最好检查一下自己是否有 添加github的私钥，因为要提交，没有的话，到时候可能会要输入密码，反正弄起来很 **烦。**所以建议，先把私钥加好。这样后面的操作就会`很丝滑`，私钥没有配，不知道咋配，可以看一下这个，有大佬写的很详细，我就直接搬了 [https://www.shuzhiduo.com/A/MAzAqP1z9p/](https://www.shuzhiduo.com/A/MAzAqP1z9p/)
首先 创建一个 `<你Github的用户名>.github.io` 的代码仓库。然后打开刚刚创建的博客目录，我这里都以`my-blog`记录，后面说这个文件的时候，你就要知道，这是哪个东西。
打开 `_config.yml`文件，找到下面 的内容,如果第一次弄的话，这里应该只有一个 deploy，可以直接在这个文件里面搜`deploy`。然后把你上面建的 github仓库地址加上。到这一步就快完成了。
```shell
# Deployment
## Docs: https://hexo.io/docs/one-command-deployment
deploy:
  type: git
  repository:
    github: git@github.com:Anoxia77/Anoxia77.github.io.git
```
在这里有个坑，下面的这个配置，好像默认的文档 url 默认是 有加一个 xxxx/projet ,这个project 要注意一下，如果自己的目录有就加，没有就不要加，不然到时候 css，js文件就会找不到
```shell
# URL
## Set your site url here. For example, if you use GitHub Page, set url as 'https://username.github.io/project'
url: https://Anoxia77.github.io
permalink: :year/:month/:day/:title/
permalink_defaults:
pretty_urls:
  trailing_index: true # Set to false to remove trailing 'index.html' from permalinks
  trailing_html: true # Set to false to remove trailing '.html' from permalinks
```
然后就可以直接开始打包部署了，
```shell
hexo clean 清除上一次打包的
hexo g 构建
hexo d 打包 打包需要安装一个插件 npm install --save hexo-deployer-git
```
打包完成后，会把静态代码上传到你创建的哪个仓库，然后就可以看了。我的地址就是  [https://anoxia77.github.io](https://Anoxia77.github.io)  ,换一下前面的名称就可以了。
