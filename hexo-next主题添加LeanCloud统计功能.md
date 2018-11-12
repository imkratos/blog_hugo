---
title: hexo next主题添加LeanCloud统计功能
date: 2016-11-10 20:51:47
tags: [hexo,next,leancloud]
categories: [hexo]
---

#### 遇到的问题
网上能查到很多next主题添加LeanCloud主题的方法，但我看都是说在站点的`_config.yml`中添加
```
leancloud_visitors:
  enable: true
  app_id: appid
  app_key: key
```
与是我也在站点的`_config.yml`中添加了，但不起作用。
与是我又去主题的目录中添加了，报错。
最后找到原因，是因为主题的`_config.yml`配置文件中已经自带了，在主题的`_config.yml`配置文件中308行左右，在这里直接配置即可。

#### 参考资料
[next主题添加LeanCloud](https://notes.wanghao.work/2015-10-21-%E4%B8%BANexT%E4%B8%BB%E9%A2%98%E6%B7%BB%E5%8A%A0%E6%96%87%E7%AB%A0%E9%98%85%E8%AF%BB%E9%87%8F%E7%BB%9F%E8%AE%A1%E5%8A%9F%E8%83%BD.html)
