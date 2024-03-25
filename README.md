

<p align="center">
	<img width="200" src="https://halo-1308808626.cos.ap-guangzhou.myqcloud.com/openchat/result%20logo.png" alt="Vue logo">
</p>

# 明日达物流 


## 项目简介

物流项目体系包含四个端：管理后台、用户端、司机端以及快递员端

该项目是物流类的项目，它对标顺丰，京东，向C端用户提供快递服务的系统。主要分为四个端，分别是：用户端、快递员端、司机端和后台系统管理。用户在微信小程序下单后，会生成订单并创建揽件任务，快递员取件后，用户对订单进行支付，会产生交易单，而后创建运单。快件开始运输。经过 三级营业部、二级分拣中心、一级转运中心的运输流转，生成多个运输单。最终到达网点，快递员派件。用户签收或者拒收。

## 项目技术栈

项目中使用的技术：SpringBoot、SpringCloud 、mybatis、mybatisPlus、mysql、mongodb、Neo4j、ES、Redis、nacos、seata、gateway、openfeign、rabbitmq、xxl-job、nginx、jwt、OSS等。

整体框架采用springcloud+springboot的微服务框架采用前后台分离的方式进行的微服务开发，其中我们使用到的组件有alibaba的nacos作为我们的注册中心和配置中心，使用gateway作为我们的网关，服务间的远程调用我们采用openfeign，还会涉及到分布式事务问题，采用seata框架解决。

使用mybtais作为持久层框架，对于复杂的查询我们采用的是sql的形式，对于一些简单的增加更改操作我们使用的是MybatisPlus，简化开发，同时我们使用mysql作为主要数据库，mongodb用于物流信息存储。使用Redis作为我们的缓存数据库，实现物流信息的快速加载。Neo4j进行路线规划。

使用rabbitmq中间件来进行数据的传递、解耦，和削峰的处理。使用xxl-job作为我们分布式任务调度框架。使用nginx作为代理服务器。

## 项目展示

