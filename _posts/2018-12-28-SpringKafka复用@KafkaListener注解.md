---
layout:     post
title:      SpringKafka复用@KafkaListener注解
subtitle:   SpringKafka复用@KafkaListener注解
date:       2018-12-28
author:     ica10888
catalog: true
tags:
    - java
    - spring
    - kafka
---

# SpringKafka复用@KafkaListener注解

在使用@KafkaListener的时候，有时候一个Spring工程需要复用@KafkaListener注解。SpringKafka提供了一种主从的方式在同一个Spring工程下复用@KafkaListener的实现。

### Config

主类的 Bean 类名字 consumerConfigs、 consumerFactory、kafkaListenerContainerFactory不能做改动。

Primary KafkaFactory Config

``` java
    @Bean
    public KafkaListenerContainerFactory<?> batchFactoryForCmd() {
        ConcurrentKafkaListenerContainerFactory<Integer, String> factory =
                new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.getContainerProperties().setAckMode(AbstractMessageListenerContainer.AckMode.MANUAL_IMMEDIATE);
        factory.setBatchListener(true);
        return factory;
    }


    @Bean
    @Primary //重要！！！指定该ContainerFactory为主要的容器工厂，kafka消费者默认关联该容器
    KafkaListenerContainerFactory<ConcurrentMessageListenerContainer<Integer, String>>
    kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<Integer, String> factory =
                new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.setConcurrency(3);
        factory.getContainerProperties().setPollTimeout(3000);
        return factory;
    }

    @Bean
    public ConsumerFactory<Integer, String> consumerFactory() {
        return new DefaultKafkaConsumerFactory<>(consumerConfigs());
    }

    @Bean
    public Map<String, Object> consumerConfigs() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.GROUP_ID_CONFIG, getGroup());
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, getServers());
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        props.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, 100);
        props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 15000);
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        return props;
    }
```



Primary必须有一个，其他的可以有多个实现。

由于Bean类都在同一个的命名空间下，且都是单例的，各个 Bean 的方法名要不同，不然 Spring  的 Bean 类工厂无法初始化。比如 Primary 的 ConsumerFactory名称是 consumerFactory。另外一个实现的名称就要不同，写作 consumerFactoryForConn 用以区别。

Other KafkaFactory Config

``` java
    @Bean
    public KafkaListenerContainerFactory<?> batchFactoryForConn() {
        ConcurrentKafkaListenerContainerFactory<Integer, String> factory =
                new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactoryForConn());
        factory.getContainerProperties().setAckMode(AbstractMessageListenerContainer.AckMode.MANUAL_IMMEDIATE);
        factory.setBatchListener(true);
        return factory;
    }


    @Bean
    KafkaListenerContainerFactory<ConcurrentMessageListenerContainer<Integer, String>>
    kafkaListenerContainerFactoryForConn() {
        ConcurrentKafkaListenerContainerFactory<Integer, String> factory =
                new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactoryForConn());
        factory.setConcurrency(3);
        factory.getContainerProperties().setPollTimeout(3000);
        return factory;
    }

    @Bean
    public ConsumerFactory<Integer, String> consumerFactoryForConn() {
        return new DefaultKafkaConsumerFactory<>(ConsumerConfigsForConn());
    }

    @Bean
    public Map<String, Object> ConsumerConfigsForConn() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.GROUP_ID_CONFIG,getGroup());
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, getServers());
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        props.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, 100);
        props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 15000);
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        return props;
    }
```

### @KafkaListener

配置不同的 containerFactory 就可以实现对不同的 topics 同时进行消费。containerFactory 就是之前 KafkaListenerContainerFactory Bean类 的方法名。id 配置不同的名称来区别。

``` java
    @KafkaListener(id = "cmdKafka", topics = "${kafka.cmd.topic}", containerFactory = "batchFactoryForCmd")
    public void listen(List<String> results,Acknowledgment ack) {
        commandProcessTask.publishCommandMessage(results);
        ack.acknowledge(); //手动提交commit
    }
    
    
    @KafkaListener(id = "connKafka", topics = "${kafka.conn.topic}", containerFactory = "batchFactoryForConn")
    public void listen(List<String> results,Acknowledgment ack) {
        dataProcessTask.publishDataMessage(results);
        ack.acknowledge(); //手动提交commit
    }
```

### BeanFacotry

##### BeanFactory和FactoryBean的区别

BeanFactory是接口 , 一般使用的高级容器  ApplicationContext 接口继承了这个接口（其实是继承了多个接口，其中ListableBeanFactory, HierarchicalBeanFactory 都继承了BeanFactory），所有Bean 由它来管理。

FactoryBean在IOC容器的基础上给Bean的实现加上了一个简单工厂模式和装饰模式，让我们可以获取Bean，是通过反射机制实现的，很多Bean类都实现了FactoryBean<T>。

这样使用不同的id就可以获取到各自的Bean实例。

``` java
@Autowired
Private ApplicationContext context;
```

``` java
KafkaListenerContainerFactory<?> batchFactoryForCmd = (KafkaListenerContainerFactory<?>) context.getBean("batchFactoryForCmd");
KafkaListenerContainerFactory<ConcurrentMessageListenerContainer<Integer, String>> kafkaListenerContainerFactory =(KafkaListenerContainerFactory<ConcurrentMessageListenerContainer<Integer, String>>) context.getBean("kafkaListenerContainerFactory");
ConsumerFactory<Integer, String> consumerFactory = (ConsumerFactory<Integer, String>) context.getBean("consumerFactory");
Map<String, Object> consumerConfigs = (Map<String, Object>)context.getBean("consumerConfigs");


KafkaListenerContainerFactory<?> batchFactoryForConn = (KafkaListenerContainerFactory<?>) context.getBean("batchFactoryForConn");
KafkaListenerContainerFactory<ConcurrentMessageListenerContainer<Integer, String>> kafkaListenerContainerFactoryForConn =(KafkaListenerContainerFactory<ConcurrentMessageListenerContainer<Integer, String>>) context.getBean("kafkaListenerContainerFactoryForConn");
ConsumerFactory<Integer, String> consumerFactoryForConn = (ConsumerFactory<Integer, String>)context.getBean("consumerFactoryForConn");
Map<String, Object> ConsumerConfigsForConn = (Map<String, Object>)context.getBean("ConsumerConfigsForConn");

```

