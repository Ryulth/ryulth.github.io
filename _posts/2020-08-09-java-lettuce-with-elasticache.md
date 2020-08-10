---
layout: post
title:  "[Spring]Spring Lettuce with ElastiCache Redis Read Replica config"
subtitle: "Spring 에서 Redis Stream 을 사용하는 방법"
categories: devnote
tags: spring
---

> Spring에서 Lettuce 를 사용해 ElastiCache Redis Read Replica 설정하기.
## Lettuce with ElastiCache 

Java에서 사용하는 대표적인 Redis Client들은 여러가지가 있다. [Jedis](https://github.com/xetorthio/jedis), [Lettuce](https://github.com/lettuce-io/lettuce-core), [Redisson](https://github.com/redisson/redisson) 등등.

이번에 사용하는 스펙은 Kotlin 과 [Spring-reactive-redis](https://spring.io/guides/gs/spring-data-reactive-redis/) 를 사용하려고 Lettuce 를 선택했다.

### Redis 설정

```kotlin
@Configuration
class RedisConfig {
    @Bean
    fun RedisConnectionFactory(
        @Value("\${redis.host}") hostName: String,
        @Value("\${redis.port:6379}") port: Int
    ): LettuceConnectionFactory {
        val redisConfiguration = RedisStandaloneConfiguration(hostName, port)
        val clientConfig = LettuceClientConfiguration.builder()
                .build()

        return LettuceConnectionFactory(redisConfiguration, clientConfig)
    }
}
```

기본적으로 Redis 설정을 해본다. 

### Write to Master, Read from Replica

서비스를 운영하던 도중 Read 와 Write 를 분리하는게 좋은 상황이 닥쳤다. ElastiCache 는 기본적으로 복제본을 둘 수 있다. 센티널한 방식은 아니지만 자체적으로 failover 를 시킨다. 그래서 관련 자료들을 찾아보기 시작했다. [공식 문서](https://docs.spring.io/spring-data/redis/docs/current/reference/html/#redis:write-to-master-read-from-replica)에 기재된 방법을 보니 매우 간단해 보여서 놀랐다. 

```kotlin
@Configuration
class WriteToMasterReadFromReplicaConfiguration {

  @Bean
  public LettuceConnectionFactory redisConnectionFactory() {

    LettuceClientConfiguration clientConfig = LettuceClientConfiguration.builder()
      .readFrom(REPLICA_PREFERRED)
      .build();

    RedisStandaloneConfiguration serverConfig = new RedisStandaloneConfiguration("server", 6379);

    return new LettuceConnectionFactory(serverConfig, clientConfig);
  }
}
```

뭐 대충 ReadFrom.REPLICA_PREFERRED 옵션만 넣어주면 된단다 하지만 너무 간단해서 위화감을 느꼇는데 밑에 문구를 보고 위화감이 현실로 다가왔다. ![스크린샷 2020-08-09 오후 5.26.50](https://user-images.githubusercontent.com/32893340/89728148-f9757d00-da65-11ea-909b-53940972a702.png)

대충 해석하자면 Redis ``INFO`` 커맨드로 Replica 정보들을 받을 수 있는데 AWS 같은 서비스들은 ``INFO`` 로 접속 불가능한 주소를 주기 때문에 ``RedisStaticMasterReplicaConfiguration`` 를 사용해라 라는 것이다. 이름부터 Static 이라니 설마 ElastiCache 노드 주소를 각각 넣어주라는 건가 라고 생각했다. 

### Write to Master, Read from Replica With ElastiCache

ElastiCache를 생성하면 여러가지 정보를 제공 받을 수 있다. Primary 엔트포인트는 Read/Wrtie 가 가능한 마스터 엔드포인트를 가르킨다. Reader 엔드포인트는 내부적으로 Replica 노드들을 바라 보고 있는 라우팅된(?) 주소로 볼 수 있다.

![스크린샷 2020-08-09 오후 5 39 50](https://user-images.githubusercontent.com/32893340/89728305-57569480-da67-11ea-8414-9ed2b33953c7.png)

이제 우리는 Primary 엔드포인트 와 Reader 엔드포인트를 사용해서 설정해주면 된다. 그 후 Redis-cli MONITOR 명령어를 통해 트래픽이 들어오는지 확인해 주면 된다.

```kotlin
@Configuration
class RedisConfig {
    @Bean
    fun RedisConnectionFactoryForAWS(
        @Value("\${redis.primary.host}") masterHostName: String,
        @Value("\${redis.primary.port:6379}") masterPort: Int,
        @Value("\${redis.reader.host}") replicaHostName: String,
        @Value("\${redis.reader.port:6379}") replicaPort: Int
    ): LettuceConnectionFactory {
        val elastiCache = RedisStaticMasterReplicaConfiguration(masterHostName, masterPort)
        elastiCache.addNode(replicaHostName, replicaPort)

        val clientConfig = LettuceClientConfiguration.builder()
                .readFrom(ReadFrom.REPLICA_PREFERRED)
                .build()

        return LettuceConnectionFactory(elastiCache, clientConfig)
    }
}
```

### RedisStaticMasterReplicaConfiguration 를 사용할 수 밖에 없는 이유

Redis 에 접속가능한 EC2 를 만들어서 Primary 엔드포인트에 접속한후 ``INFO`` 커맨드를 사용해보자 

```
ubuntu@ip-172-31-14-81:~$ redis-cli -h lettuce-test.cdlgsu.ng.0001.apn2.cache.amazonaws.com
lettuce-test.cdlgsu.ng.0001.apn2.cache.amazonaws.com:6379> INFO
...
# Replication
role:master
connected_slaves:1
slave0:ip=10.27.0.148,port=6379,state=online,offset=1366573192,lag=0
...
```

이제 저 ip 를 다시 EC2 에서 접속해보자. 바로 끊어진다. 제공하는 주소는 Redis 노드들 간의 내부 통신에 사용하는 ip 주소다. 따라서 ElastiCache 는 INFO Replication 명령어로 Replica 엔드포인트를 가져올 수 없기 때문에 따로 지정해주어야 하는 것이다.

 Reader 엔드포인트에 붙어서 ``INFO`` 명령어를 쳐보자

```
ubuntu@ip-172-31-14-81:~$ redis-cli -h lettuce-test-ro.cdlgsu.ng.0001.apn2.cache.amazonaws.com
lettuce-test-ro.cdlgsu.ng.0001.apn2.cache.amazonaws.com:6379> INFO
# Replication
role:slave
master_host:lettuce-test.cdlgsu.ng.0001.apn2.cache.amazonaws.com
master_port:6379
master_link_status:up
...
```

...? Replica 엔드포인트에서는 쓰기 가능한 Primary 엔드포인트를 가져 올 수 있다. 

### Lettuce Failover with ElastiCache

결과부터 말하면 된다. Lettuce faileover 방식은 ConnectionWatchdog 가 Redis Node가 죽는걸 감지하고 기존 설정된 host 정보로  DNS resolver로 AutoReconnect 를 통해 다시 커넥션을 맺는다. 

ElastiCache 에서 failover 가 나면 기존 Primary 엔드포인트 주소가 변경되는 것이 아닌 내부에서 Replica 노드가 Primary 노드로 승격이 되면서 Primary 엔드포인트에 붙게되 Lettuce 는 AutoReconnect 과정을 통해 승격된 노드로 다시 붙게된다.

라이브러리 마다 Failover 방식이 다름으로 ElatiCache 를 사용할 때 주의해야 할 것같다. 

[Sample code](https://github.com/Ryulth/spring-elasticache-config)

끝

### Reference

- https://docs.spring.io/spring-data/redis/docs/current/reference/html/#redis:write-to-master-read-from-replica
- https://stackoverflow.com/questions/41048313/redis-client-lettuce-master-slave-configuration-for-aws-elasticache
- https://github.com/lettuce-io/lettuce-core/issues/1325