![image-20240324192228205](https://halo-1308808626.cos.ap-guangzhou.myqcloud.com/images/202403241925501.png)

首页工作大屏

![image-20240324192550646](https://halo-1308808626.cos.ap-guangzhou.myqcloud.com/images/202403241925405.png)

![image-20240324192636424](https://halo-1308808626.cos.ap-guangzhou.myqcloud.com/images/202403241926556.png)

![mm](https://halo-1308808626.cos.ap-guangzhou.myqcloud.com/images/202403241930987.gif)

## 我的职责

**责任描述：主要负责调度中心、路线管理、轨迹管理等功能模块的设计与开发**

### 1、双 Token 三验证

**用户端采用security + jwt 设计双Token（access_token、refresh_token）三验证的方案，实现Token的无感刷新。将Token存入Redis中，实现用户单点登录。**

![image-20240324210934105](https://halo-1308808626.cos.ap-guangzhou.myqcloud.com/images/202403242109206.png)



>使用双 Token （access_token、refresh_token） 的 理由？
>
>- token 有效期设置过长，用户会频繁的登录。设置的过长，会不安全，一旦 token 被黑客截取的话，就可以使用此 token 与服务端进行交互了。因此，使用 refresh_token 为 access_token 续期。
>
>  1. 用户登录成功后，生成2个token，分别是 ： access_token、refresh_token，前者有效期短（5min）后者的有效期（24h）
>
>  2. 正常请求后端服务时，携带 access_token，如果发现 access_token失效，通过refresh_token 到后台服务中换取新的 access_token和 refresh_token
>
>- 另外，由于 token 是无状态的，服务端一旦颁发了token就无法让其失效（除非过了有效期），这样的话，如果我们检测到token异常也无法使其失效，所以这也是无状态token存在的问题。
>
>  1. 为了使后端可以控制其提前失效，将 refresh_token 设计成只能使用一次
>  2. 将 refresh_token 存储到 redis中，并且设置其过期时间
>  3. 检测到用户token 有安全隐患时，将 refresh_token 失效

### 2、运费模块

**负责用户下单后运费模块（==使用责任链模式选择不同运件收寄地的运费模板。编写相应的运费计算流程==）。**

```
运费是由运费模板进行定义的，不同的寄件收件城市，匹配的运费模板是不一样的
- 运费模板类型有：同城寄、省内寄、经济区互寄和跨省全国寄多种
一件快递，最终产生的运费，根据不同运费模板以及货物重量和体积进行计算的


在运费计算流程中，需要找到匹配的运费模板（对应的计算规则，才能进行最终运费的计算）

运费模版数据表也是我自己设计的，表中字段有：模版类型、运送类型、关联城市、首重、续重、轻抛系数等字段，其中轻抛系数指的是体积和重量的转换。在计算运费的时候，比较轻抛系数计算出来的值和货品本身重量，按照更大的值计算运费。
```

运费计算流程

```
用户在小程序端填写完毕（（寄件地址、收件地址、货物重量体积））后，需要预估一个价格；
还有快递员收件时，确认经过测量后的货物重量体积、订单信息后，也需要预估一个价格；

1、查找运费模板
用户将 参数（寄件地址、收件地址、货物重量体积）传入 快递员微服务中，根据 寄收件城市查询 计费模板（使用责任链模式进行匹配）并返回。运费模板中 保存了各种计费的项目，比如：首重价格多钱、续重价格多钱、轻抛系数，结合前端给到体积和重量。
2、计算重量
计算出货物的体积（cm^3），然后除以 运费模板中的 轻抛系数 得到 体积重量，在和前端传入的重量参数比对，哪个大，就按照 哪个 重量进行计费。
然后对 重量进行小数点处理：
- 小于一公斤，按照一公斤计算；
- 大于等于一公斤，小于十公斤，重量按照 0.1kg 四舍五入 保留1位小数（3.3 取 3 、3.6取 4）
- 大于等于 十公斤，小于一百公斤，重量按照 0.5 kg计算，四舍五入保留1位小数，18.1 取 18.5 ，18.6取19
- 大于一百公斤，直接四舍五入取整了
3、计算运费
根据运费计算公式： 运费 = 首重价格 + （实际重量 - 1） * 续重价格，计算出来运费以后，将运费模板、重量、价格这些信息封装为运费DTO实体类对象，返回给前端展示即可。
```



![image-20240324211758547](https://halo-1308808626.cos.ap-guangzhou.myqcloud.com/images/202403242118621.png)



### 3、支付模块

**负责 交易中台部分模块开发，实现物流运单交易等操作。实现支付宝沙箱支付、微信支付等操作；使用工厂模式动态获取相应处理类，完成第三方支付平台的异步回调通知、使用定时任务主动轮询等获取支付结果等操作；**

```
首先设计之初，为了实现支付模块的 独立于其他微服务（对接其他服务），以此来达到无论哪个业务服务发起的支付的，只负责值得交易订单，依托与 RabbitMQ 解耦合其他微服务；无论是订单服务调用支付服务还是支付服务异步通知订单服务，都是通过 MQ 来进行服务间的通信的；

用户提交寄件申请后，平台会分配任务，委派快递员上门取件。快递员重新确定货品数据，比如：重量、体积等，然后确认收件，最后进入到支付环节。（根据用户选择支付宝支付还是微信支付生成二维码给用户支付），由订单服务通过 MQ 给 支付模块发送消息，支付结果 通过 第三方平台 异步回调 和 分布式任务调度 XXL-JOB 主动查询得出结果，然后 更改订单的交易状态，通过 mq 更改用户前端订单状态
```

>说说订单创建流程？
>
>```
>用户下单：用户下单的时候，将快递信息（比如 重量、体积）等发送到订单微服务，生成订单
>快递员揽件：快递员进行揽件后，再次确认货物数据信息，然后在快递员端 将这笔订单转为运单；
>订单微服务：封装好订单 ID、基本信息、订单金额、支付方式、支付企业号、支付渠道等信息，然            后调用 支付微服务，创建交易单，将交易单信息保存在数据库中，发起第三方调用。
>
>```
>
>说说你的支付流程？（创建交易单的流程）
>
>创建交易单的流程涉及到 幂等性、分布式锁的处理，同时在获取支付方式的时候，涉及使用了工厂模式进行优化




**4、使用 RabbitMQ 延时队列 完成订单转运单逻辑操作；调度中心模块使用 Redis List、Set 数据结构处理运单分发;
5、设计使用 XXL-JOB 分布式任务调度实现定时自动分配运单生成运输任务，并使用分片广播任务调度策略，实现并行处理车辆安排，提升调度的处理效率。
6、为了提高物流追踪模块的查询效率，使用MongoDB存储物流追踪信息。另外通过 Redis + Caffieine 实现多级缓存。通过Redis的消息订阅发布模型解决集群环境下一级缓存与Redis缓存不一致的问题。**



