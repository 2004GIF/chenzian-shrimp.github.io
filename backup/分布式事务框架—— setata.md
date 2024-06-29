# 分布式事务框架—— setata
## 为什么要使用分布式事务框架：

  业务场景： 

![image-20240629182546354](https://github.com/2004GIF/chenzian-shrimp.github.io/assets/126451952/8790679a-6ba1-45c3-882c-fd84121a1f40)
代码实现：

````java
   @Override
    @GlobalTransactional // 开启全局事务管理器
    public Long createOrder(OrderFormDTO orderFormDTO) {
        // 1.订单数据
        Order order = new Order();
        // 1.1.查询商品
        List<OrderDetailDTO> detailDTOS = orderFormDTO.getDetails();
        // 1.2.获取商品id和数量的Map
        Map<Long, Integer> itemNumMap = detailDTOS.stream()
                .collect(Collectors.toMap(OrderDetailDTO::getItemId, OrderDetailDTO::getNum));
        Set<Long> itemIds = itemNumMap.keySet();
        // 1.3.查询商品
        List<ItemDTO> items = itemClient.queryItemsByIds(itemIds);

        if (items == null || items.size() < itemIds.size()) {
            throw new BadRequestException("商品不存在");
        }
        // 1.4.基于商品价格、购买数量计算商品总价：totalFee
        int total = 0;
        for (ItemDTO item : items) {
            total += item.getPrice() * itemNumMap.get(item.getId());
        }
        order.setTotalFee(total);
        // 1.5.其它属性
        order.setPaymentType(orderFormDTO.getPaymentType());
        order.setUserId(UserContext.getUser()); //   登录用户id
        order.setStatus(1);
        // 1.6.将Order写入数据库order表中
        save(order);

        // 2.保存订单详情
        List<OrderDetail> details = buildDetails(order.getId(), items, itemNumMap);
        detailService.saveBatch(details);

        // 3.清理购物车商品
        cartClient.deleteCartItemByIds(itemIds);

        // 4.扣减库存
        try {
            itemClient.deductStock(detailDTOS);
        } catch (Exception e) {
            throw new RuntimeException("库存不足！");
        }
        return order.getId();
    }
````

出现问题：清空购物车的业务完成，SQL 语句执行完成已经提交了，到了扣减商品库存的业务出错了，应该整个创建订单的业务都失败了，失败就应该回滚恢复原来的数据，但是清空购物车的业务的SQL语句已经提交不可能回滚了。这就导致数据不一致的问题。

事务并未遵循ACID的原则，归其原因就是参与事务的多个子业务在不同的微服务，跨越了不同的数据库。虽然每个单独的业务都能在本地遵循ACID，但是它们互相之间没有感知，不知道有人失败了，无法保证最终结果的统一，也就无法遵循ACID的事务特性了。

这就是分布式事务问题，出现以下情况之一就可能产生分布式事务问题：

- 业务跨多个服务实现
- 业务跨多个数据源实现


## 解决方案利用 seata 框架来进行分布式事务

解决分布式事务的方案有很多，但实现起来都比较复杂，因此我们一般会使用开源的框架来解决分布式事务问题。在众多的开源分布式事务框架中，功能最完善、使用最多的就是阿里巴巴在2019年开源的Seata了。

https://seata.io/zh-cn/docs/overview/what-is-seata.html

其实分布式事务产生的一个重要原因，就是参与事务的多个分支事务互相无感知，不知道彼此的执行状态。因此解决分布式事务的思想非常简单：

就是找一个统一的**事务协调者**，与多个分支事务通信，检测每个分支事务的执行状态，保证全局事务下的每一个分支事务同时成功或失败即可。大多数的分布式事务框架都是基于这个理论来实现的。

Seata也不例外，在Seata的事务管理中有三个重要的角色：

-  **TC** **(****Transaction Coordinator****) -** **事务协调者：**维护全局和分支事务的状态，协调全局事务提交或回滚。 
-  **TM (Transaction Manager) -** **事务管理器：**定义全局事务的范围、开始全局事务、提交或回滚全局事务。 
-  **RM (Resource Manager) -** **资源管理器：**管理分支事务，与TC交谈以注册分支事务和报告分支事务的状态，并驱动分支事务提交或回滚。 

Seata的工作架构如图所示：

![image-20240629145542457](https://github.com/2004GIF/chenzian-shrimp.github.io/assets/126451952/942caf27-5dd2-41f4-a630-82167edb950d)


其中，**TM**和**RM**可以理解为Seata的客户端部分，引入到参与事务的微服务依赖中即可。将来**TM**和**RM**就会协助微服务，实现本地分支事务与**TC**之间交互，实现事务的提交或回滚。

而**TC**服务则是事务协调中心，是一个独立的微服务，需要单独部署,部署参考：  https://b11et3un53m.feishu.cn/wiki/QfVrw3sZvihmnPkmALYcUHIDnff
### 使用 seata 步骤

以下步骤给每一个需要分布式事务的微服 （ 参与分布式事务的每一个微服务都需要集成Seata）

1. 导入坐标

   ````xml
     <!--seata-->
     <dependency>
         <groupId>com.alibaba.cloud</groupId>
         <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
     </dependency>
   ````

2. 在每一个需要分布式事务的微服务中

   ````yaml
   seata:
     registry: # TC服务注册中心的配置，微服务根据这些信息去注册中心获取tc服务地址
       type: nacos # 注册中心类型 nacos
       nacos:
         server-addr: 192.168.150.101:8848 # nacos地址
         namespace: "" # namespace，默认为空
         group: DEFAULT_GROUP # 分组，默认是DEFAULT_GROUP
         application: seata-server # seata服务名称
         username: nacos
         password: nacos
     tx-service-group: hmall # 事务组名称
     service:
       vgroup-mapping: # 事务组与tc集群的映射关系
         hmall: "default"
   ````

3. 选择解决分布式事务的模式

​	Seata支持四种不同的分布式事务解决方案：

- **XA**
- **TCC**
- **AT**
- **SAGA**

这里我们以`XA`模式和`AT`模式来给大家讲解其实现原理。

`XA` 规范 是` X/Open` 组织定义的分布式事务处理（DTP，Distributed Transaction Processing）标准，XA 规范 描述了全局的`TM`与局部的`RM`之间的接口，几乎所有主流的数据库都对 XA 规范 提供了支持。

### 2.4.1.两阶段提交

XA是规范，目前主流数据库都实现了这种规范，实现的原理都是基于两阶段提交。

正常情况：

![img](https://b11et3un53m.feishu.cn/space/api/box/stream/download/asynccode/?code=ODY2YjZhZjg5ODgwMzYxMTcxM2Q2MWI2NzgyZjBiODlfSGt5cFlVYjlYTXhuVlpESXJQcDRBWG9TVUIyYmlHT2NfVG9rZW46RkZhSmJZOVByb1V1dmN4c0NLd2MweHpNbno4XzE3MTk2NTc2ODc6MTcxOTY2MTI4N19WNA)

异常情况：

![img](https://b11et3un53m.feishu.cn/space/api/box/stream/download/asynccode/?code=MzY5MzIxM2I0ZDA3MGVmYTMzNDExMDkxZDE5YTQ3YjhfQVM1TUdpVnpyMndZTlpVdHpiMDBZUmV3SVJNelJRZDNfVG9rZW46R0RYcGJKS3BWb3kzRGV4MGdOTGNlWUdDbmlmXzE3MTk2NTc2ODc6MTcxOTY2MTI4N19WNA)

一阶段：

- 事务协调者通知每个事务参与者执行本地事务
- 本地事务执行完成后报告事务执行状态给事务协调者，此时事务不提交，继续持有数据库锁

二阶段：

- 事务协调者基于一阶段的报告来判断下一步操作
- 如果一阶段都成功，则通知所有事务参与者，提交事务
- 如果一阶段任意一个参与者失败，则通知所有事务参与者回滚事务

### 2.4.2.Seata的XA模型

Seata对原始的XA模式做了简单的封装和改造，以适应自己的事务模型，基本架构如图：

![img](https://b11et3un53m.feishu.cn/space/api/box/stream/download/asynccode/?code=NzJhZTkwNGU2MjQxMDQyZjQwMGZmYmIzZjdjN2M2Y2Nfb0pTVzdUUXB1SzhlSGNYRktHMzhVZ3FBVVp0Z3pqWGZfVG9rZW46VmFCemI3cnMwb24yQ2N4V3JPVmNNNlhZbjNnXzE3MTk2NTc2ODc6MTcxOTY2MTI4N19WNA)

`RM`一阶段的工作：

1. 注册分支事务到`TC`
2. 执行分支业务sql但不提交
3. 报告执行状态到`TC`

`TC`二阶段的工作：

1.  `TC`检测各分支事务执行状态
   1. 如果都成功，通知所有RM提交事务
   2. 如果有失败，通知所有RM回滚事务 

`RM`二阶段的工作：

- 接收`TC`指令，提交或回滚事务

### 2.4.3.优缺点

`XA`模式的优点是什么？

- 事务的强一致性，满足ACID原则
- 常用数据库都支持，实现简单，并且没有代码侵入

`XA`模式的缺点是什么？

- 因为一阶段需要锁定数据库资源，等待二阶段结束才释放，性能较差
- 依赖关系型数据库实现事务

### 2.4.4.实现步骤

首先，我们要在配置文件中指定要采用的分布式事务模式。我们可以在Nacos中的共享shared-seata.yaml配置文件中设置：

```YAML
seata:
  data-source-proxy-mode: XA
```

其次，我们要利用`@GlobalTransactional`标记分布式事务的入口方法：

![img](https://b11et3un53m.feishu.cn/space/api/box/stream/download/asynccode/?code=ZTdiY2U0YTc4ODI4YTNkZWY1NGI0N2M1YWYwZmRjNmJfemE1N1dzcW82WFNRa203UkRTbjBIY1pZdnVHWlIwVG1fVG9rZW46VW1NMmJ6RWZQb2hadEh4bEM0a2NPa1M2blZoXzE3MTk2NTc2ODc6MTcxOTY2MTI4N19WNA)

## 2.5.AT模式

`AT`模式同样是分阶段提交的事务模型，不过缺弥补了`XA`模型中资源锁定周期过长的缺陷。

### 2.5.1.Seata的AT模型

基本流程图：

![img](https://b11et3un53m.feishu.cn/space/api/box/stream/download/asynccode/?code=MGFhMTYwYTE3NDFlZjNlNTUzMTNhZGNkMzI0YjdjYTdfUWhsUDV6ZUpwdlVhaDZzTTZ3NVJpVk9ycG1VQmxOSHVfVG9rZW46Q1hRcWJ1SUN2b0I2YVp4MUFSM2NkS0tPbk5iXzE3MTk2NTc2ODc6MTcxOTY2MTI4N19WNA)

阶段一`RM`的工作：

- 注册分支事务
- 记录undo-log（数据快照）
- 执行业务sql并提交
- 报告事务状态

阶段二提交时`RM`的工作：

- 删除undo-log即可

阶段二回滚时`RM`的工作：

- 根据undo-log恢复数据到更新前

### 2.5.3.AT与XA的区别

简述`AT`模式与`XA`模式最大的区别是什么？

- `XA`模式一阶段不提交事务，锁定资源；`AT`模式一阶段直接提交，不锁定资源。
- `XA`模式依赖数据库机制实现回滚；`AT`模式利用数据快照实现数据回滚。
- `XA`模式强一致；`AT`模式最终一致

可见，AT模式使用起来更加简单，无业务侵入，性能更好。因此企业90%的分布式事务都可以用AT模式来解决。





