---
title: Spring JMS -- 基于activemq
date: 2017-06-25 15:21:48
categories:
- java
tags:
- spring
- activemq
---

### 缘起
最近的项目使用activemq比较多，遇到了点坑，总结一下。用spring提供的jmsTemplate，包装一下jms的原始接口，使得消息发送代码很简洁；同时DefaultMessageListenerContainer，对消费消息也做了一定的包装，可以配置消费者数量的上下限。

### pom依赖
```
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>${spring-framework.version}</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-jms</artifactId>
    <version>${spring-framework.version}</version>
</dependency>
<dependency>  
    <groupId>org.apache.activemq</groupId>  
    <artifactId>activemq-core</artifactId>  
    <version>5.7.0</version>  
</dependency> 
```

### 消息发送：jmsTemplate
如果消费端需要给多方消费使用，那么最好采用VirtualTopic的模式，具体方法就是在发送端将队列名字命名VirtualTopic.xxxx，同时把队列设置成topic；在消费端，以queue的形式消费，队列名字设置为Consumer.xx.VirtualTopic.xxxx，这种订阅消费是持久订阅，所以一旦消费，不用担心中途停止而丢失消息，不过要注意消息积压。

#### 1.ActiveMQConnectionFactory
* spring配置
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans-4.2.xsd
       http://www.springframework.org/schema/context
       http://www.springframework.org/schema/context/spring-context-4.2.xsd">

    <bean id="idolScoreMqPropertyConfig"
          class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
        <property name="fileEncoding" value="UTF-8" />
        <property name="locations">
            <list>
                <value>classpath*:properties/acmq-${profiles.active}.properties</value>
            </list>
        </property>
    </bean>

    <!-- ActiveMQ Topic -->
    <bean id="idolScoreDestination" class="org.apache.activemq.command.ActiveMQTopic">
        <constructor-arg index="0">
            <value>${activemq.idolscore.queuename}</value>
        </constructor-arg>
    </bean>

    <!-- ActiveMQ 连接工厂 -->
    <bean id="idolScoreConnectionFactory" class="org.apache.activemq.ActiveMQConnectionFactory">
        <property name="brokerURL">
            <value>${activemq.idolscore.server}</value>
        </property>
        <property name="userName">
            <value>${activemq.idolscore.user}</value>
        </property>
        <property name="password">
            <value>${activemq.idolscore.password}</value>
        </property>
    </bean>

    <!-- jms queue template -->
    <bean id="jmsTemplate"
          class="org.springframework.jms.core.JmsTemplate">
        <property name="connectionFactory" ref="idolScoreConnectionFactory" />
        <property name="defaultDestination" ref="idolScoreDestination" />
    </bean>

    <bean id="bombIdolScoreMsgProducer"
          class="selflearning.mq.BombIdolScoreMsgProducer">
        <property name="jmsTemplate" ref="jmsTemplate"/>
    </bean>

</beans>
```

* 消息发送代码
```
public class BombIdolScoreMsgProducer {
    private static BombIdolScoreMsgProducer producer = null;
    private JmsTemplate jmsTemplate;

    private BombIdolScoreMsgProducer() {}

    public static BombIdolScoreMsgProducer instance() {
        if (producer == null) {
            synchronized (BombIdolScoreMsgProducer.class) {
                if (producer == null) {
                    ApplicationContext context = new ClassPathXmlApplicationContext(
                            new String[] {"classpath*:spring/idolscore-mq.xml"});
                    producer = context.getBean(BombIdolScoreMsgProducer.class);
                }
            }
        }

        return producer;
    }

    public void send(final BombIdolScoreMessage message) {
        jmsTemplate.send(new MessageCreator() {
            @Override
            public Message createMessage(Session session) throws JMSException {
                return session.createTextMessage(message.toString());
            }
        });
    }

