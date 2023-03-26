## This is My First Post

This is my First Post.

Hello Word!

Test Test Test.

---

### This is a header

#### Some Java Code

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
        //重试的次数
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

