---
layout: post
title: "GitHub Pages内容维护（非程序员）"
date: 2021-09-22 09:00 -0700
last_modified_at: 2021-09-24 11:45 -0700
toc: true
tags: [technology, chinese]
---

> 本文假设你已经有一个搭建好的GitHub Pages repository，有系统管理员帮你处理技术细节，你只需要做一些简单的内容维护：增、删、改文章内容，更新背景图片等。

## 创建GitHub用户号与授权

如果你还没有一个GitHub用户号，可以到[这里](https://github.com/)申请，按网页提示操作就可以了，这里不再赘述。然后，把你的用户名（不需要密码哦！）提供给系统管理员，作必要的授权。

## 使用Git

Git是一个版本维护工具，可以到[这里](https://git-scm.com/downloads)下载并安装。
GitHub Pages的内容（包括代码和设置文件）是以Git repository的形式存在的，repository里面有一个master branch（主干版本），我们也可以从它分出其他的branch，但简单起见，这里只使用master。在命令行运行下面的命令，你将在本地建立一个和GitHub repository关联的文件夹，而内容就是master branch的：

```
git clone <REPOSITORY_LINK> [LOCAL_FOLDER_NAME]
```

运行命令的时候，可能会提示你登陆，上一步创建的用户号就派上用场了。可能你会奇怪，为什么要用命令行，而不是在网页上直接下载zip文件呢？因为git可以帮助你管理内容改动。比如你修改了一些文章、添加了新的文章，运行下面的命令，你会看到改动了的文件列表：

```
git status
```

要查看文件内容的具体改动，你可以用这个命令：

```
git diff head <FILE_PATH>
```

当你对改动满意了，就可以把它提交到本地的master branch。运行以下命令，第一条是把所有改动加入清单，第二条是提交清单里所有文件：

```
git add .
git commit -m "<COMMENTS>"
```

最后，你还要把本地的改动上传到GitHub服务器上：

```
git pull
git push
```

我们又不是打太极，为什么要先来一个pull再push呢？这是因为Git可以管理多人协作，你上次clone或者pull到本地的内容，在服务器上可能已经被别人修改过了。再pull一次，Git会帮你把本地的改动与服务器上的合并，如果有不能自动解决的冲突，命令会中止。这时候，你需要手动修改有冲突的文件，再重新提交本地改动、上传服务器的操作。如果push成功了，GitHub会在几分钟后更新你的网页。如果网页一直没有更新，可能是你的改动导致了编译错误，请联系系统管理员解决。

### Git的后悔药

当你想重置某些改动，如果你还没有提交（commit），可以用这个命令：

```
git checkout -- <FILE_PATH>
```

如果已经提交，则须要使用以下命令。而且你只能重置整个提交清单，而不是单个文件了：

```
git revert <COMMIT_ID>
```

怎么找到你的commit ID呢？你可以运行以下命令列出所有以往的提交：

```
git log
```

不过，更普遍的需求是，你只需要重置上一个提交，那么可以直接运行：

```
git revert head
```

Git还有很多功能，如果感兴趣，可以阅读[官方文档](https://git-scm.com/docs)。

## GitHub Pages的目录和格式

GitHub Pages使用[Jekyll](https://jekyllrb.com/) 去生成静态网页。当然，你不须要完全理解Jekyll。以内容更新为导向，你只须知道在哪里找到相应的页面和图片，以及了解页面内容的基本格式就可以了。

一般的页面在根目录下，后缀是`.html`或者`.md`。比如，`./about.md`对应的是`http://<YOUR_WEBSITE>/about.html`。但像`index.html`比较特殊，它是默认的主页。至于博客文章的页面，则存放在`_posts`目录下，文件名的格式是`YEAR-MONTH-DAY-title.md`，比如，`2021-09-22-how-to-earn-your-first-million.md`。所以，要添加新文章的话，你只要按照这个命名方式在该目录添加新文件就可以了。而图片，则可以放在任何目录下，一般习惯放在`img`或者`assets`目录下。注意，当在页面中引用图片时，须提供正确的文件路径。

```
<root>
│   README.md
│   about.md    
│   contact.html
│   index.html
├── _posts
│   │   2021-09-22-how-to-earn-your-first-million.md
└── img
    │   bg-index.jpg
    │   bg-post.jpg
    └── posts
        │   million_bg.jpg
        │   million_data.jpg
```

页面文件开头一般包含[font matter](https://jekyllrb.com/docs/front-matter/)的数据信息，这部分是以yaml格式存在的，而正文则支持HTML和Markdown格式。先来看一个例子：

```
---
layout: post
title: "How to Earn Your First Million"
date: 2021-06-01 17:21:00 -0700
background: '/img/posts/million_bg.jpg'
---

Lorem ipsum dolor sit amet, consectetur adipiscing elit. Cras quis orci placerat, molestie velit sed, lacinia nisi. Curabitur viverra id felis non cursus. Phasellus et sodales ligula. Sed imperdiet sed dui vitae aliquet. Quisque sed luctus dolor, sed congue ex. In iaculis turpis magna, non mattis nisi pellentesque id. Nam in metus nec dolor dapibus imperdiet et ut nunc.

<img src="{% raw %}{% link img/posts/million_data.jpg %}{% endraw %}" alt="bath tub 1" width="600"/>
```

开头在两个`---`之间的部分就是font matter，除了`layout`外（这是页面使用的模板），其他像`title`、`date`和`background`都可以按需要改动。`background`的图片需要添加到相应的目录下，在例子里你需要确保`/img/posts/`下有对应的图片文件。

接下来就是文章内容，Markdown提供了很多文本格式支持，你可以参考[GitHub的相关文档](https://guides.github.com/features/mastering-markdown/)。例子中的文本没有使用任何格式，所以就是基本的段落文字。
但例子中的配图，则直接使用了[HTML img标签](https://www.w3schools.com/html/html_images.asp)。Markdown格式的图片不便于调整图片尺寸，而HTML可以通过图片属性`width=600`，把大图片的宽度缩小为600像素（高度按比例缩小）。另外，图片路径，须要引用在`{% raw %}{% link %}{% endraw %}`里，以便编译时能生成正确的地址。

> :warning: 关于Markdown，GitHub Pages支持两种处理器： [Kramdown](https://kramdown.gettalong.org/)和[GitHub Flavored Markdown (GFM)](https://github.github.com/gfm/)，他们只有细微的差别。详情请看[这里](https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/setting-a-markdown-processor-for-your-github-pages-site-using-jekyll)。

## 实践出真知

到这里你可以动手尝试了，不要害怕改坏了，Git的后悔药并不昂贵，而且遇到问题还有系统管理员作为你的后盾。Go and good luck! :smiley:
