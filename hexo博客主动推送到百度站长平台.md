---
title: hexo博客主动推送到百度站长平台
date: 2016-11-26 17:19:09
tags: [hexo,百度站长]
categories: [hexo]
---
#### 推送百度站长平台
百度站长平台有几种方式提交自己的链接
1. 主动推送(实时)
2. 自动推送
3. sitemap

或者自己手工提交，这里主要说以上三种方式。

<!--more-->

##### 主动推送
第一种方式使用了`baidu_url_submitter`这个工具：
首先在hexo根目录安装插件`npm install hexo-baidu-url-submit --save`
然后在根目录`_config.yml`中增加
```yml
baidu_url_submit:
  count: 1 ## 提交的链接数
  host: www.zhishuo.info ## 在百度站长平台中注册的域名
  token: token ## 百度站长平台里的token，在链接提交-自动提交-主动推送中有
  path: baidu_urls.txt ## 这个是会自动生成要推送的url地址
```
`count`是控制几条数据会生成在`baidu_urls.txt`中，这里推荐首次使用时填写你所有博客的数量，然后修改为1即可。
这里还要注意一个问题，站点配置文件中`url: http://www.zhishuo.info`这里一定要带上www要不然推送不成功。
最后把deploy调整一下，就可以正常使用了。
```yml
deploy:
  - type: git
  repository: https://github.com/imkratos/imkratos.github.io.git
  branch: master
  - type: baidu_url_submitter # baidu push
```
 - `hexo g`会生成baidu_urls.txt文件。
 - `hexo d`会在部署完git代码之后，把新增的url也会推送到百度。


 #####  自动推送
 由于我是使用的hexo next主题，next主题中自带了此功能，在next主题根目录下`_config.yml`中找到`baidu_push: false`改为`true`即可。

 ##### sitemap
 此种方式好像被github官方禁止了，博主添加了发现效果不是很明显，需要使用的同学们可自行查找。


 `baidu_url_submitter`[作者](http://hui-wang.info/2016/10/23/Hexo%E6%8F%92%E4%BB%B6%E4%B9%8B%E7%99%BE%E5%BA%A6%E4%B8%BB%E5%8A%A8%E6%8F%90%E4%BA%A4%E9%93%BE%E6%8E%A5/)
