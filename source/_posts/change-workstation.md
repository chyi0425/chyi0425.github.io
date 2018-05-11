---
title: change_workstation
date: 2018-05-12 01:15:54
categories: tools #文章文类
tags: [git,hexo] #文章标签，多于一项时用这种格式
---
hexo部署blog到github确实很方便，下载相关插件，然后一条命令就搞定了。不明白？参考这篇[hexo配置](http://chyi0425.github.io/2018/05/11/hello-world/)，神马？不知道怎么配置nodejs环境？那就参考[这里](http://jingyan.baidu.com/article/9113f81b01c4e72b3214c7d3.html)。

如果只满足于简单的部署发布，现在就完全够用了。可是有人就喜欢在不同时间、不同地点、不同电脑上写文章( •̀ ω •́ )y！(嗯，那个人就是我～)。这样的问题如何解决呢，总体思路是使用github的分支思想，将所有文件放在hexo分支下，将public文件夹通过插件自动发布放在master分支以供加载展示。同时因为有配置.gitignore文件，无需担心node_modules等文件被手动发布到hexo分支下，达到文件分类存放的目的。follow下面几步让你赏心悦目、随心所欲写文章。
<!-- more -->

### 步骤

1). 创建仓库，http://chyi0425.github.io；
2). 创建两个分支：master 与 hexo；
3). 设置hexo为默认分支（因为我们只需要手动管理这个分支上的Hexo网站文件）；
4). 使用git clone git@github.com:chyi0425/chyi0425.github.io.git拷贝仓库；
5). 在本地chyi0425.github.io文件夹下依次执行

``` bash
$ npm install hexo
$ hexo init
$ npm install
$ npm install hexo-deployer-git  #此时当前分支应显示为hexo
```
6). 修改_config.yml中的deploy参数，分支应为master；
7). 依次执行

``` bash
$ git add .  #注意最后的.,这个.表示当前目录
$ git commit -m "..." 
$ git push origin hexo  #提交网站相关的文件
$ hexo g -d  #执行生成网站并部署到GitHub上
```


### 其他可能需要用到的命令

``` bash
$ git branch hexo  #创建hexo分支
$ git chechout hexo  #切换到hexo分支
$ git clone https://github.com/litten/hexo-theme-yilia.git themes/yilia
$ rm -rf *  #删除所有文件，隐藏文件不会删除 
$ git status
```


### 发布新文章

新建markdown文件，编辑文章。

``` bash
$ hexo new "new blog"
```

在本地对博客进行修改（添加新博文、修改样式等等）后，通过下面的流程进行管理。

``` bash
$ git add .
$ git commit -m "..."
$ git push origin hexo #指令将改动推送到GitHub(此时当前分支应为hexo)
```

最后执行

``` bash
hexo g -d
```
发布网站到master分支上。