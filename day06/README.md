#### Soul数据同步至`WebSocket`原理

##### 1. Websocket 连接如何建立

* 启动`soul-admin`

  * 创建`websocket`监听，`WebsocketListener.java`

  ```java
  public class WebsocketDataChangedListener implements DataChangedListener {
  	
      // 插件
      @Override
      public void onPluginChanged(final List<PluginData> pluginDataList, final DataEventTypeEnum eventType) {
          WebsocketData<PluginData> websocketData =
                  new WebsocketData<>(ConfigGroupEnum.PLUGIN.name(), eventType.name(), pluginDataList);
          WebsocketCollector.send(GsonUtils.getInstance().toJson(websocketData), eventType);
      }
  	
      // 选择器
      @Override
      public void onSelectorChanged(final List<SelectorData> selectorDataList, final DataEventTypeEnum eventType) {
          WebsocketData<SelectorData> websocketData =
                  new WebsocketData<>(ConfigGroupEnum.SELECTOR.name(), eventType.name(), selectorDataList);
          WebsocketCollector.send(GsonUtils.getInstance().toJson(websocketData), eventType);
      }
  	
      // 规则
      @Override
      public void onRuleChanged(final List<RuleData> ruleDataList, final DataEventTypeEnum eventType) {
          WebsocketData<RuleData> configData =
                  new WebsocketData<>(ConfigGroupEnum.RULE.name(), eventType.name(), ruleDataList);
          WebsocketCollector.send(GsonUtils.getInstance().toJson(configData), eventType);
      }
  	
      // 应用
      @Override
      public void onAppAuthChanged(final List<AppAuthData> appAuthDataList, final DataEventTypeEnum eventType) {
          WebsocketData<AppAuthData> configData =
                  new WebsocketData<>(ConfigGroupEnum.APP_AUTH.name(), eventType.name(), appAuthDataList);
          WebsocketCollector.send(GsonUtils.getInstance().toJson(configData), eventType);
      }
  	
      // 元数据
      @Override
      public void onMetaDataChanged(final List<MetaData> metaDataList, final DataEventTypeEnum eventType) {
          WebsocketData<MetaData> configData =
                  new WebsocketData<>(ConfigGroupEnum.META_DATA.name(), eventType.name(), metaDataList);
          WebsocketCollector.send(GsonUtils.getInstance().toJson(configData), eventType);
      }
  
  }
  ```

  * 初始化 `WebsocketCollector.java`  主要使用 `@ServerEndpoint("/websocket")` 注解

* 启动 `soul-bootstrap`

  * 初始化`websocketSyncDataService`，


  ```java
  @Bean
  public SyncDataService websocketSyncDataService(final ObjectProvider<WebsocketConfig> websocketConfig, 
                                                  final ObjectProvider<PluginDataSubscriber> pluginSubscriber,
                                         		 final ObjectProvider<List<MetaDataSubscriber>> metaSubscribers, final ObjectProvider<List<AuthDataSubscriber>> authSubscribers) {
      log.info("you use websocket sync soul data.......");
      // websocketConfig.getIfAvailable(WebsocketConfig::new) Websocket 地址
      return new WebsocketSyncDataService(websocketConfig.getIfAvailable(WebsocketConfig::new), pluginSubscriber.getIfAvailable(),
              metaSubscribers.getIfAvailable(Collections::emptyList), authSubscribers.getIfAvailable(Collections::emptyList));
  }
  ```

```java
public WebsocketSyncDataService(final WebsocketConfig websocketConfig,
                                final PluginDataSubscriber pluginDataSubscriber,
                                final List<MetaDataSubscriber> metaDataSubscribers,
                                final List<AuthDataSubscriber> authDataSubscribers) {
    // 可以配置多个websocket 连接地址
    String[] urls = StringUtils.split(websocketConfig.getUrls(), ",");
    // 创建定时任务的线程池
    executor = new ScheduledThreadPoolExecutor(urls.length, SoulThreadFactory.create("websocket-connect", true));
    for (String url : urls) {
        try {
            // 定义每一个客户端的 handler
            clients.add(new SoulWebsocketClient(new URI(url), Objects.requireNonNull(pluginDataSubscriber), metaDataSubscribers, authDataSubscribers));
        } catch (URISyntaxException e) {
            log.error("websocket url({}) is error", url, e);
        }
    }
    try {
        for (WebSocketClient client : clients) {
            // 确定是否能连接成功
            boolean success = client.connectBlocking(3000, TimeUnit.MILLISECONDS);
            if (success) {
                log.info("websocket connection is successful.....");
            } else {
                log.error("websocket connection is error.....");
            }
            
            // 开启定时任务线程池，每30秒重新连接
            executor.scheduleAtFixedRate(() -> {
                try {
                    if (client.isClosed()) {
                        boolean reconnectSuccess = client.reconnectBlocking();
                        if (reconnectSuccess) {
                            log.info("websocket reconnect is successful.....");
                        } else {
                            log.error("websocket reconnection is error.....");
                        }
                    }
                } catch (InterruptedException e) {
                    log.error("websocket connect is error :{}", e.getMessage());
                }
            }, 10, 30, TimeUnit.SECONDS);
        }
        /* client.setProxy(new Proxy(Proxy.Type.HTTP, new InetSocketAddress("proxyaddress", 80)));*/
    } catch (InterruptedException e) {
        log.info("websocket connection...exception....", e);
    }

}
```

当 与`websocket`连接成功后，存在如下动作：

```java
// 保存session
@OnOpen
public void onOpen(final Session session) {
    log.info("websocket on open successful....");
    SESSION_SET.add(session);
}
```

