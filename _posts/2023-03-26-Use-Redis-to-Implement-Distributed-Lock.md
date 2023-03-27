## Use Redis to Implement Distributed Lock

We can use Redis's property "set the key if not exist" to implement the distributed Lock. In Redis there is a command called SETNX, The term SETNX is an abbreviation of the phrase “setting the key if not exists”; thus, the command will not run if the key value already exists in Redis.

When writing in Java code, we should import the jar package below.

```xml
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>2.9.0</version>
</dependency>
```

#### To Add a Lock
jedis.set(String key, String value, String nxxx, String expx, int time), this set() method has five parameters:

The first parameter is key, the key is unique id for locking.

The second parameter is value, we pass in requestId. We use this requestId to know which request acquired this lock. We can use UUID.randomUUID().toString() to generate a requestId.

The third parameter is NX, means SET IF NOT EXIST. When key doesn't exist, the key will be set. If the key already exists, it will do nothing.

The fourth parameter is PX, it is to set the expiration time of the key.

The fifth parameter is time, represent the expiration time for the fourth parameter.

Notice: We need to add an expiration time to avoid deadlock.  Use UUID to guarantee when deleting a key, it will below to the request. The keys belonging to other requests will not be deleted.

#### To Release a Lock

#### The following is Sample Java Code to Lock or Release Distributed Lock

```java
public class RedisLock {
    public static final String LOCK_SUCCESS = "OK";
    private static final Long RELEASE_SUCCESS = 1L;
    private static final int MAX_TYR_COUNT = 10;

    @Autowired
    private JedisPool jedisPool;

    /**
     * Get Lock
     *
     * @param lockKey    lock
     * @param requestId  request
     * @param expireTime set expiration time
     * @return success or failure
     */
    public boolean tryGetLock(String lockKey, String requestId, int expireTime) {
        Jedis jedisClient = jedisPool.getResource();
        int tryCount = 0;
        try {
            do {
                String result = jedisClient.set(lockKey, requestId, "NX", "PX", expireTime);
                if (LOCK_SUCCESS.equals(result)) {
                    return true;
                }
                Thread.sleep(100);
                tryCount++;
            } while (tryCount < MAX_TYR_COUNT);
            log.error("tryLock error,lockKey:{}", lockKey);
        } catch (Throwable e) {
            log.error("tryLock error,lockKey:{}", lockKey, e);
        } finally {
            jedisClient.close();
        }
        return false;
    }

    /**
     * unlock
     * use lua to guarantee atomic
     *
     * @param lockKey   lock
     * @param requestId requestid
     * @return
     */
    public boolean releaseLock(String lockKey, String requestId) {
        String script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
        Jedis jedis = jedisPool.getResource();
        Object result = jedis.eval(script, Collections.singletonList(lockKey), Collections.singletonList(requestId));
        jedis.close();
        if (RELEASE_SUCCESS.equals(result))
            return true;
        return false;
    }
}
```

