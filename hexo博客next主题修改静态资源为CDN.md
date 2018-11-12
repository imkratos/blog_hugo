---
title: hexo博客next主题修改静态资源为CDN
date: 2016-11-15 23:55:21
tags: [hexo,next,cdn]
categories: [hexo]
---

## 问题
博主最近看到博客打开速度非常慢，点开chrome的开发者工具查看是由于css,js,图片加载太慢，故`css,js`换成了国内的cdn，图片换成了[七牛云](http://www.qiniu.com/)。
打开主题配置文件`_config.yml`以下为修改方式:
<!--more-->
```yml
vendors:
  # Internal path prefix. Please do not edit it.
  _internal: vendors

  # Internal version: 2.1.3
  jquery: //cdn.bootcss.com/jquery/2.1.3/jquery.min.js

  # Internal version: 2.1.5
  # Fancybox: http://fancyapps.com/fancybox/
  fancybox: //cdn.bootcss.com/fancybox/2.1.5/jquery.fancybox.pack.js
  fancybox_css: //cdn.bootcss.com/fancybox/2.1.5/jquery.fancybox.min.css

  # Internal version: 1.0.6
  fastclick: //cdn.bootcss.com/fastclick/1.0.6/fastclick.min.js

  # Internal version: 1.9.7
  lazyload: //cdn.bootcss.com/jquery_lazyload/1.9.7/jquery.lazyload.min.js

  # Internal version: 1.2.1
  velocity: //cdn.bootcss.com/velocity/1.3.1/velocity.min.js

  # Internal version: 1.2.1
  velocity_ui: //cdn.bootcss.com/velocity/1.3.1/velocity.ui.min.js

  # Internal version: 0.7.9
  ua_parser: //cdn.bootcss.com/UAParser.js/0.7.12/ua-parser.min.js

  # Internal version: 4.4.0
  # http://fontawesome.io/
  fontawesome: //cdn.bootcss.com/font-awesome/4.6.2/css/font-awesome.min.css

```

最终打开时间由原来的20-40s，缩短到现在的5s左右，可能不稳定，但足够了。

以下是几个国内比较好的cdn网站
- http://www.bootcdn.cn/
- https://www.staticfile.org/
- http://cdn.code.baidu.com/
