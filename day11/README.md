#### `Resilience4J`插件学习

##### 1.` Resilience4J` 简介

> Resilience4j是受到Netflix Hystrix的启发，为Java8和函数式编程所设计的轻量级容错框架。整个框架只是使用了Varr的库，不需要引入其他的外部依赖。与此相比，Netflix Hystrix对Archaius具有编译依赖，而Archaius需要更多的外部依赖，例如Guava和Apache Commons Configuration。
>
> Resilience4j提供了提供了一组高阶函数（装饰器），包括断路器，限流器，重试机制，隔离机制。你可以使用其中的一个或多个装饰器对函数式接口，lambda表达式或方法引用进行装饰。这么做的优点是你可以选择所需要的装饰器进行装饰。

##### 2. 规则配置

|                    参数                     |                             含义                             |
| :-----------------------------------------: | :----------------------------------------------------------: |
|          token filling period (ms)          |        等待获取令牌的超时时间，单位ms，默认值：5000。        |
|            token filling number             |                                                              |
|        control behavior timeout (ms)        |           熔断器开启持续时间，单位ms，默认值：10。           |
|               circuit enable                |         是否开启熔断，0：关闭，1：开启，默认值：0。          |
|            circuit timeout (ms)             |             熔断超时时间，单位ms，默认值：30000              |
|                fallback uri                 |                       降级处理的uri。                        |
|             sliding window size             |                  滑动窗口大小，默认值：100                   |
|             sliding window type             |      滑动窗口类型，0：基于计数，1：基于时间，默认值：0       |
| enabled error minimum calculation threshold | 开启熔断的最小请求数，超过这个请求数才开启熔断统计，默认值：100。 |
|          degrade opening duration           |                                                              |
|             half open threshold             | 半开状态下的环形缓冲区大小，必须达到此数量才会计算失败率，默认值：10。 |
|            degrade failure rate             |    错误率百分比，达到这个阈值，熔断器才会开启，默认值50。    |

**以上参数整理不一定正确**，官网与现在版本存在差异



##### 3.相关资料

[soul官网](https://dromara.org/zh/projects/soul/resilience4j-plugin/)

[resilience4j 源码地址](https://github.com/resilience4j/resilience4j)

##### 4. 核心代码

```java
@Override
protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
    final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
    assert soulContext != null;
    // 规则的值需要设置，否则会报错
    Resilience4JHandle resilience4JHandle = GsonUtils.getGson().fromJson(rule.getHandle(), Resilience4JHandle.class);
    if (resilience4JHandle.getCircuitEnable() == 1) {
        // 熔断
        return combined(exchange, chain, rule);
    }
    return rateLimiter(exchange, chain, rule);
}
```

```java
private Mono<Void> combined(final ServerWebExchange exchange, final SoulPluginChain chain, final RuleData rule) {
    Resilience4JConf conf = Resilience4JBuilder.build(rule);
    return combinedExecutor.run(
            chain.execute(exchange).doOnSuccess(v -> {
                if (exchange.getResponse().getStatusCode() != HttpStatus.OK) {
                    HttpStatus status = exchange.getResponse().getStatusCode();
                    exchange.getResponse().setStatusCode(null);
                    throw new CircuitBreakerStatusCodeException(status);
                }
            }), fallback(combinedExecutor, exchange, conf.getFallBackUri()), conf);
}
```

```java
// 感觉此代码是熔断的核心
public class CombinedExecutor implements Executor {

    @Override
    public <T> Mono<T> run(final Mono<T> run, 
    final Function<Throwable, Mono<T>> fallback, 
    final Resilience4JConf resilience4JConf) {
        RateLimiter rateLimiter = Resilience4JRegistryFactory.rateLimiter(resilience4JConf.getId(), resilience4JConf.getRateLimiterConfig());
        // 这里是构建一个断路器
        CircuitBreaker circuitBreaker = Resilience4JRegistryFactory.circuitBreaker(resilience4JConf.getId(), resilience4JConf.getCircuitBreakerConfig());
        Mono<T> to = run.transformDeferred(CircuitBreakerOperator.of(circuitBreaker))
                .transformDeferred(RateLimiterOperator.of(rateLimiter))
                .timeout(resilience4JConf.getTimeLimiterConfig().getTimeoutDuration())
                .doOnError(TimeoutException.class, t -> circuitBreaker.onError(
                        resilience4JConf.getTimeLimiterConfig().getTimeoutDuration().toMillis(),
                        TimeUnit.MILLISECONDS,
                        t));
        if (fallback != null) {
            to = to.onErrorResume(fallback);
        }
        return to;
    }
}
```



总结：今天先总结到这，`Resilience4J`熔断器的核心原理还未搞懂，明天继续分析，感觉对这块挺感兴趣，深入分析下去，惭愧，惭愧