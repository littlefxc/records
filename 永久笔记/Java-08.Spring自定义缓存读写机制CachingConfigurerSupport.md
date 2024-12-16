---
title: Spring自定义缓存读写机制CachingConfigurerSupport
status: Done
Tags:
  - Java
  - Spring
---

---

## 概述

缓存在springboot项目中很常见，分布式项目中最常见的缓存机制就是通过redis缓存mybatis的查询数据，如下示例代码：

```java
@Configuration
@EnableCaching
public class RedisConfig extends CachingConfigurerSupport {

	@Bean
	public CacheManager redisCacheManager(RedisConnectionFactory connectionFactory) {
		RedisSerializationContext.SerializationPair serializationPair =
				RedisSerializationContext.SerializationPair.fromSerializer(getRedisSerializer());
		RedisCacheConfiguration redisCacheConfiguration = RedisCacheConfiguration.defaultCacheConfig()
				.entryTtl(Duration.ofSeconds(30))
				.serializeValuesWith(serializationPair);
		return RedisCacheManager
				.builder(RedisCacheWriter.nonLockingRedisCacheWriter(connectionFactory))
				.cacheDefaults(redisCacheConfiguration).build();
	}

  	private RedisSerializer<Object> getRedisSerializer(){
		return new GenericFastJsonRedisSerializer();
	}

}
```

```java
public interface UserMapper {

    @Cacheable(cacheNames = "User:Id",key="#p0")
    public User findById(@Param("id") Integer id);
}
```

上述代码的作用，是在调用`findById`方法时优先查询redis中的缓存数据。如果redis对应缓存不存在，则请求mysql查询数据，并将数据缓存到redis中，设置缓存的过期时间为30秒。

## 问题

示例代码简单明了，但是有两个问题：

1. 当redis连接出现异常时，调用findById方法会抛出异常影响到正常的业务流程；
2. 扩展性差，不能实现多层缓存，无法灵活切换多种缓存中间件（在@Cacheable中指定cacheManager只能实现一个方法固定使用一种缓存机制）；

## CacheErrorHandler

缓存仅仅是为了业务更快地查询而存在的，如果因为缓存操作失败导致正常的业务流程失败，有点得不偿失了。因此需要开发者自定义CacheErrorHandler处理缓存读写的异常。

```java
/**
 * 当缓存读写异常时,忽略异常
 */
public class IgnoreExceptionCacheErrorHandler implements CacheErrorHandler {

	private static final Logger log = LoggerFactory.getLogger(IgnoreExceptionCacheErrorHandler.class);

	@Override
	public void handleCacheGetError(RuntimeException exception, Cache cache, Object key) {
		log.error(exception.getMessage(), exception);
	}

	@Override
	public void handleCachePutError(RuntimeException exception, Cache cache, Object key, Object value) {
		log.error(exception.getMessage(), exception);
	}

	@Override
	public void handleCacheEvictError(RuntimeException exception, Cache cache, Object key) {
		log.error(exception.getMessage(), exception);
	}

	@Override
	public void handleCacheClearError(RuntimeException exception, Cache cache) {
		log.error(exception.getMessage(), exception);
	}
}
```

```java
@Configuration
@EnableCaching
public class RedisConfig extends CachingConfigurerSupport {

    /**
     * 添加自定义缓存异常处理
     * 当缓存读写异常时,忽略异常
     */
    @Override
    public CacheErrorHandler errorHandler() {
        return new IgnoreExceptionCacheErrorHandler();
    }
    
	@Bean
	public CacheManager redisCacheManager(RedisConnectionFactory connectionFactory) {
		RedisSerializationContext.SerializationPair serializationPair =
				RedisSerializationContext.SerializationPair.fromSerializer(getRedisSerializer());
		RedisCacheConfiguration redisCacheConfiguration = RedisCacheConfiguration.defaultCacheConfig()
				.entryTtl(Duration.ofSeconds(30))
				.serializeValuesWith(serializationPair);
		return RedisCacheManager
				.builder(RedisCacheWriter.nonLockingRedisCacheWriter(connectionFactory))
				.cacheDefaults(redisCacheConfiguration).build();
	}

  	private RedisSerializer<Object> getRedisSerializer(){
		return new GenericFastJsonRedisSerializer();
	}

}
```

缓存读写发生了异常，如果是读取redis异常，上述代码会导致调用 `findById` 读取缓存的值为空，从而继续从mysql读取数据，对业务没有影响。但是如果请求量很大就会出现缓存雪崩的问题，大量的查询请求发送到mysql导致mysql负载过大而阻塞甚至宕机，建议使用多层缓存兜底。

