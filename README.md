# myStudy

### redis高并发问题

缓存问题：

​	redis 缓存穿透、缓存击穿、缓存雪崩

#### 什么是缓存穿透？

``` yaml
	数据库和缓存数据库中都没有数据的时候，就是key不存在时，大量的数据进来查询DB。就会发生缓存穿透。
# 解决方案：
有很多解决方案，最常见的是采用布隆过滤器，将所有可能存在的数据hash到一个足够大bitmap中，一个一定不存在的数据会被这个bitmap拦截掉，从而避免对低层存储系统的查询压力。
还有一个简单粗暴的方法：如果一个查询返回的数据为空（不管是数据不存在，还是系统故障），我们仍然把这个空结果进行缓存，但它的过期时间很短，最长不超过五分钟。
```

#### 什么是缓存击穿?

```yaml
一些设置了过期时间的key，如果这些key可能会在某些时间点被超高并发地访问，是一种非常“热点”的数据。这个和缓存雪崩的区别在于这里针对某一key缓存，前者则是很多key
# 解决方案
问题的原因是同一时间差，同一时间写缓存，导致并发下缓存也没用，所以考虑使用单线程等方法将写缓存保证只有一个去查了写其他的使用缓存。
# 1.使用mutex，就是在缓存失效的时候（判断拿出来的值为空），不是立即去load db，而是先使用缓存工具的某些带成功操作返回值的操作（比如redis的SETNX或者Memcache的ADD）去set一个mutex key，当操作返回成功是，在进行load db的操作并回设缓存；否则，就重试整个get缓存方法。
```

#### 什么是缓存雪崩?

```yaml
缓存雪崩是指在我们设置缓存时采用了相同的过期时间，导致缓存在某一时刻同时失效，请求全部转发到DB，DB瞬时压力过重雪崩
# 解决方案：
缓存失效时雪崩效应对低层系统的冲击非常可怕，大多数系统设计者考虑加锁或者队列的方式保证缓存的单线程（进程）写，从而避免失效时大量的并发请求落到低层存储系统上。
# 在原有失效时间基础上增加一个随机值，比如1-5分钟随机，这样每一个缓存的过期时间的重复率就会降低，就很难引发集体失效的时间。

关键字：时间岔开，确保大家的key不会落在同一个expire点上
```

#### 1.使用互斥锁(mutex key)

```java
//2.6.1前单机版本锁  
String get(String key) {    
   String value = redis.get(key);    
   if (value  == null) {    
    if (redis.setnx(key_mutex, "1")) {    
        // 3 min timeout to avoid mutex holder crash    
        redis.expire(key_mutex, 3 * 60)    
        value = db.get(key);    
        redis.set(key, value);    
        redis.delete(key_mutex);    
    } else {    
        //其他线程休息50毫秒后重试    
        Thread.sleep(50);    
        get(key);    
    }    
  }    
}  
最新版本代码：
[java] view plain copy
public String get(key) {  
      String value = redis.get(key);  
      if (value == null) { //代表缓存值过期  
          //设置3min的超时，防止del操作失败的时候，下次缓存过期一直不能load db  
          if (redis.setnx(key_mutex, 1, 3 * 60) == 1) {  //代表设置成功  
               value = db.get(key);  
                      redis.set(key, value, expire_secs);  
                      redis.del(key_mutex);  
              } else {  //这个时候代表同时候的其他线程已经load db并回设到缓存了，这时候重试获取缓存值即可  
                      sleep(50);  
                      get(key);  //重试  
              }  
          } else {  
              return value;        
          }  
 }  
memcache代码：
[java] view plain copy
if (memcache.get(key) == null) {    
    // 3 min timeout to avoid mutex holder crash    
    if (memcache.add(key_mutex, 3 * 60 * 1000) == true) {    
        value = db.get(key);    
        memcache.set(key, value);    
        memcache.delete(key_mutex);    
    } else {    
        sleep(50);    
        retry();    
    }    
}   
```

#### “提前”使用互斥锁(mutex key)：

```java
v = memcache.get(key);    
if (v == null) {    
    if (memcache.add(key_mutex, 3 * 60 * 1000) == true) {    
        value = db.get(key);    
        memcache.set(key, value);    
        memcache.delete(key_mutex);    
    } else {    
        sleep(50);    
        retry();    
    }    
} else {    
    if (v.timeout <= now()) {    
        if (memcache.add(key_mutex, 3 * 60 * 1000) == true) {    
            // extend the timeout for other threads    
            v.timeout += 3 * 60 * 1000;    
            memcache.set(key, v, KEY_TIMEOUT * 2);    

            // load the latest value from db    
            v = db.get(key);    
            v.timeout = KEY_TIMEOUT;    
            memcache.set(key, value, KEY_TIMEOUT * 2);    
            memcache.delete(key_mutex);    
        } else {    
            sleep(50);    
            retry();    
        }    
    }    
}   
```

#### 永远不过期”

```java
String get(final String key) {    
        V v = redis.get(key);    
        String value = v.getValue();    
        long timeout = v.getTimeout();    
        if (v.timeout <= System.currentTimeMillis()) {    
            // 异步更新后台异常执行    
            threadPool.execute(new Runnable() {    
                public void run() {    
                    String keyMutex = "mutex:" + key;    
                    if (redis.setnx(keyMutex, "1")) {    
                        // 3 min timeout to avoid mutex holder crash    
                        redis.expire(keyMutex, 3 * 60);    
                        String dbValue = db.get(key);    
                        redis.set(key, dbValue);    
                        redis.delete(keyMutex);    
                    }    
                }    
            });    
        }    
        return value;    
}  
```

####  资源保护：

```
采用netflix的hystrix，可以做资源的隔离保护主线程池，如果把这个应用到缓存的构建也未尝不可。
```

[博客链接](https://blog.csdn.net/doujinlong1/article/details/82024340)

### Spring Cloud Alibab Dubbo

#### Nacos  [官方文档](https://nacos.io/zh-cn/docs/quick-start.html)

### 附

在spring cloud G版中对eureka加密中需要通过java配置





​	

​	