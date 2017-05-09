# 26. JMS (Java 消息服务)
## 26.1 介绍

Spring提供了一个JMS的集成框架，简化了JMS API的使用，就像Spring对JDBC API的集成一样。

JMS可以大致分为两部分功能，即消息的生产与消费。```JmsTemplate```类用于消息生产和消息的同步接收。 对于类似于Java EE的消息驱动对象方式的异步接收，Spring提供了大量用于创建消息驱动对象（MDPs）的消息监听器。Spring还提供了声明式的方式来创建消息监听器。

包```org.springframework.jms.core```提供了使用JMS的核心功能。它包含了JMS模板类，用来处理资源的创建与释放，从而简化了JMS的使用，就像```JdbcTemplate```对JDBC做的一样。Spring模板类通用的设计原则对执行常规操作提供了帮助方法，对于更复杂的使用，将处理任务的本质委托给用户自己实现的回调接口上。JMS模板遵循了同样的设计理念。这些类对消息的发送与同步消费提供了各种便利的方法，同时JMS的会话与消息生产者暴露给了用户。

包```org.springframework.jms.support```提供可JMSException的转换功能。该转换将检查的```JMSException```层次结构转换为未检查异常的镜像层次结构。 如果检查的```javax.jms.JMSException```有任何提供者特定的子类，则该异常将被包装在未经检查的```UncategorizedJmsException```中。

包```org.springframework.jms.support.converter```提供了一个```MessageConverter```抽象，用于在JMS消息与JAVA对象之间的互相转换。

包```org.springframework.jms.support.destination```管理JMS目的地的各种策略，例如为JNDI中存储的目的地提供服务定位器。

包```org.springframework.jms.annotation```提供了使用```@JmsListener```作为注解驱动的监听端点的必要基础设施。

包```org.springframework.jms.config```提供了jms命名空间的解析器实现以及配置监听器和创建监听端点的java配置支持。

最后，包```org.springframework.jms.connection```提供适用于独立应用程序的```ConnectionFactory```实现。 它还包含Spring对JMS的```PlatformTransactionManager```的实现（即JmsTransactionManager）。 这允许将JMS作为事务资源无缝集成到Spring的事务管理机制中。

## 26.2 Spring JMS的使用
### 26.2.1 JmsTemplate
```JmsTemplate```类是JMS核心包中的中心类。 它简化了JMS的使用，因为在发送或同步接收消息时它帮我们处理了资源的创建和释放。

使用JmsTemplate的代码只需要实现回调接口，在上层给接口一个明确的定义。 MessageCreator回调接口创建一个给定由JmsTemplate中调用代码提供的会话的消息。。 为了允许JMS API的更复杂使用，SessionCallback回调为用户提供了JMS会话，并且ProducerCallback回调公开了一个Session所对应的一个MessageProducer。

JMS API公开了两种类型的发送方法，一种采用交付模式，优先级和生存时间（即服务质量（QOS））参数，另一种则不需要使用默认值的QOS参数。由于JmsTemplate中有很多发送方法，因此QOS参数的设置已经被暴露为bean属性，以避免发送方法的重复。 类似地，同步接收调用的超时值是使用属性setReceiveTimeout设置的。

一些JMS提供程序允许通过配置ConnectionFactory来管理默认QOS值。 这样做的结果是调用MessageProducer的send方法```send（Destination destination，Message message）```将使用与JMS规范中指定的不同的QOS默认值。 为了提供QOS值的一致管理，因此JmsTemplate必须通过将布尔属性isExplicitQosEnabled设置为true来专门启用自己的QOS值。

为了方便起见，JmsTemplate还公开了一个基本的请求回复操作，允许发送消息并等待临时队列的回复，该队列是作为操作的一部分而被创建的。

> 配置JmsTemplate类的实例是线程安全的。这很重要，因为这意味着你可以配置JmsTemplate的单例，然后将该共享引用安全地注入多个协作者。 要清楚，JmsTemplate是有状态的，因为它保持对ConnectionFactory的引用，但是这个状态不是会话状态。

从Spring Framework 4.1开始，JmsMessagingTemplate构建在JmsTemplate之上，并提供与消息传递抽象（即org.springframework.messaging.Message）的集成。 这允许您创建以通用方式发送的消息。

