

# soul-study

#### 一、结合divde插件，发起http请求soul网关，体验http代理

##### 1. 前言

* 通过`day01`的学习，我们大致了解 `soul`的divide插件是网关处理 `http协议`请求的核心处理插件，通过配置`选择器`与`规则`可以实现通过

  网关对`http`请求的代理，因为，本文主要学习 `soul-divide` 插件如果代理 `http`请求，如果`负载均衡`及其相关源码解析

##### 2. http代理过程

* 分别启动 `soul-admin`、`soul-bootstrap`、`soul-examples-http` 

* 启动 `soul-examples-http`  时，会自动将`soul-examples-http` 的信息写入到`admin`后台中，如下：

  ![image-20210115232607670](pictures\image-20210115232607670.png)

* 指定不同端口，启动另外一个 `soul-examples-http`，启动成功后，在后台选择其中可以看到对应的服务信息已经写入到选择器中

  ![image-20210115232943375](pictures\image-20210115232943375.png)

* 下面我们通过`soul`网关访问两个服务，体验网关带来的负载均衡

  注意：当前负载均衡的权重为 50:50

  * 可以使用idea自带的`HttpClient`工具测试

    ![image-20210115233603270](pictures\image-20210115233603270.png)

* 当我们随机发起多次请求是，每一个`http`服务依次被访问，既每一个服务被访问的概率是相同的，如下：

  ![image-20210115234358111](pictures\image-20210115234358111.png)

  综上：可以验证负载均衡策略是成功的

##### 3. 深入分析

* 我们在分析源码时，可以首先全局的看一下整个功能的结构，了解每一个模块的大概作用，所以，本次我们分析`divide`插件时，自然定位到如下模块下学习具体原理

  ````json
  soul-plugin --> soul-plugin-divide 模块
  ````

