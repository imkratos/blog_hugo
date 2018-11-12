title: 阅读ActiveMQ源码总结
date: 2016-01-07 22:06:48
tags: [java,ActiveMQ]
categories: [java]
---


### 准备工作

- 下载源码`git clone https://github.com/apache/activemq.git`
- 使用maven编译源码并且下载依赖`mvn clean install`
- 导入到开发工具eclipse`mvn eclipse:eclipse` 或者idea `mvn idea:idea`
- 默认会去maven中央仓库下载jar包，如果下载速度慢可以翻墙或者改成开源中国的镜像仓库
```xml
<mirrors>
    <mirror>
      <id>CN</id>
      <name>OSChina Central</name>
      <url>http://maven.oschina.net/content/groups/public/</url>
      <mirrorOf>central</mirrorOf>
    </mirror>
</mirrors>
```

<!--more-->


### 遇到的问题
- idea导入之后无任何显示，事实证明楼主太着急，等编译之后就有了。。


### 客户端启动
 - java客户端代码我是按照《Java消息服务》这本书中的例子完成的

```java
  public class Chat implements javax.jms.MessageListener{

	private TopicSession pubSession;
	private TopicPublisher publisher;
	private TopicConnection connection;
	private String username;

	/**
	 * @param topicFactory
	 * @param topicName
	 * @param username
	 * @throws Exception
	 */
	public Chat(String topicFactory,String topicName,String username) throws Exception {
		//使用JNDI？
		InitialContext ctx = new InitialContext();
		//拿到工场
		TopicConnectionFactory conFactory = (TopicConnectionFactory)ctx.lookup(topicFactory);
		//创建连接
		TopicConnection connection = conFactory.createTopicConnection();

		TopicSession pubSession = connection.createTopicSession(false, Session.AUTO_ACKNOWLEDGE);
		TopicSession subSession = connection.createTopicSession(false, Session.AUTO_ACKNOWLEDGE);

		Topic chatTopic = (Topic)ctx.lookup(topicName);

		TopicPublisher publisher = pubSession.createPublisher(chatTopic);
		String selector = "username=zhishuo";
		TopicSubscriber subscriber = subSession.createSubscriber(chatTopic,selector,true);

		subscriber.setMessageListener(this);

		this.connection = connection;
		this.pubSession = pubSession;
		this.publisher = publisher;
		this.username = username;
		connection.start();

	}


	public void onMessage(Message message) {
		try {
			TextMessage textMessage = (TextMessage)message;
			System.out.println(textMessage.getText());
			System.out.println(message.getJMSDestination());
		} catch (Exception e) {
			e.printStackTrace();
		}
	}

	/**
	 * 发送消息
	 * @param text
	 * @throws JMSException
	 */
	protected void writeMessage(String text) throws JMSException {
		TextMessage textMessage = pubSession.createTextMessage(username+":"+text);
		publisher.publish(textMessage);
	}

	public void close() throws JMSException {
		connection.close();
	}

	public static void main(String[] args) {
		try {
			if(args.length!=3){
				System.out.println("something is missing");
			}
			// topicFactory, topicName, username
			Chat chat = new Chat(args[0],args[1],args[2]);
			BufferedReader commondLine = new BufferedReader(new InputStreamReader(System.in));

			while(true){
				String s = commondLine.readLine();
				if(s.equalsIgnoreCase("exit")){
					chat.clone();
					System.exit(0);
				}else{
					chat.writeMessage(s);
				}
			}
		} catch (Exception e) {
			e.printStackTrace();
		}
	}

}
```
- 还需要在classpath中加入`jndi.properties`
```properties
#配置实现工场
java.naming.factory.initial=org.apache.activemq.jndi.ActiveMQInitialContextFactory
#配置activemq地址
java.naming.provider.url=tcp://localhost:61616
#这里是配置的安全机制？
java.naming.security.principal=system
java.naming.security.credentials=manager
#订阅主题的名称，可以以逗号,隔开
connectionFactoryNames=TopicCF
#topic
topic.topic1=jms.topic1
```

- 启动`Chart`类main方法时，传入`TopicCF topic1 zhishuo`这样客户端就可以正常启动，
并且发送消息了，如果启动多个客户端，然后按回车发送消息，所有的人都可以看到。