如果缓存写发生了异常，就可能导致mysql的数据和redis缓存的数据不一致的问题。为了解决该问题，需要继续扩展`CacheErrorHandler`的`handleCachePutError`和`handleCacheEvictError`方法，思路就是将redis写操作失败的key保存下来，通过重试任务删除这些key对应的redis缓存解决mysql数据与redis缓存数据不一致的问题。

## CacheResolver

开发者可以通过自定义`CacheResolver`实现动态选择`CacheManager`，如下通过代码实现对`findById`调用时使用多种缓存机制：优先从堆内存读取缓存，堆内存缓存不存在时再从redis读取缓存，redis缓存不存在时最后从mysql读取数据，并将读取到的数据依次写到redis和堆内存中。

```java
public class CustomCacheResolver implements CacheResolver, InitializingBean {

    @Nullable
    private List<CacheManager> cacheManagerList;

    public CustomCacheResolver(){
    }
    public CustomCacheResolver(List<CacheManager> cacheManagerList){
        this.cacheManagerList = cacheManagerList;
    }

    public void setCacheManagerList(@Nullable List<CacheManager> cacheManagerList) {
        this.cacheManagerList = cacheManagerList;
    }
    public List<CacheManager> getCacheManagerList() {
        return cacheManagerList;
    }

    @Override
    public void afterPropertiesSet()  {
        Assert.notNull(this.cacheManagerList, "CacheManager is required");
    }

    @Override
    public Collection<? extends Cache> resolveCaches(CacheOperationInvocationContext<?> context) {
        Collection<String> cacheNames = getCacheNames(context);
        if (cacheNames == null) {
            return Collections.emptyList();
        }
        Collection<Cache> result = new ArrayList<>();
        for(CacheManager cacheManager : getCacheManagerList()){
            for (String cacheName : cacheNames) {
                Cache cache = cacheManager.getCache(cacheName);
                if (cache == null) {
                    throw new IllegalArgumentException("Cannot find cache named '" +
                            cacheName + "' for " + context.getOperation());
                }
                result.add(cache);
            }
        }
        return result;
    }

    private Collection<String> getCacheNames(CacheOperationInvocationContext<?> context){
        return context.getOperation().getCacheNames();
    }
}
```

```java
@Configuration
@EnableCaching
public class RedisConfig extends CachingConfigurerSupport {

    @Autowired
    private RedisConnectionFactory connectionFactory;
    
    @Override
    public CacheResolver cacheResolver() {
        // 通过Guava实现的自定义堆内存缓存管理器
        CacheManager guavaCacheManager = new GuavaCacheManager();
        CacheManager redisCacheManager = redisCacheManager();
        List<CacheManager> list = new ArrayList<>();
        // 优先读取堆内存缓存
        list.add(concurrentMapCacheManager);
        // 堆内存缓存读取不到该key时再读取redis缓存
        list.add(redisCacheManager);
        return new CustomCacheResolver(list);
    }
    
    /**
     * 添加自定义缓存异常处理
     * 当缓存读写异常时,忽略异常
     */
    @Override
    public CacheErrorHandler errorHandler() {
        return new IgnoreExceptionCacheErrorHandler();
    }
    
	@Bean
	public CacheManager redisCacheManager() {
		RedisSerializationContext.SerializationPair serializationPair =
				RedisSerializationContext.SerializationPair.fromSerializer(getRedisSerializer());
		RedisCacheConfiguration redisCacheConfiguration = RedisCacheConfiguration.defaultCacheConfig()
				.entryTtl(Duration.ofSeconds(30))
				.serializeValuesWith(serializationPair);
		return RedisCacheManager
				.builder(RedisCacheWriter.nonLockingRedisCacheWriter(connectionFactory))
				.cacheDefaults(redisCacheConfiguration).build();
	}

  	private RedisSerializer<Object> getRedisSerializer(){
		return new GenericFastJsonRedisSerializer();
	}

}
```

通过自定义`CacheResolver`开发者可以实现更多的自定义功能，例如热点缓存自动升降级的场景：

项目大多数情况下只使用redis做缓存，当某些场景下个别数据成为了热数据，通过例如storm实时统计出热数据后，项目将这些热数据缓存到堆内存，缓解网络和redis的负载压力。

这种场景完全可以通过自定义`CacheResolver`来实现，storm实时统计出热数据，自定义的`CacheResolver`在调用`resolveCaches`选择`CacheManager`前，先判断此次读写的缓存key是否是热数据。如果是热数据则使用堆内存的`CacheManager`，否则使用`redis的CacheManager`。