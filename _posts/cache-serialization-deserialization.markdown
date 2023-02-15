---
layout: post
title:  "Cache Serialization and Deserialization with Redis and Spring Boot"
date:   2022-02-25 16:00:00
published: false
---
### Overview

### Versioning



### Array
Don't use array with primitives like int/long or Integer/Long/String. If you save an array, the cache layer could throw an Exception due to the serialization and deserialization with Jackson/Redis. We strongly suggest using a POJO wrapper instead of raw array in all cases.

### Serialization - Basic Rules
Often, the cache layer uses Jackson to serialize/deserialize your objects with Redis, so it follows Jackson's rule.

#### Basic Rule 1: you need a default constructor for Jackson to deserialize and build your object at runtime. If you don't have a default constructor, you need to add JsonCreator annotation to one of your constructors, otherwise you will encounter InvalidDefinitionException.

e.g. you have an object with 2 fields: f1 and f2,  and you have a constructor MyObj(int f1, int f2), you must add annotations like this
```java
@JsonCreator MyObj(@JsonProperty("f1") int f1, @JsonProperty("f2") int f2)
```

#### Basic Rule 2: you need JsonProperty or setter/getter to let Jackson know what fields you want to deserialize.

e,g. you have an object with 4 fields: f1, f2, f3 and f4. We demonstrate 4 different ways to let Jackson deserialize your object below.

```java
class MyObj { 
    @JsonProperty("f1") int f1; // f1 can be set via reflection at runtime 
    int f2; 
    int f3; 
    int f4; 

    @JsonCreator(@JsonProperty("f2") int f2) { ... } // f2 can be set via the constructor 
    void setF3(int v) { ... } // f3 has a setter, Jackson will recognize f3 as a field and set its value via this setter 
    void getF4() { ... } // f4 has a getter but no setter, Jackson will recognize f4 as a field and set its value via reflection 
}
```

#### Serialization - Compatibility
Should the cache key contain the service version or entity version? like this: <tenant id>:<entity version>:<your key> or <tenant id>:<your key>

Each time you deploy a new version of your service, you probably have changed entity, the key can be different than the old version, so your new service will have brand new cache. This helps you ensure the data structure of the old and new service are isolated in cache. But this also causes other issues:

1) your new service instance starts with cold cache, if we don't take traffic carefully to the new service instance in a canary release, your service could be slowed down dramatically, and affect the user experience or other services like a rolling snow ball.

2) sometimes your new service does not change the data structure and the existing cached data in Redis can be reused, in that case it's a waste.

3) the Redis will take twice memory as needed during deployment, be very careful about your capacity planning.

To avoid those problems, we remove the version and change the key format to ${service_name}:${tenant_id}:${cache_name}:${raw_key} (This feature is available in cache 1.4.0).

But, this is not free, you must keep your data structure compatible for two consecutive releases.

1) If you add a new field, when your old service sees extra field when deserialization, that means you are reading the data cached by the newer version of your service, and you should ignore the unknown fields to avoid UnrecognizedPropertyException (this is ensured by the cache starter framework). When your new service see the field is missing from cache, you assume it's saved by the old version of the service and you have to re-populate the cache or provide a default value (this has to be ensured by yourself). And it's always a good practice to put a data structure version field into your cached entity for troubleshooting.

2) never rename a field, and follow the java bean standard for your setter/getter naming convention.

If you want bizx styled cache key you can configure the property "platform.cache.key-pattern" to "bizx".

#### Serialization - Transient Field
The transient field will not be serialized to Redis, so avoid using transient fields unless you expect those fields with null values from cache.

#### Serialization - Polymorphism
You should always cache very simple POJOs, don't cache any data with subclasses if possible. It makes the serialization/deserialization error prone and it's difficult for the framework to deserialize a field. 

```java
Object user = new User(); // cannot be deserialized from Redis/Jackson 
User user = new User(); // works 
void saveUser(String uid, Object user) // cannot be deserialized from Redis/Jackson 
void saveUser(String uid, User user) // does not work
```
Secondly, there are security risks. If for some reasons you really really need a field with subclasses, at least don't use it with Object/Serializable as field type. e.g.
```java
@JsonTypeInfo(use = Id.CLASS) private Object shape; // NEVER DO THIS!!!
```
If hackers can modify cached data in Redis by specifying a different subclass, the deserialized "shape" could execute remote command or retrieve extra sensitive information. It's a very complicated topic, for more details refer to Jackson CVEs