### 代码分析

- `InitialContext ctx = new InitialContext();`
这里是JMS给出的公用调用方法，new对象时调用了默认的init()方法。
`myProps = (Hashtable<Object,Object>)   ResourceManager.getInitialEnvironment(environment);`里面主要是去加载当前环境变量
和一些JVM设置的环境变量参数。
- 主要初始化都在`getDefaultInitCtx()--->NamingManager.getInitialContext(myProps);`
中完成，会在环境变量配置参数中找到`String INITIAL_CONTEXT_FACTORY = "java.naming.factory.initial";`
key值，由于我们这里配置的是`java.naming.factory.initial=org.apache.activemq.jndi.ActiveMQInitialContextFactory`，所以实例化的也就是`ActiveMQInitialcontextFactory`，最后再调用`ActiveMQInitialcontextFactory.getInitialContext(env)`这里传过去的参数是环境变量。
- 我们来到`ActiveMQInitialcontextFactory.getInitialContext(env)`方法中，此方法便是真正用来初始化队列，
主题
```java
public Context getInitialContext(Hashtable environment) throws NamingException {
       // lets create a factory
       Map<String, Object> data = new ConcurrentHashMap<String, Object>();
       String[] names = getConnectionFactoryNames(environment);
       for (int i = 0; i < names.length; i++) {
           ActiveMQConnectionFactory factory = null;
           String name = names[i];

           try {
               factory = createConnectionFactory(name, environment);
           } catch (Exception e) {
               throw new NamingException("Invalid broker URL");

           }
           /*
            * if( broker==null ) { try { broker = factory.getEmbeddedBroker(); }
            * catch (JMSException e) { log.warn("Failed to get embedded
            * broker", e); } }
            */
           data.put(name, factory); // 实际上是 tocipIF ActiveMQConnectionFactory
       }

       createQueues(data, environment);
       createTopics(data, environment);
       /*
        * if (broker != null) { data.put("destinations",
        * broker.getDestinationContext(environment)); }
        */
       data.put("dynamicQueues", new LazyCreateContext() {
           private static final long serialVersionUID = 6503881346214855588L;

           @Override
           protected Object createEntry(String name) {
               return new ActiveMQQueue(name);
           }
       });
       data.put("dynamicTopics", new LazyCreateContext() {
           private static final long serialVersionUID = 2019166796234979615L;

           @Override
           protected Object createEntry(String name) {
               return new ActiveMQTopic(name);
           }
       });

       return createContext(environment, data);
   }
```

`getConnectionFactoryNames`方法中，是为了遍历并且初始化`connectionFactoryNames`配置的Value值，我们这里配置的`connectionFactoryNames=TopicCF`，这里用到了`StringTokenizer`感觉很好用，平时自己都是用string的split，没想到还有这种用法。
然后遍历刚才解析的Value值，并且最终创建`ActiveMQConnectionFactory`对象返回。
把创建好的对像放入Hashmap中，map.put(TopicCF,ActiveMQConnectionFactory)，
> failover://tcp://localhost:61616 这是此对象初始化时url属性的默认值
初始化的时候修改成了tcp://localhost:61616

这里是每一个value值都对应一个自己的工厂类。

接下来创建Queues和Topics查找配置文件中以queue.和topic.开头的，并且生成ActiveMQQueue对象。
我们这里配置的`jms.topic1`最后也放入map.put(topic1,ActiveMQTopic)。

接下来创建了动态的队列和主题，不知何用。
dynamicQueues
dynamicTopics
并且都放入了map
最后创建了ReadOnlyContext对象，并且把相关的数据绑定都传入其中。
至此第一句代码初始化完成。

