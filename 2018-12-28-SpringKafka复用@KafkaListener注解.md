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

