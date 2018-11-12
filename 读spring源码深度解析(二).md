title: '读spring源码深度解析(二)'
date: 2016-03-09 17:16:21
tags: [spring,spring源码]
categories: [java]
---
***不明白***
`DefaultListableBeanFactory.java`中使用了`AccessController.doPrivileged()`方法，个人理解此方法好像是以当前上下文执行一些代码。
`javaUtilOptionalClass`这个属性，是用了JDK8的`java.util.Optional`,此类是用来避免在Java中出现各种null的问题,但不知道为什么是写在了静态代码块中，而且是以动态加载的方式来加载的。


***初始类***
从`DefaultListableBeanFactory.java`类的getBean()中，开始对整个工厂类进分析。

![类的关系调用图](http://ogflhfadi.bkt.clouddn.com/DefaultListableBeanFactory.png)

静态导入，如果你多次用到某个工具类的静态方法，可以使用静态导入，这样使代码更整洁美观。
`SimpleAliasRegistry.java`这个类实现了对Alias的一些操作。
`BeanFactory.java`接口定义一些对Bean的基本操作。
`DefaultSingletonBeanRegistry.java`对spring对Bean的一些操作做了各种封装，创建，消毁，依赖等。
`AbstractAutowireCapableBeanFactory.java`对`AutowireCapableBeanFactory`定义的方法进行了实现，此类
`XmlBeanDefinitionReader.java`实现了用xml解析资源文件，并且实现了`DefaultListableBeanFactory.java`类

<!--more-->

xml如果要使用DTD文件来验证的话，需要在xml文件头中加入`<!-- DOCTYPE`

在解析DTD和XSD时，使用的是不同的机制，如果是DTD的话，会直接取到systemid在当前目录下寻找，如果是xsd时，会在META/schemas中找到对应的systemid

在解析xml时，如果设置了ResourceLoader，就使用用户设置的，如果没有设置，就使用默认的ResourceEntityResolver解析，解析Xml是使用的SAX方法解析，解析出来Document之后，会把Document进行解析，进行Bean的注册。

在注册的时候，使用了一个class类的cast方法，感觉这个方法使用的比较好，以前没有见过这种强制转换类型的使用，使用的`DefaultBeanDefinitionDocumentReader.java`来注册Bean，这里注册Bean的那些Map，是使用的`SimpleBeanDefinitionRegistry.java`类，如果没有指定应该是`DefaultSingletonBeanRegistry.java`?,注册的时候还会返回此次注册的类的数量。
创建这个对象XmlReaderContext不知道是什么作用，里面是对XmlBeanDefinitionReader的一些封装，解析的时候还留了一前一后，供子类自己实现。`DefaultBeanDefinitionDocumentReader.java`中是对xml的解析，先根据标签来，然后再解析其中所有的属性，把xml组装成Java对象`GenericBeanDefinition.java`

解析Bean标签过程，获取id和name，如果没有指定id，就用name做这个类的key
检查类名的唯一性
判断此xml是否配置了一系列Spring的属性，并且在对应的Java对象中，设置对应的属性。
