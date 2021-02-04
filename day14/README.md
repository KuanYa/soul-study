#### rateLimater

##### 1.lua 脚本

* `rateLimater` 限流是基于`lua` 脚本的，所以，分析一下脚本

```lua
-- 令牌
local tokens_key = KEYS[1]
-- 时间戳
local timestamp_key = KEYS[2]
--redis.log(redis.LOG_WARNING, "tokens_key " .. tokens_key)

-- 速率
local rate = tonumber(ARGV[1])
-- 容量
local capacity = tonumber(ARGV[2])
-- 当前时间（秒）
local now = tonumber(ARGV[3])
-- 请求数（默认1）
local requested = tonumber(ARGV[4])

-- 容量/速率 理解为填满整个桶需要的时间（单位秒）
local fill_time = capacity/rate
-- 
local ttl = math.floor(fill_time*2)

--redis.log(redis.LOG_WARNING, "rate " .. ARGV[1])
--redis.log(redis.LOG_WARNING, "capacity " .. ARGV[2])
--redis.log(redis.LOG_WARNING, "now " .. ARGV[3])
--redis.log(redis.LOG_WARNING, "requested " .. ARGV[4])
--redis.log(redis.LOG_WARNING, "filltime " .. fill_time)
--redis.log(redis.LOG_WARNING, "ttl " .. ttl)

-- 最后的令牌数量
local last_tokens = tonumber(redis.call("get", tokens_key))
if last_tokens == nil then
  last_tokens = capacity
end
--redis.log(redis.LOG_WARNING, "last_tokens " .. last_tokens)
-- 最后的时间戳
local last_refreshed = tonumber(redis.call("get", timestamp_key))
if last_refreshed == nil then
  last_refreshed = 0
end
--redis.log(redis.LOG_WARNING, "last_refreshed " .. last_refreshed)
-- 可以理解为执行的时间片长度 
local delta = math.max(0, now-last_refreshed)
-- delta*rate 需要填充的令牌数量 ，取出最小的数量，作为可以允许通过的请求数
local filled_tokens = math.min(capacity, last_tokens+(delta*rate))
-- 是否允许请求通过 
local allowed = filled_tokens >= requested
-- 设置新的容量大小
local new_tokens = filled_tokens
local allowed_num = 0
if allowed then
   -- 桶数量建议
  new_tokens = filled_tokens - requested
  allowed_num = 1
end

--redis.log(redis.LOG_WARNING, "delta " .. delta)
--redis.log(redis.LOG_WARNING, "filled_tokens " .. filled_tokens)
--redis.log(redis.LOG_WARNING, "allowed_num " .. allowed_num)
--redis.log(redis.LOG_WARNING, "new_tokens " .. new_tokens)

redis.call("setex", tokens_key, ttl, new_tokens)
redis.call("setex", timestamp_key, ttl, now)

return { allowed_num, new_tokens }
```

* 核心代码

```java
public Mono<RateLimiterResponse> isAllowed(final String id, final double replenishRate, final double burstCapacity) {
    if (!this.initialized.get()) {
        throw new IllegalStateException("RedisRateLimiter is not initialized");
    }
    List<String> keys = getKeys(id);
    List<String> scriptArgs = Arrays.asList(replenishRate + "", burstCapacity + "", Instant.now().getEpochSecond() + "", "1");
    Flux<List<Long>> resultFlux = Singleton.INST.get(ReactiveRedisTemplate.class).execute(this.script, keys, scriptArgs);
    return resultFlux.onErrorResume(throwable -> Flux.just(Arrays.asList(1L, -1L)))
            .reduce(new ArrayList<Long>(), (longs, l) -> {
                longs.addAll(l);
                return longs;
            }).map(results -> {
        		// 会获取 lua 中的返回值，如果 allowed 则 抛弃请求
                boolean allowed = results.get(0) == 1L;
                Long tokensLeft = results.get(1);
                RateLimiterResponse rateLimiterResponse = new RateLimiterResponse(allowed, tokensLeft);
                log.info("RateLimiter response:{}", rateLimiterResponse.toString());
                return rateLimiterResponse;
            }).doOnError(throwable -> log.error("Error determining if user allowed from redis:{}", throwable.getMessage()));
}
```

* 总结：

  在处理请求之前先要从桶中获得一个令牌，如果桶中已经没有了令牌，那么就需要等待新的令牌或者直接拒绝服务；

  桶中的令牌总数也要有一个限制，如果超过了限制就不能向桶中再增加新的令牌了。这样可以限制令牌的总数，一定程度上可以避免瞬时流量高峰的问题。