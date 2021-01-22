#### 数据同步之`zookeeper`

##### １.什么是`zookeeper`

* `ZooKeeper` 是一个分布式的，开放源码的分布式应用程序协同服务。

  `ZooKeeper` 的设计目标是将那些复杂且容易出错的分布式一致性服务封装起来，

  构成一个高效可靠的原语集，并以一系列简单易用的接口提供给用户使用

* **典型应用场景**

  * 配置管理（configurationmanagement） 

  * DNS服务

  * 组成员管理（groupmembership） 

  *  各种分布式锁

  > Zookeepers 适用于存储和协同相关的关键数据，不适合用于大数据量存储

##### 2.`zookeeper`在soul的使用方式 

* 依赖 `zookeeper` 的 watch 机制。

* `soul-admin`启动时会将数据全量写入到`zookeeper `中，后续存在数据更新，则增量更新`zookeeper `节点的数据。

* `soul-web`会监听节点信息的变化，同步更新到本地缓存中。

##### 3.源码解析

* 启动`zookeeper`数据同步机制

  `soul-admin` 下   `application.yml`

  ```yaml
  soul:
    database:
      dialect: mysql
      init_script: "META-INF/schema.sql"
    sync:
      websocket:
        enabled: false # 由于soul默认数据同步机制是 Websocket 所以需要手动关闭，排除干扰
        zookeeper:
            url: localhost:2181
            sessionTimeout: 5000
            connectionTimeout: 2000
  ```

  `soul-bootstrap` 下 `application-local.yml`

  ```yml
  soul :
      file:
        enabled: true
      corss:
        enabled: true
      dubbo :
        parameter: multi
      sync:
          #websocket :
               #urls: ws://localhost:9095/websocket
  
          zookeeper:
               url: localhost:2181
               sessionTimeout: 5000
               connectionTimeout: 2000
  ```

  

* 启动`soul-admin` 启动时创建 `zk` 客户端

  ```java
  @Bean
  @ConditionalOnMissingBean(ZkClient.class)
  public ZkClient zkClient(final ZookeeperProperties zookeeperProp) {
      return new ZkClient(zookeeperProp.getUrl(), zookeeperProp.getSessionTimeout(), zookeeperProp.getConnectionTimeout());
  }
  ```

  启动中遇到一个问题，`soul-admin`在启动时，应该会创建`zk` 监听，但是没有执行

  ```java
  /**
   * The type Zookeeper listener.
   */
  @Configuration
  @ConditionalOnProperty(prefix = "soul.sync.zookeeper", name = "url")
  @Import(ZookeeperConfiguration.class)
  static class ZookeeperListener {
  
      /**
       * Config event listener data changed listener.
       *
       * @param zkClient the zk client
       * @return the data changed listener
       */
      @Bean
      @ConditionalOnMissingBean(ZookeeperDataChangedListener.class)
      public DataChangedListener zookeeperDataChangedListener(final ZkClient zkClient) {
          // 此处没有执行
          return new ZookeeperDataChangedListener(zkClient);
      }
  
      /**
       * Zookeeper data init zookeeper data init.
       *
       * @param zkClient        the zk client
       * @param syncDataService the sync data service
       * @return the zookeeper data init
       */
      @Bean
      @ConditionalOnMissingBean(ZookeeperDataInit.class)
      public ZookeeperDataInit zookeeperDataInit(final ZkClient zkClient, final SyncDataService syncDataService) {
          // 此处没有执行
          return new ZookeeperDataInit(zkClient, syncDataService);
      }
  }
  ```

  问题原因：
  
  具体问题还未找到，明天还要加班，下班后继续分析，今天就先水一起，昨天的周末一并补上