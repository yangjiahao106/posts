
---
title: docker 部署kafka
date: 2022-03-29 20:20:01
tags:
---

# docker 部署 kafka 

  镜像使用 bitnami/zookeeper
  注意  KAFKA_CFG_LISTENERS 为在容器中绑定的地址和端口
       KAFKA_CFG_ADVERTISED_LISTENERS 为映射到容器外的地址和端口。

```yaml

version: "3"
services:
  zookeeper:
    image: 'bitnami/zookeeper:latest'
    ports:
      - '2181:2181'
    environment:
      #- 匿名登录--必须开启
      - ALLOW_ANONYMOUS_LOGIN=yes
    volumes:
      - ./zookeeper:/bitnami/zookeeper
    networks:
      kafka_net:
        aliases:
          - zookeeper


  kafka:
    image: 'bitnami/kafka:latest'
    ports:
      - '9092:9092'
    environment:
      - KAFKA_BROKER_ID=1
      - KAFKA_CFG_LISTENERS=PLAINTEXT://0.0.0.0:9092 # 实际绑定地址IP端口
      - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://IP:9092 # 暴露出去的宿主机的地址和端口
      - KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper:2181
      - ALLOW_PLAINTEXT_LISTENER=yes
      #- 全局消息过期时间 6 小时(测试时可以设置短一点)
      - KAFKA_CFG_LOG_RETENTION_HOURS=6
    volumes:
      - ./kafka:/bitnami/kafka
    depends_on:
      - zookeeper
    networks:
      kafka_net:
        aliases:
          - kafka


  kafka2:
    image: 'bitnami/kafka:latest'
    ports:
      - '9093:9093'
    environment:
      - KAFKA_BROKER_ID=2
      - KAFKA_CFG_LISTENERS=PLAINTEXT://0.0.0.0:9093
      - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://IP:9093
      - KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper:2181
      - ALLOW_PLAINTEXT_LISTENER=yes
      #- 全局消息过期时间 6 小时(测试时可以设置短一点)
      - KAFKA_CFG_LOG_RETENTION_HOURS=6
    volumes:
      - ./kafka2:/bitnami/kafka
    depends_on:
      - zookeeper
    networks:
      kafka_net:
        aliases:
          - kafka2


  kafka3:
    image: 'bitnami/kafka:latest'
    ports:
      - '9094:9094'
    environment:
      - KAFKA_BROKER_ID=3
      - KAFKA_CFG_LISTENERS=PLAINTEXT://0.0.0.0:9094
      - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://IP:9094
      - KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper:2181
      - ALLOW_PLAINTEXT_LISTENER=yes
      #- 全局消息过期时间 6 小时(测试时可以设置短一点)
      - KAFKA_CFG_LOG_RETENTION_HOURS=6
    volumes:
      - ./kafka3:/bitnami/kafka
    depends_on:
      - zookeeper
    networks:
      kafka_net:
        aliases:
          - kafka3



  kafka_manager:
    image: 'hlebalbau/kafka-manager:latest'
    ports:
      - "9002:9000"
    environment:
      ZK_HOSTS: "zookeeper:2181"
      APPLICATION_SECRET: letmein
    depends_on:
      - zookeeper
      - kafka
      - kafka2
      - kafka3
    networks:
      kafka_net:
        aliases:
          - kafka_manager

networks:
  kafka_net:

  ```