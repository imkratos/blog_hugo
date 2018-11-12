title: 'hexo 搭建 github pages  问题'
date: 2014-05-31 18:45:03
tags: [hexo,github pages]
categories: [hexo]
---
由于自己想搭建一个免费的博客，所以找到了万能的github，但在搭建的过程中遇到了几个小问题，所以记下来
基本搭建的方法都是参考于[利用github搭建博客](http://ibruce.info/2013/11/22/hexo-your-blog/)
这个博主已经写的很全了，搭建的过程就不一一提了说下遇到的问题
1.	hexo上传后博客还是官方的
  	建立了github pages的项目，并且使用了官方的建立建好了一个博客，官方默认会在master分支上开出一个gh-pages分支，然后查了官网资料说所有的内容都是显示gh-pages分支上的，但是我用hexo却传不到gh-pages分支上，并且传到了master分支内容也不会显示，后来找到原因是因为我建立的项目名字不是以#你的github帐户.github.io#命名的，所以会使hexo的上传程序传到master分支上也不会起作用，把项目名修改之后就可以了。
<!--more-->
2.	添加多说的问题
  在官网注册后，复制了多说代码，按[教程](http://www.leejianyang.com/2014/05/25/duoshuo_tutorial/)做完之后发现多说模块不显示，所以查看了comment.ejs源代码
			<% if (config.disqus_shortname && page.comments){ %>
  这里是2个判断，把config.disqus_shortname && 删掉之后，就可以正常使用了。
