---
title: 【Spring】Redis的两个典型应用场景
published: true
date: '12-03-2016 00:00'
visible: true
summary:
    enabled: '1'
    format: short
taxonomy:
    tag:
        - 原创
        - boot
---

#### Redis简介

[Redis](http://redis.io/)是目前业界使用最广泛的内存数据存储。相比memcached，Redis支持更丰富的数据结构，例如hashes, lists, sets等，同时支持数据持久化。除此之外，Redis还提供一些类数据库的特性，比如事务，HA，主从库。可以说Redis兼具了缓存系统和数据库的一些特性，因此有着丰富的应用场景。本文介绍Redis在Spring Boot中两个典型的应用场景。

#### 场景1：数据缓存

第一个应用场景是数据缓存，最典型的当属缓存数据库查询结果。对于高频读低频写的数据，使用缓存可以第一，加速读取过程，第二，降低数据库压力。通过引入spring-boot-starter-redis依赖和注册RedisCacheManager，Redis可以无缝的集成进Spring的缓存系统，自动绑定@Cacheable, @CacheEvict等缓存注解。

引入依赖：

	<dependency>
    	<groupId>org.springframework.boot</groupId>
	    <artifactId>spring-boot-starter-redis</artifactId>
    </dependency>

Redis配置（application.properties）:

	# REDIS (RedisProperties)
    spring.redis.host=localhost
    spring.redis.password=
    spring.redis.database=0

注册RedisCacheManager：

	@Configuration
	@EnableCaching
    public class CacheConfig {

        @Autowired
        private JedisConnectionFactory jedisConnectionFactory;

        @Bean
        public RedisCacheManager cacheManager() {
            RedisCacheManager redisCacheManager = new RedisCacheManager(redisTemplate());
            return redisCacheManager;
        }

        @Bean
        public RedisTemplate<Object, Object> redisTemplate() {
            RedisTemplate<Object, Object> redisTemplate = new RedisTemplate<Object, Object>();
            redisTemplate.setConnectionFactory(jedisConnectionFactory);
            // 使用String格式序列化缓存键
            redisTemplate.setKeySerializer(new StringRedisSerializer());
            return redisTemplate;
        }

    }
    
@Cachable, @CacheEvict使用，Redis中的存储结构可参见场景2中的配图：

	@Cacheable(value="signonCache", key="'petstore:signon:'+#username", unless="#result==null")
    public Signon findByName(String username) {
        return dao.fetchOneByUsername(username);
    }
    
    @CacheEvict(value="signonCache", key="'petstore:signon:'+#user.username")
    public void update(Signon user) {
        dao.update(user);
    }

- @Cacheable: 插入缓存
	- value: 缓存名称
	- key: 缓存键，一般包含被缓存对象的主键，支持Spring EL表达式
	- unless: 只有当查询结果不为空时，才放入缓存
- @CacheEvict: 失效缓存

> Tip: Spring Redis默认使用JDK进行序列化和反序列化，因此被缓存对象需要实现java.io.Serializable接口，否则缓存出错。

> Tip: 当被缓存对象发生改变时，可以选择更新缓存或者失效缓存，但一般而言，后者优于前者，因为执行速度更快。

> Watchout! 在同一个Class内部调用带有缓存注解的方法，缓存并不会生效。

#### 场景2：共享Session

共享Session是第二个典型应用场景，这是利用了Redis的堆外内存特性。要保证分布式应用的可伸缩性，带状态的Session对象是绕不过去的一道坎。一种方式是将Session持久化到数据库中，缺点是读写成本太高。另一种方式是去Session化，比如Play直接将Session存到客户端的Cookie中，缺点是存储信息的大小受限。将Session缓存到Redis中，既保证了可伸缩性，同时又避免了前面两者的限制。

引入依赖：

	<dependency>
        <groupId>org.springframework.session</groupId>
        <artifactId>spring-session-data-redis</artifactId>
    </dependency>

Session配置：

    @Configuration
    @EnableRedisHttpSession(maxInactiveIntervalInSeconds = 86400)
    public class SessionConfig {
    }

- maxInactiveIntervalInSeconds: 设置Session失效时间，使用Redis Session之后，原Boot的server.session.timeout属性不再生效

Redis中的session对象：

![](QQ20160313-0.png)

#### 小结

上面结合示例代码介绍了数据缓存，共享Session两个Redis的典型应用场景，除此之外，还有分布式锁，全局计数器等高级应用场景，以后在其他文章中再详细介绍。

#### 参考

- [Spring Data Redis](http://docs.spring.io/spring-data/redis/docs/current/reference/html/)
- [Spring Cache抽象详解](http://jinnianshilongnian.iteye.com/blog/2001040)
- [HttpSession with Redis](http://docs.spring.io/spring-session/docs/current/reference/html5/#httpsession-redis)
