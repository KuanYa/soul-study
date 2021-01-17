### soul网关环境的搭建

#### 一、什么是soul网关

* 深入学习`soul`前，了解一下，什么是`soul`网关，官网解释如下：

  * 这是一个异步的,高性能的,跨语言的,响应式的API网关。我希望能够有一样东西像灵魂一样，保护您的微服务。参考了Kong，Spring-Cloud-Gateway等优秀的网关后，站在巨人的肩膀上，Soul由此诞生！

* 特性
  * 支持各种语言(http协议)，支持 dubbo，springcloud协议。
  * 插件化设计思想，插件热插拔,易扩展。
  * 灵活的流量筛选，能满足各种流量控制。
  * 内置丰富的插件支持，鉴权，限流，熔断，防火墙等等。
  * 流量配置动态化，性能极高，网关消耗在 1~2ms。
  * 支持集群部署，支持 A/B Test, 蓝绿发布。

* [官方文档](https://shimo.im/docs/QyRXW8VtGTT3JGjc)

* 以上为`soul`的简单介绍，详细内容参考官方文档

#### 二、编译`soul`代码

* 拉取工程

  ````
  # 先 fork
  # 再 git clone https://github.com/KuanYa/soul.git
  # idea 打开
  ````

* 编辑代码

  ```
  # 命令如下
  mvn clean package install -Dmaven.examples.skip=true -Dmaven.javadoc.skip=true -Drat.skip=true -Dcheckstyle.skip=true
  ```

  ![image-20210115001730210](https://github.com/KuanYa/soul-study/blob/main/day01/pictures/image-20210115001730210.png)

  * **注意**
    * win10系统在执行 上面`mvn`命令时，需要使用`cmd`进入 `soul`目录下执行，`PowerShell`需要对 `-Dmaven.examples.skip=true` 等添加引号。

* 总结：通过跳过相关的模块后，可以很大程度提供编译速度

#### 三、启动模块

* 启动`admin`模块

  * 修改数据库配置文件 `application.yml`,支持`mysql` 与 `h2`

  * 运行`SoulAdminBootstrap`

    启动成功后，访问 http://localhost:9095 用户名：`admin`,密码 :`123456`

    ![image-20210115002956235](D:\Java-training-camp\soul-study\day01\pictures\image-20210115002956235.png)

* 启动`bootstrap`模块

  * 运行`SoulBootstrapApplication` 即可

    ```
    2021-01-15 00:45:47.444  INFO 30144 --- [ocket-connect-1] o.d.s.p.s.d.w.WebsocketSyncDataService   : websocket reconnect is successful.....
    ```

* 总结：经过上述的操作，已经将 `admin` 与 `bootstrap`模块启动完成，下面我们了解具体原理 

* 使用自定义的`选择器`与`规则`

  * [参考文档](https://dromara.org/zh-cn/docs/soul/selector.html)

  * 选择器如下

    * ![image-20210115015359238](D:\Java-training-camp\soul-study\day01\pictures\image-20210115015359238.png)

  * 规则如下

    * ![image-20210115015431109](D:\Java-training-camp\soul-study\day01\pictures\image-20210115015431109.png)

  * 配置完成后，需要启动业务服务来接入网关，我们以`soul-examples-http`为例

    * 首先启动

      ```yaml
      server:
        port: 8188 # 选择器中会配置此业务系统的ip port
        address: 0.0.0.0
      
      
      soul:
        http:
          adminUrl: http://localhost:9095
          port: 8188
          contextPath: /http # contextPath 业务系统接入网关时，访问时需要
          appName: http
          full: false
      
      logging:
        level:
          root: info
          org.springframework.boot: info
          org.apache.ibatis: info
          org.dromara.soul.test.bonuspoint: info
          org.dromara.soul.test.lottery: debug
          org.dromara.soul.test: debug
      ```

  * 验证业务系统是否接入网关成功

    * Controller 如下

      ```java
      @RestController
      @RequestMapping("/order")
      @SoulSpringMvcClient(path = "/order")
      public class OrderController {
          /**
           * Find by id order dto.
           *
           * @param id the id
           * @return the order dto
           */
          @GetMapping("/findById")
          @SoulSpringMvcClient(path = "/findById", desc = "Find by id")
          public OrderDTO findById(@RequestParam("id") final String id) {
              OrderDTO orderDTO = new OrderDTO();
              orderDTO.setId(id);
              orderDTO.setName("hello world findById");
              return orderDTO;
          }
      }
      ```

      那么，我在访问我的业务系统时,可以不直接指定我业务系统的地址，则执行网关的地址，如下：http://localhost:9195/http/order/findById?id=1

      ![image-20210115020131107](D:\Java-training-camp\soul-study\day01\pictures\image-20210115020131107.png)

      ```java
      2021-01-15 02:01:42.639  INFO 30144 --- [work-threads-13] o.d.soul.plugin.base.AbstractSoulPlugin  : divide selector success match , selector name :/http
      2021-01-15 02:01:42.639  INFO 30144 --- [work-threads-13] o.d.soul.plugin.base.AbstractSoulPlugin  : divide selector success match , selector name :/http/order/findById
      2021-01-15 02:01:42.639  INFO 30144 --- [work-threads-13] o.d.s.plugin.httpclient.WebClientPlugin  : The request urlPath is http://192.168.56.1:8188/order/findById?id=1, retryTimes is 0
      2021-01-15 02:01:42.670 ERROR 30144 --- [work-threads-14] o.d.soul.plugin.base.utils.CheckUtils    : can not match selector data: divide
      ```

  * 总结

    * 经过上述的配置，将业务系统接入网关已经完成，从而实现流量的控制。

### 四、总结

* `soul-admin` 后台管理可以自定义`选择器`与`规则`
  * `选择器`：相当于流量过滤的第一道网
  * `规则`：流量的最终匹配
* `soul-bootstrap` 网关的核心组件，流量的转发控制

* 今天是学习`soul`网关的第一天，通过参考官网文档等资料，将`soul-admin`、`soul-bootstrap`启动成功，同时，通过将业务系统接入网关中，

  初步了解了网关的大致工作流程，为后面深入学习及源码分析打下基础。

