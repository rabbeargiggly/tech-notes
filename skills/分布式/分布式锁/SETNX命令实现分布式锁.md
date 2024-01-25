# SETNX命令实现分布式锁



```java
private static String LOCK_PREFIX = "prefix";

public boolean lock(String key) {

        String lock = LOCK_PREFIX + key;

        return (Boolean) redisTemplate.execute((RedisCallback) connection -> {

            long expireAt = System.currentTimeMillis() + LOCK_EXPIRE + 1;
            Boolean acquire = connection.setNX(lock.getBytes(), String.valueOf(expireAt).getBytes());


            if (acquire) {
                return true;
            } else {
              // setNX()失败，则判断过期时间
                byte[] value = connection.get(lock.getBytes());

                if (Objects.nonNull(value) && value.length > 0) {

                    long expireTime = Long.parseLong(new String(value));

                    if (expireTime < System.currentTimeMillis()) {
                      // 已过期，可尝试加锁，注意，可能会有多个线程进入该if语句内
                        byte[] oldValue = connection.getSet(lock.getBytes(), String.valueOf(System.currentTimeMillis() + LOCK_EXPIRE + 1).getBytes());

                      // 高并发场景下，如果是别的线程先获取到了锁，那么判断oldValue与当前时间就会为false
                        return Long.parseLong(new String(oldValue)) < System.currentTimeMillis();
                    }
                }
            }
            return false;
        });
    }
```