### 26.2.2 Connections
JmsTemplate需要引用一个ConnectionFactory。 ConnectionFactory是JMS规范的一部分，作为使用JMS的切入点。 它被客户端应用程序用作工厂来创建与JMS提供者的连接，并封装各种配置参数，其中许多配置参数是供应商特定的，如SSL配置选项。

在EJB内部使用JMS时，供应商提供JMS接口的实现，以便他们可以参与声明式事务管理并执行连接和会话池。 为了使用此实现，Java EE容器通常要求您将JMS连接工厂声明为EJB或Servlet部署描述符中的资源引用。 为了确保在EJB内使用JmsTemplate这些特性，客户端应用程序应该确保它引用了ConnectionFactory的托管实现。

#### 缓存消息资源
标准的API涉及创建许多中间对象。要发送消息，将执行以下“API”步骤

	ConnectionFactory->Connection->Session->MessageProducer->send

在ConnectionFactory和Send操作之间，有三个被创建和销毁的中间对象。 为了优化资源使用并提高性能，提供了两个ConnectionFactory的实现。

#### SingleConnectionFactory

Spring提供了一个ConnectionFactory接口的实现类SingleConnectionFactory，它将在所有createConnection（）调用上返回相同的Connection，并忽略对close（）的调用。 这对于测试和独立环境非常有用，因此可以将多个使用相同连接的JmsTemplate调用用于任何数量的事务。 SingleConnectionFactory引用了通常来自JNDI的标准ConnectionFactory。

#### CachingConnectionFactory
CachingConnectionFactory扩展了SingleConnectionFactory的功能，它添加了Sessions，MessageProducers和MessageConsumers的缓存。 初始缓存大小设置为1，使用属性sessionCacheSize增加缓存会话的数量。 请注意，实际缓存会话的数量将超过该数量，因为会话将基于其确认模式进行缓存，因此当sessionCacheSize设置为1时，每个确认模式存储一个，最多到达4个缓存的会话实例。 MessageProducers和MessageConsumers被缓存在他们自己的会话中，并且还考虑到缓存时生产者和消费者的独特属性。 MessageProducer根据目的地进行缓存。 MessageConsumers基于由目的地，选择器，noLocal传递标志和持久订阅名称（如果创建持久消费者）组成的键缓存。

###26.2.3 Destination Management
目的地，如ConnectionFactories，是JMS管理的对象，可以在JNDI中存储和检索。在配置Spring应用程序上下文时，可以使用JNDI工厂类```JndiObjectFactoryBean / <jee：jndi-lookup>```对你的JMS目标的引用对象执行依赖注入。但是，如果应用程序中有大量的目的地，或者如果JMS提供程序具有独特的高级目标管理功能，那么这种策略通常显得笨重。这种高级目的地管理的示例将是创建动态目的地或支持目的地的分级命名空间。 JmsTemplate将目标名称的解析委派给JMS目标对象，以实现接口DestinationResolver。 DynamicDestinationResolver是JmsTemplate使用的默认实现，适用于解析动态目标。还提供了JndiDestinationResolver，作为JNDI中包含的目的地的服务定位器，并且可选地还原在DynamicDestinationResolver中包含的行为。

通常，在JMS应用程序中使用的目标只在运行时已知，因此在部署应用程序时不能进行管理创建。这通常是因为根据众所周知的命名约定，在运行时创建目的地的交互系统组件之间存在共享的应用程序逻辑。即使动态目的地的创建不是JMS规范的一部分，大多数供应商都提供了这种功能。使用用户定义的名称创建动态目标，它们将自己与临时目标区分开来，并且通常不会在JNDI中注册。用于创建动态目的地的API因提供者而异，因为与目的地关联的属性是供应商特定的。然而，供应商有时做出的简单实现选择是忽略JMS规范中的警告，并使用TopicSession方法createTopic（String topicName）或QueueSession方法createQueue（String queueName）创建具有默认目标属性的新目标。根据供应商的实现，DynamicDestinationResolver可能还会创建一个物理目标，而不是只解析一个。

