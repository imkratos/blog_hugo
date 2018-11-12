title: '读spring源码深度解析'
date: 2015-11-12 11:35:44
tags: [spring,spring源码]
categories: [java]
---
工作也有几年了，以前也尝试过想要读spring源码，但由于前几年没有参考资料，所以很难去理解，最近发现一本`spring源码深度解析`，怀着试试看的心情，踏上了源码之路。

记录一下自己遇到的问题，初次读完之后感觉到了spring的强大之处，处处都是设计模式，把代码拆的很分散，但理解上就很难了。

1. 源码下载，spring最新的git源码地址为[spring源码](https://github.com/spring-projects/spring-framework.git)
2. 由于spring3.X以后就是使用gradle构建了，所以还需要安装gradle，Mac OS 可以使用HomeBrew brew install gradle 即可
3. 安装好gradle以后，可以在想要查看的spring项目下执行gradle eclipse 生成eclipse的配置文件
4. 最后在Eclipse->import->existing Eclipse projects进行导入就可以了
5. 书中有一个查看类实现的接口图，一开始不知道在哪使用，后来发现对着想要查看的类右击，出现选项菜单，点击->Open Type Hierarchy 即可，默认是显示的子类实现，然后点击上方小图标show the supertype Hierarchy即可
6. 书中作者看到了哪个类中没有做明确提示，如果找不到需要自行搜索，我用的[STS](https://spring.io/tools)，commond+shift+l 可按关键字搜索