* 当我们确定了模块以后，如果`Debugger`时，断点打在哪里呢？

  * 在大致查看了`soul-plugin-divide`代码结构后，`DividePlugin#doExecute`   上，如下：

  ````java
  protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
          final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
          assert soulContext != null;
          final DivideRuleHandle ruleHandle = GsonUtils.getInstance().fromJson(rule.getHandle(), DivideRuleHandle.class);
          final List<DivideUpstream> upstreamList = UpstreamCacheManager.getInstance().findUpstreamListBySelectorId(selector.getId());
          if (CollectionUtils.isEmpty(upstreamList)) {
              log.error("divide upstream configuration error： {}", rule.toString());
              Object error = SoulResultWrap.error(SoulResultEnum.CANNOT_FIND_URL.getCode(), SoulResultEnum.CANNOT_FIND_URL.getMsg(), null);
              return WebFluxResultUtils.result(exchange, error);
          }
          final String ip = Objects.requireNonNull(exchange.getRequest().getRemoteAddress()).getAddress().getHostAddress();
          DivideUpstream divideUpstream = LoadBalanceUtils.selector(upstreamList, ruleHandle.getLoadBalance(), ip);
          if (Objects.isNull(divideUpstream)) {
              log.error("divide has no upstream");
              Object error = SoulResultWrap.error(SoulResultEnum.CANNOT_FIND_URL.getCode(), SoulResultEnum.CANNOT_FIND_URL.getMsg(), null);
              return WebFluxResultUtils.result(exchange, error);
          }
          // set the http url
          String domain = buildDomain(divideUpstream);
          String realURL = buildRealURL(domain, soulContext, exchange);
          exchange.getAttributes().put(Constants.HTTP_URL, realURL);
          // set the http timeout
          exchange.getAttributes().put(Constants.HTTP_TIME_OUT, ruleHandle.getTimeout());
          exchange.getAttributes().put(Constants.HTTP_RETRY, ruleHandle.getRetry());
          return chain.execute(exchange);
      }
  ````

  

  * 当发送请求时，果然可以进入断点，此时，我们需要分析函数的调用情况,可以发现，有两处函数的调用，如下：

  ```java
  @Override
          public Mono<Void> execute(final ServerWebExchange exchange) {
              return Mono.defer(() -> {
                  // plugins 保存所有插件的集合，如下
                  if (this.index < plugins.size()) {
                      SoulPlugin plugin = plugins.get(this.index++);
                      Boolean skip = plugin.skip(exchange);
                      if (skip) {
                          return this.execute(exchange);
                      }
                      return plugin.execute(exchange, this);
                  }
                  return Mono.empty();
              });
          }
  ```

  ​		在经过不管的调试后，可以发现，`plugins`是保存所有插件的集合，`skip`决定是否跳过执行此插件，`plugin.execute(exchange, this)`执行插件，会执行到如下方法中。

  ```
    @Override
        public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
            String pluginName = named();
            final PluginData pluginData = BaseDataCache.getInstance().obtainPluginData(pluginName);
            if (pluginData != null && pluginData.getEnabled()) {
                final Collection<SelectorData> selectors = BaseDataCache.getInstance().obtainSelectorData(pluginName);
                if (CollectionUtils.isEmpty(selectors)) {
                    return handleSelectorIsNull(pluginName, exchange, chain);
                }
                final SelectorData selectorData = matchSelector(exchange, selectors);
                if (Objects.isNull(selectorData)) {
                    return handleSelectorIsNull(pluginName, exchange, chain);
                }
                selectorLog(selectorData, pluginName);
                final List<RuleData> rules = BaseDataCache.getInstance().obtainRuleData(selectorData.getId());
                if (CollectionUtils.isEmpty(rules)) {
                    return handleRuleIsNull(pluginName, exchange, chain);
                }
                RuleData rule;
                if (selectorData.getType() == SelectorTypeEnum.FULL_FLOW.getCode()) {
                    //get last
                    rule = rules.get(rules.size() - 1);
                } else {
                    rule = matchRule(exchange, rules);
                }
                if (Objects.isNull(rule)) {
                    return handleRuleIsNull(pluginName, exchange, chain);
                }
                ruleLog(rule, pluginName);
                return doExecute(exchange, chain, selectorData, rule);
            }
            return chain.execute(exchange);
        }
  ```

  ​	经过分析，`execute`中会获取插件的一些基本信息，例如`选择器`、`规则`、`插件元信息`等，最后`doExecute(exchange, chain, selectorData, rule)`才是真正执行对应规则的方法，以`divide`插件为例，代码如下:

  ```java
  @Override
      protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
          final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
          assert soulContext != null;
          final DivideRuleHandle ruleHandle = GsonUtils.getInstance().fromJson(rule.getHandle(), DivideRuleHandle.class);
          final List<DivideUpstream> upstreamList = UpstreamCacheManager.getInstance().findUpstreamListBySelectorId(selector.getId());
          if (CollectionUtils.isEmpty(upstreamList)) {
              log.error("divide upstream configuration error： {}", rule.toString());
              Object error = SoulResultWrap.error(SoulResultEnum.CANNOT_FIND_URL.getCode(), SoulResultEnum.CANNOT_FIND_URL.getMsg(), null);
              return WebFluxResultUtils.result(exchange, error);
          }
          final String ip = Objects.requireNonNull(exchange.getRequest().getRemoteAddress()).getAddress().getHostAddress();
          // 负载均衡
          DivideUpstream divideUpstream = LoadBalanceUtils.selector(upstreamList, ruleHandle.getLoadBalance(), ip);
          if (Objects.isNull(divideUpstream)) {
              log.error("divide has no upstream");
              Object error = SoulResultWrap.error(SoulResultEnum.CANNOT_FIND_URL.getCode(), SoulResultEnum.CANNOT_FIND_URL.getMsg(), null);
              return WebFluxResultUtils.result(exchange, error);
          }
          // set the http url
          String domain = buildDomain(divideUpstream);
          // 解析获取真正需要调用的服务器地址及方法
          String realURL = buildRealURL(domain, soulContext, exchange);
          exchange.getAttributes().put(Constants.HTTP_URL, realURL);
          // set the http timeout
          exchange.getAttributes().put(Constants.HTTP_TIME_OUT, ruleHandle.getTimeout());
          exchange.getAttributes().put(Constants.HTTP_RETRY, ruleHandle.getRetry());
          return chain.execute(exchange);
      }
  ```

  ​	当经过负载均衡、获取服务器地址等操作后，`chain.execute(exchange)` 可以理解为，直接对解析后的地址发送`HttpClient`请求，通过`response`获取到返回值, 具体执行代码如下：

  ```
  @Override
      public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
          final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
          assert soulContext != null;
          String urlPath = exchange.getAttribute(Constants.HTTP_URL);
          if (StringUtils.isEmpty(urlPath)) {
              Object error = SoulResultWrap.error(SoulResultEnum.CANNOT_FIND_URL.getCode(), SoulResultEnum.CANNOT_FIND_URL.getMsg(), null);
              return WebFluxResultUtils.result(exchange, error);
          }
          long timeout = (long) Optional.ofNullable(exchange.getAttribute(Constants.HTTP_TIME_OUT)).orElse(3000L);
          int retryTimes = (int) Optional.ofNullable(exchange.getAttribute(Constants.HTTP_RETRY)).orElse(0);
          log.info("The request urlPath is {}, retryTimes is {}", urlPath, retryTimes);
          HttpMethod method = HttpMethod.valueOf(exchange.getRequest().getMethodValue());
          WebClient.RequestBodySpec requestBodySpec = webClient.method(method).uri(urlPath);
          return handleRequestBody(requestBodySpec, exchange, timeout, retryTimes, chain);
      }
  ```

  ````
   private Mono<Void> handleRequestBody(final WebClient.RequestBodySpec requestBodySpec,
                                           final ServerWebExchange exchange,
                                           final long timeout,
                                           final int retryTimes,
                                           final SoulPluginChain chain) {
          return requestBodySpec.headers(httpHeaders -> {
              httpHeaders.addAll(exchange.getRequest().getHeaders());
              httpHeaders.remove(HttpHeaders.HOST);
          })
                  .contentType(buildMediaType(exchange))
                  .body(BodyInserters.fromDataBuffers(exchange.getRequest().getBody()))
                  .exchange()
                  .doOnError(e -> log.error(e.getMessage()))
                  .timeout(Duration.ofMillis(timeout))
                  .retryWhen(Retry.onlyIf(x -> x.exception() instanceof ConnectTimeoutException)
                      .retryMax(retryTimes)
                      .backoff(Backoff.exponential(Duration.ofMillis(200), Duration.ofSeconds(20), 2, true)))
                  .flatMap(e -> doNext(e, exchange, chain));
  
      }
  ````

  ​	当时，对`execute(final ServerWebExchange exchange, final SoulPluginChain chain)` 方法返回到哪里，并且客户端是如何接受到返回值还存在疑惑之处？系统通过后面的不断加深学习，从而找到答案

##### 4. 总结

* `divide` 插件处理`http` 请求大致流程如下

  ![image-20210116021010197](pictures\image-20210116021010197.png)



总体来说，soul 网关的 divide 插件在处理http 请求是，主要核心逻辑就是，通过 `选择器`、`规则`来实现请求的代理，例如：

1) 请求  http://localhost:9195/http/order/findById?id=1  

2) 经过中间divide 插件的处理，最终会使用真正的请求地址去访问服务器，地址可能会为 http://192.168.56.1:8188/order/findById?id=1

3) 客户端接收返回值



以上为soul网关 divide 插件如何代理 http 的分析过程，还存在诸多不足之处，同时，还需要去学习 `响应式编程模式`等前置知识。





