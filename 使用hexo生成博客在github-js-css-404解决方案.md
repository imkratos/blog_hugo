---
title: 使用hexo生成博客在github js css 404解决方案
date: 2016-11-04 13:32:58
tags: [hexo]
categories: [hexo]
---

### 问题
博主最近又开始写博客了，发现使用hexo next 主题，上传到github上之后，所有的vendors文件夹下的资源访问404，博主查了一些资料没有解决，故给github官方写了一封邮件，官方这样回复
> Thanks for reaching out! We recently updated to Jekyll v3.3, which ignores the vendor folder by default. If you're not using Jekyll, you can add a .nojekyll file to the root of your repository to disable Jekyll from building your site. Once you do that, your site should build with your vendor folder.

所以根据提示，在要目录下建立.nojekyll文件即可恢复正常，另外在hexo里，如果把文件放在source目录下，不会被生成到public目录中，根据github上网友的提示，把.nojekyll文件放在hexo主目录.deploy_git/ 文件夹下即可正常使用hexo d 上传，并且此文件也会上传到根目录。

### 2016-11-7号更新
next主题源码更新了，把vendors目录改名了，更新为最新代码即可解决。