#### Serialization - Cyclic Dependencies
The data you are about to cache should be simple POJOs without any cyclic dependency. If you object has a reference back to itself, no matter directly or indirectly, you will see JsonMappingException. e.g.
```
myObject1.parent = myObject2; 
myObject2.child = myObject1; 
objectMapper.writeValueAsString(myObject1); // JsonMappingException: Infinite recursion (StackOverflowError) 
```

#### Serialization - Inner Class
Don't use non-static inner class, Jackson cannot handle them.

#### Serialization - Customization
You can use Jackson's built-in annotations like JsonTypeInfo to customize the serialization process. BUT DON'T support polymorphism unless you are an expert on Jackson serialization.

### Atomic Immediately Operations
By default, all caches are wrapped in TransactionAwareCacheDecorator, hence cache is updated after transaction commits in write through mode. e.g. you start a transaction and you call put(...) in your transaction, that will be registered as a deferred operation with TransactionSynchronizationManager. Before the transaction commits, the cache is not updated and other threads will not see the change in cache. Only after it commits successfully to database, TransactionSynchronizationManager then update the cache. This is a very useful feature of Spring cache. But putIfAbsent(...) and evictIfPresent(...) is atomic, it cannot be deferred after transaction commits. For this reason, both Spring annotation and our cache template does not support putIfAbsent. Don't manually get the Cache and call putIfAbsent(...).

You can enable this feature by platform.cache.transaction-aware: true. Once enabled, the PlatformCacheManager will wrap cache with TransactionAwareCacheDecorator and defer the cache operation after commits. e.g. service A calls service B and C, the database operations in B and C will be committed first, then the cache updates will be executed after the commit. If the transaction is rolled back, there will be no cache operations at all. Note, don't use localCacheManager or remoteCacheManager directly, they don't have this feature. And CacheTemplate does not support this too, when you use CacheTemplate means you have full control on your own, including transaction collaboration.

### Redis Timeout
There are several timeouts but connection timeout and command timeout are the most important ones. Connection timeout limits how long it takes to establish a Redis connection in Jedis/Lettuce pool. Command timeout limits how long it takes to execute the Redis command. In spring boot, both of them are configured by the same key in spring boot (spring.redis.timeout). The default value is 60 seconds which is too long, if your cache layer takes 60 seconds to figure out Redis is down definitely your application will consume all executor threads and hang. We strongly suggest shorten it to 2 seconds, or even shorter if you are confident about your application (if you have no hot key or big key, in most cases Redis command should return within a few milliseconds). It's always recommended to configure that value in your Consul so you can change it in production very easily to adapt your actual performance.

### Conclusions




### Evict The Whole Cache
Don't use @CacheEvict(allEntries=true) when using Redis Cache. Under the hood, Spring Redis Cache will call RedisCache.clear() to iterate all keys to find the matching keys then delete them one by one. If you are running a large redis cluster iterating all keys will go to all master nodes and cause high latency, sometimes even severe incident. For local cache managers like Caffeine it's perfectly fine, since the affect scope is very limited (one cache name, one JVM). See the code below from Spring.

```java
public void clear() { 
    byte[] pattern = conversionService.convert(createCacheKey("*"), byte[].class); 
    cacheWriter.clean(name, pattern); 
}
```

### Return void when put
The code below will not cache "s", Spring annotation takes the return value to cache, if you declare void, no error or even warning, but your value will never be cached.
```java
@CachePut(key = "#id") public void putSomething(String id, Something s)
```

### Cache Huge Object
If you put a huge value into cache, Redis cluster will be unbalanced and the node holding the huge value will be slowed down due to the heavy network IO. 

e.g.

1) if you have a single huge set and you don't SINTER, SUNION, or SDIFF, you just check membership by SISMEMBER, then consider using keys instead of a huge set to distribute data across nodes evenly.

2) if you have a huge JSON value to cache, split it into smaller granularity and only cached the needed part on demand.

### Hot Key
If you have a hot key, which is much more frequently accessed than other keys, the access to Redis cluster will be unbalanced and your system won't scale. Please avoid this in design. If it's unavoidable in design, use a local cache on top of Redis to mitigate the pressure to Redis.

### metrics
Always collect and monitor cache metrics. e.g., to enable micrometer metrics with elastic search, you just specify the elastic search host and index.
```yaml
management:
 metrics:
 enable:
 all: true
 export:
 elastic:
 enabled: true
 host: http://127.0.0.1:9200
 index: application-metrics
 step: 120s
```

For Caffeine, you need to add "recordStats" into your spec. e.g.
```yaml
 spec: maximumSize=100, expireAfterWrite=120s, recordStats
```