```java
// 发送消息
@OnMessage
public void onMessage(final String message, final Session session) {
    if (message.equals(DataEventTypeEnum.MYSELF.name())) {
        WebsocketCollector.session = session;
        // 获取bean，执行实现类中的方法
        SpringBeanUtils.getInstance().getBean(SyncDataService.class).syncAll(DataEventTypeEnum.MYSELF);
    }
}
```

```java
// 全量同步消息
@Override
public boolean syncAll(final DataEventTypeEnum type) {
    // 应用
    appAuthService.syncData();
    List<PluginData> pluginDataList = pluginService.listAll();
    eventPublisher.publishEvent(new DataChangedEvent(ConfigGroupEnum.PLUGIN, type, pluginDataList));
    List<SelectorData> selectorDataList = selectorService.listAll();
    eventPublisher.publishEvent(new DataChangedEvent(ConfigGroupEnum.SELECTOR, type, selectorDataList));
    List<RuleData> ruleDataList = ruleService.listAll();
    eventPublisher.publishEvent(new DataChangedEvent(ConfigGroupEnum.RULE, type, ruleDataList));
    metaDataService.syncData();
    return true;
}
```

小结：通过上面的启动过程，可以发现在启动过程中，会通过`websocket` 全量 同步一次数据，但是，`websocket` 是存在增量同步数据的，下面我们来查看是如何实现的

##### 2. 增量同步数据

* 在前台修改 dubbo 插件，会触发下列动作

```java
// 发送消息
public static void send(final String message, final DataEventTypeEnum type) {
    if (StringUtils.isNotBlank(message)) {
        if (DataEventTypeEnum.MYSELF == type) {
            try {
                session.getBasicRemote().sendText(message);
            } catch (IOException e) {
                log.error("websocket send result is exception: ", e);
            }
            return;
        }
        for (Session session : SESSION_SET) {
            try {
                session.getBasicRemote().sendText(message);
            } catch (IOException e) {
                log.error("websocket send result is exception: ", e);
            }
        }
    }
}
```

```java
// 执行更新动作
@Override
public void handle(final String json, final String eventType) {
    List<T> dataList = convert(json);
    if (CollectionUtils.isNotEmpty(dataList)) {
        DataEventTypeEnum eventTypeEnum = DataEventTypeEnum.acquireByName(eventType);
        switch (eventTypeEnum) {
            case REFRESH:
            case MYSELF:
                doRefresh(dataList);
                break;
            case UPDATE: // update 与 create 使用一个方法？
            case CREATE: // update 与 create 使用一个方法？
                doUpdate(dataList);
                break;
            case DELETE:
                doDelete(dataList);
                break;
            default:
                break;
        }
    }
}
```

```java
private <T> void subscribeDataHandler(final T classData, final DataEventTypeEnum dataType) {
    Optional.ofNullable(classData).ifPresent(data -> {
        if (data instanceof PluginData) {
            PluginData pluginData = (PluginData) data;
            if (dataType == DataEventTypeEnum.UPDATE) {
                // 更新缓存中的插件信息
                BaseDataCache.getInstance().cachePluginData(pluginData);
                // 执行更新
                Optional.ofNullable(handlerMap.get(pluginData.getName())).ifPresent(handler -> handler.handlerPlugin(pluginData));
            } else if (dataType == DataEventTypeEnum.DELETE) {
                BaseDataCache.getInstance().removePluginData(pluginData);
                Optional.ofNullable(handlerMap.get(pluginData.getName())).ifPresent(handler -> handler.removePlugin(pluginData));
            }
        } else if (data instanceof SelectorData) {
            SelectorData selectorData = (SelectorData) data;
            if (dataType == DataEventTypeEnum.UPDATE) {
                BaseDataCache.getInstance().cacheSelectData(selectorData);
                Optional.ofNullable(handlerMap.get(selectorData.getPluginName())).ifPresent(handler -> handler.handlerSelector(selectorData));
            } else if (dataType == DataEventTypeEnum.DELETE) {
                BaseDataCache.getInstance().removeSelectData(selectorData);
                Optional.ofNullable(handlerMap.get(selectorData.getPluginName())).ifPresent(handler -> handler.removeSelector(selectorData));
            }
        } else if (data instanceof RuleData) {
            RuleData ruleData = (RuleData) data;
            if (dataType == DataEventTypeEnum.UPDATE) {
                BaseDataCache.getInstance().cacheRuleData(ruleData);
                Optional.ofNullable(handlerMap.get(ruleData.getPluginName())).ifPresent(handler -> handler.handlerRule(ruleData));
            } else if (dataType == DataEventTypeEnum.DELETE) {
                BaseDataCache.getInstance().removeRuleData(ruleData);
                Optional.ofNullable(handlerMap.get(ruleData.getPluginName())).ifPresent(handler -> handler.removeRule(ruleData));
            }
        }
    });
}
```

```java
public void cachePluginData(final PluginData pluginData) {
    // 更新缓存
    Optional.ofNullable(pluginData).ifPresent(data -> PLUGIN_MAP.put(data.getName(), data));
}		
```

##### 3. 总结

*  以上是`Websocket` 同步数据的流程，其中在`soul-admin` 与 `soul-bootstrap` 启动时，会建立链接，全量同步一下。
*  如果不重启服务，则在`admin`后台修改数据，则会触发增量更新，其本质是修改本地缓存map中的值。


  收获

> ​	1.了解`Websocket` 
>
> ​	2.学习和体会了`soul` 源码中的一些设计思路，今后加以应用到工作中。

​		