布尔属性pubSubDomain用于配置JmsTemplate正在使用的JMS的域名。 默认情况下，此属性的值为false，表示将使用点对点域，队列。 JmsTemplate使用的此属性通过DestinationResolver接口的实现来确定动态目标解析的行为。

您还可以通过defaultDestination属性用默认目标位置来配置JmsTemplate。 默认目的地将用于不涉及特定目的地的发送和接收操作。

###26.2.4 消息监听器
EJB世界中最常用的JMS消息之一是驱动消息驱动的beans（MDB）。 Spring提供了一种解决方案，以不将用户绑定到EJB容器的方式来创建消息驱动的POJO（MDP）。 （参见第26.4.2节“异步接收 - 消息驱动的POJO”，详细介绍了Spring的MDP支持）。从Spring Framework 4.1开始，端点方法可以使用@JmsListener来简单的进行注解，参见第26.6节“注释驱动的侦听器端点 “ 更多细节。

消息监听器用于从JMS消息队列接收消息，并驱动注入到其中的MessageListener。 监听器容器负责消息接收和传递到进行处理的监听器的所有线程。 消息监听器容器是MDP和消息传递提供者之间的中介，并且负责注册接收消息，参与事务，资源获取和释放，异常转换等。 这允许你作为应用程序开发人员来编写接收（有可能响应）消息相关联的（可能复杂的）业务逻辑，并将样板JMS基础架构的关注委托给框架。

有两个Spring包装的标准JMS消息监听器容器，每个都具有其专门的功能集。

####SimpleMessageListenerContainer
这个消息监听器容器是两个标准风格中更简单的。 它在启动时创建一个固定数量的JMS会话和消费者，使用标准的JMS MessageConsumer.setMessageListener（）方法注册该监听器，并将它放在JMS提供者上以执行监听器回调。 此变体不允许动态适应运行时需求或参与外部管理的事务。 兼容性方面，它保持非常接近独立JMS规范的精神 - 但通常与Java EE的JMS限制不兼容。

> 虽然SimpleMessageListenerContainer不允许参与外部管理的事务，但它确实支持原生JMS事务：只需将“sessionTransacted”标志切换为“true”，或者在命名空间中将'acknowledge'属性设置为“transacted”：从你的监听器抛出的异常将导致回滚，然后消息被重新传递。 或者，考虑使用'CLIENT_ACKNOWLEDGE'模式，在异常的情况下提供重新传递，但不使用事务会话，因此在事务协议中不包括任何其他会话操作（例如发送响应消息）。

####DefaultMessageListenerContainer
这个消息监听器容器是大多数情况下使用的容器。 与SimpleMessageListenerContainer相反，此容器变体允许动态调整运行时需求，并能够参与外部管理的事务。 当使用JtaTransactionManager配置时，每个接收到的消息都会用XA事务注册; 所以处理可以利用XA事务语义。该监听器容器在JMS提供者的低要求，高级功能（如参与外部管理的事务）以及与Java EE环境的兼容性之间取得了良好的平衡。

可以定制容器的缓存级别。 请注意，当不启用缓存时，将为每个消息接收创建一个新的连接和一个新的会话。 将此与以高负载的非持久订阅组合可能会导致消息丢失。 在这种情况下，请确保使用适当的缓存级别。

当代理商停机时，此容器也具有可恢复的功能。 默认情况下，一个简单的BackOff实现每5秒重试一次。 可以为更细粒度的恢复选项指定自定义BackOff实现，请参见ExponentialBackOff示例。

> 与其同级SimpleMessageListenerContainer一样，DefaultMessageListenerContainer支持原生JMS事务，并允许自定义确认模式。 如果可行的话，强烈建议您使用外部管理的事务：即，如果您可以在JVM挂掉的情况下偶尔重复发送消息。 业务逻辑中的自定义重复消息检测步骤可能涵盖这些情况，例如 以业务实体的形式存在检查或协议表检查。 任何这样的安排将比其他方式显着更有效：用XA事务（通过使用JtaTransactionManager配置DefaultMessageListenerContainer）来包裹整个过程，覆盖JMS消息的接收以及消息监听器中业务逻辑的执行 （包括数据库操作等）。