    public void setJmsTemplate(JmsTemplate jmsTemplate) {
        this.jmsTemplate = jmsTemplate;
    }
}
```

消息发送如果按照上面的方式，完全可以工作，但是效率不高，每次发送一条消息需要新建connection、session和producer，发完之后，还需要关闭，这个过程需要请求mq broker7次。
> JMS is designed for high performance. In particular its design is such that you are meant to create a number of objects up front on the startup of your application and then resuse them throughout your application. e.g. its a good idea to create upfront and then reuse the following
* Connection
* Session
* MessageProducer
* MessageConsumer
The reason is that each create & destroy of the above objects typically requires an individual request & response with the JMS broker to ensure it worked. e.g. creating a connection, session, producer, then sending a message, then closing everything down again - could result in 7 request-responses with the server!

#

#### 2.PooledConnectionFactory
解决上面低效问题的方法就是池化连接，需要引入新的依赖，实测效果显示池化之后确实提高好几倍。
```
        <dependency>
            <groupId>org.apache.activemq</groupId>
            <artifactId>activemq-pool</artifactId>
            <version>5.14.1</version>
        </dependency>
```

spring配置修改，可以设置最大连接数，最大active session功能
```
    <!-- a pooling based ActiveMQ 连接工厂 -->
    <bean id="idolScoreConnectionFactory" class="org.apache.activemq.pool.PooledConnectionFactory" destroy-method="stop">
        <property name="maxConnections" value="3"/>
        <property name="maximumActiveSessionPerConnection" value="10"/>
        <property name="connectionFactory">
            <bean class="org.apache.activemq.ActiveMQConnectionFactory">
                <property name="brokerURL">
                    <value>${activemq.idolscore.server}</value>
                </property>
                <property name="userName">
                    <value>${activemq.idolscore.user}</value>
                </property>
                <property name="password">
                    <value>${activemq.idolscore.password}</value>
                </property>
            </bean>
        </property>
    </bean>
```

* 遇到的问题1：Initialization of bean failed; nested exception is java.lang.reflect.MalformedParameterizedTypeException
这是因为pom里依赖了commons-pool和commons-pool2，将activemq-pool版本升到5.14就都依赖commons-pool2了，这样就米有问题了。
* 遇到的问题2：failover的两个broker同时收到消息
解决方案很简单，将failover的url里randomize=false

### 消息接收：DefaultMessageListenerContainer
* spring配置
```
    <!-- 异步接收Queue消息Container -->
    <bean id="jmsTemplate"
          class="org.springframework.jms.listener.DefaultMessageListenerContainer">
        <property name="connectionFactory" ref="connectionFactory"/>
        <property name="destination" ref="destination"/>
        <property name="messageListener" ref="entityUpdateMessageListener"/>
        <!-- 初始5个Consumer, 可动态扩展到10 -->
        <property name="concurrentConsumers" value="5"/>
        <property name="maxConcurrentConsumers" value="10"/>
        <!-- 设置消息确认模式为Client -->
        <property name="sessionAcknowledgeModeName" value="AUTO_ACKNOWLEDGE"/>
        <property name="autoStartup" value="true"/>
    </bean>
	
    <bean id="entityUpdateMessageListener"
         class="selflearning.controller.PopCircleMessageListener"/>

    <bean id="destination" class="org.apache.activemq.command.ActiveMQQueue">
        <constructor-arg index="0">
            <value>${paopao.activemq.circle.queuename}</value>
        </constructor-arg>
    </bean>
```

* 消息处理的代码
```
public class PopCircleMessageListener implements MessageListener {

    public void onMessage(Message message) {
        if (message instanceof TextMessage) {
            TextMessage textMessage = (TextMessage) message;
            try {
                String msgJson = textMessage.getText();
                System.out.println("----> Raw msg: " + msgJson);
            } catch (JMSException e) {
                e.printStackTrace();
            }
        }
    }
}
```


### reference
1. http://elim.iteye.com/blog/1893038
2. http://activemq.apache.org/spring-support.html
3. https://codedependents.com/2009/10/16/efficient-lightweight-jms-with-spring-and-activemq/
4. http://wiki.qiyi.domain/pages/viewpage.action?pageId=6817067#ActiveMQ开发者手册-FAQ

