---
title: hugo搭建简明指南
weight: 2
toc: true
---

## 安装&初始化Hugo

```bash
sudo pacman -S hugo
```

在自己喜欢的目录，比如`blog`，起一个自己喜欢的名字，比如`myblog`执行如下命令

```bash
hugo new site myblog
cd myblog
git init
```

以book主题为例

```bash
git submodule add https://github.com/alex-shpak/hugo-book.git themes/book
```

之后更新主题只需要`git submodule update`

## 配置站点文件

我用micro打开

```bash
micro hugo.toml
```

输入如下内容

```to
baseURL = 'https://jadeble.github.io/'
languageCode = 'zh-cn'
title = '我的知识库'
theme = 'book'

[params]
  BookSection = 'docs'
  BookSearch = true
  BookComments = false

# 左侧菜单（可按需增减）
[[menu.main]]
  name = '文档'
  url = '/docs/'
  weight = 1
[[menu.main]]
  name = '标签'
  url = '/tags/'
  weight = 2
```

在content文件夹下创建分类文件夹，例如

```ba
mkdir -p content/docs/linux content/docs/git content/docs/web
```

每一个文件夹中需要有一个`_index.md`，开头格式如下

```mark
---
title: linux
weight: 2
---
```

`weight`决定了该目录在第几个，文章的开头也可以如此

执行如下内容开启预览

```bash
hugo server -D --disableFastRender
```

## 推送到GitHub

这里采用源码在`main`分支，编译得到的文件在`gh-pages`分支

直接写了一个`fish`脚本，应该能看懂流程

```bash
function deploy
    # 1. 推送源码
    cd /home/jade/blog/myblog
    git add .
    git commit -m "更新源文件" # 若没有修改会报错，可忽略
    git push origin main

    # 2. 生成网站
    hugo

    # 3. 部署到 gh-pages
    cd public
    if not test -d .git
        git init
    end
    # 下面一行：如果 origin 不存在就添加，存在则更新地址（不报错）
    if test -z (git remote get-url origin 2>/dev/null)
        git remote add origin git@github.com:Jadeble/jadeble.github.io.git
    else
        git remote set-url origin git@github.com:Jadeble/jadeble.github.io.git
    end
    git add .
    git commit -m "Deploy website"
    git branch -m gh-pages
    git push -f -u origin gh-pages
    cd ..
    echo "✅ 网站源码和网页都已更新！"
end
```



{{< big-center >}}
大功告成
{{< /big-center >}}