- `(TopicConnectionFactory)ctx.lookup(topicFactory)`这里是去加载实现工厂的类，由于上一个初始化最终返回的是
ReadOnlyContext对象，所以这里调用的是ReadOnlyContext.lookup方法，我们来看`public Object lookup(String name) throws NamingException {`方法，这里其实就是根据传入的名称，取出相应的工厂，由于我们在启动时传入的工厂类名是
TopicCF 所以，这里就是取出刚才放入的ActiveMQConnectionFactory工厂类，类图如下
![类图](http://ogflhfadi.bkt.clouddn.com/ActiveMQConnectionFactory%E7%B1%BB%E5%9B%BE.png)

- `conFactory.createTopicConnection()`，这里是调用的ActiveMQConnectionFactory中的方法，该类中初始化了传输协议模式
```java
 if (scheme.equals("auto")) {
     connectBrokerUL = new URI(brokerURL.toString().replace("auto", "tcp"));
  } else if (scheme.equals("auto+ssl")) {
     connectBrokerUL = new URI(brokerURL.toString().replace("auto+ssl", "ssl"));
 } else if (scheme.equals("auto+nio")) {
     connectBrokerUL = new URI(brokerURL.toString().replace("auto+nio", "nio"));
  } else if (scheme.equals("auto+nio+ssl")) {
     connectBrokerUL = new URI(brokerURL.toString().replace("auto+nio+ssl", "nio+ssl"));
  }
```
如果以上都没有配置，默认使用TCP协议传输，在`TransportFactory.connect()-->findTransportFactory()`中，查找资源对应的配置协议，如果没有查到，使用默认的在
`private static final FactoryFinder TRANSPORT_FACTORY_FINDER = new FactoryFinder("META-INF/services/org/apache/activemq/transport/");`此文件目录中，因为这里是用的`tcp://localhost:61616`所以这里使用的就是`META-INF/services/org/apache/activemq/transport/tcp`文件，此文件中实际配置的是TCP的传输器
```properties
class=org.apache.activemq.transport.tcp.TcpTransportFactory
```
然后调TcpTransportFactory.doConnect()进行连接，至此传输器创建完成。
- 接下来创建ActiveMQConnection连接
- 最后启动连接，并且设置一些默认值，至此创建连接完成，并返回连接。

- `TopicSession pubSession = connection.createTopicSession(false, Session.AUTO_ACKNOWLEDGE);`创建发布和订阅Session，
  最终调用的是ActiveMQConnection.createTopicSession()，创建ActiveMQSession对像并返回。

- `Topic chatTopic = (Topic)ctx.lookup(topicName);`加载主题，这里的值为`topic1`，
  因为在前边初始化的时候已经放入了对像，所以这里的值为`ActiveMQTopic`，调用ActiveMQSession.createPublisher() 和 createSubscriber() 创建发布者ActiveMQTopicPublisher和消费者ActiveMQTopicSubscriber。

- 创建完成之后，把自己设置为订阅者的监听器，最终启动连接，客户端启动代码分析完成，因为有一些自己也不是很理解，所以未能表达。



### 服务端启动
- 以下是服务端启动的类图调用关系:
![启动类图](http://ogflhfadi.bkt.clouddn.com/active_server_start_seq.png)

*这里还了解了java -D 是可以把一些参数设置到JVM中的*

服务端启动这里充分体现了命令设计模式的案例，我认为是很好的学习例子，最终调用的是StartCommand.runTask()方法，类中有如下几个重要功能
  - 初始化broker
  - 添加JVM shutdownhook 停止时进行清理

下面来看`broker = BrokerFactory.createBroker(configURI);`此类中在创建工厂时也使用了文件配置的方法由于默认是使用的`xbean:activemq.xml`资源文件，所以在创建工厂时就是实例化的
`META-INF/services/org/apache/activemq/broker/`xbean文件中配置的类，
```properties
class=org.apache.activemq.xbean.XBeanBrokerFactory
```
实际调用的是`XBeanBrokerFactory.createBroker()`此类中创建了Spring的`ResourceXmlApplicationContext`实现对象，但不知这里为何要这样。这里最后还调用了spring框架中refresh方法，此方法会初始化spring整个框架。
最终创建一个BrokerService返回，broker中包含了所有上下文环境。

KahaDBPersistenceAdapter 调用关系图
![调用关系](http://ogflhfadi.bkt.clouddn.com/active_db_ref.png)

KahaDBPersistenceAdapter 用例图

![调用关系](http://ogflhfadi.bkt.clouddn.com/active_server_db_ref.png)
最后启动，至此服务启动完成。















### 问题
