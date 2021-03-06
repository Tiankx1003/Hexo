---
title: Interview--项目架构
typora-copy-images-to: Interview--项目架构
tags:
 - BigData
 - Interview
---
云上数据仓库解决方案：https://www.aliyun.com/solution/datavexpo/datawarehouse
云上数据集成解决方案：https://www.aliyun.com/solution/datavexpo/cdp

# 数仓概念
#### 数据仓库的输入数据源和输出系统分别是什么
**输入系统** 埋点产生的用户行为数据、JavaEE后台产生的业务数据、个别公司有爬虫数据
**输出系统** 报表系统、用户画像系统、推荐系统

# 系统数据流程设计
<!-- TODO 手绘数仓架构图并讲述 -->

# 框架版本选型
**Apache** 运维比较麻烦，组建兼容性需要自己调研(一般大厂使用，技术实力雄厚且有专业的运维人员)
**CDH** 国内使用做多的版本，但CM不开源，但其实对中、小公司使用来说没有影响(建议使用)
**HDP** 开源，可以进行二次开发，但是没有CDH稳定，国内使用较少

**Apache框架具体型号**

| 产品          | 版本     |
| :------------ | :------- |
| Hadoop        | 2.7.2    |
| Flume         | 1.7.0    |
| Kafka         | 0.11.0.2 |
| Kafka Manager | 1.3.3.22 |
| Hive          | 1.2.1    |
| Sqoop         | 1.4.6    |
| MySQL         | 5.6.24   |
| Azkaban       | 2.5.0    |
| Java          | 1.8      |
| Zookeeper     | 3.4.10   |
| Presto        | 0.189    |

CDH框架版本 5.12.1

| 产品      | 版本  |
| :-------- | :---- |
| Hadoop    | 2.6.0 |
| Spark     | 1.6.0 |
| Flume     | 1.6.0 |
| Hive      | 1.1.0 |
| Sqoop     | 1.4.6 |
| Oozie     | 4.1.0 |
| Zookeeper | 3.4.5 |
| Impala    | 2.9.0 |

# 服务器选型
服务器使用物理机还是云主机？
1. 机器成本考虑：
   * 物理机：以128G内存，20核物理CPU，40线程，8THDD和2TSSD硬盘，单台报价4W出头，惠普品牌。需考虑托管服务器费用。一般物理机寿命5年左右。
   * 云主机，以阿里云为例，差不多相同配置，每年5W
2. 运维成本考虑：
   * 物理机：需要有专业的运维人员
   * 云主机：很多运维工作都由阿里云已经完成，运维相对较轻松


# 集群规模
### 集群规模的确定
 * 每天日活跃用户100万，每人一天平均100条: 100万*100条=10000万条
 * 每条日志1k左右，每天1亿条: 100000000 / 1024 / 1024 ≈ 100G
 * 半年内不扩容服务器来算: 100G * 180天 ≈ 18T
 * 保存3个副本: 18T * 3 = 54T
 * 预留20% ~ 30% Buff = 54T / 0.7 = 77T
 * 越8T * 10台服务器

### 数仓分层
服务器将近再扩容1~2倍
<!-- TODO 详述 -->

### 用户行为数据
 * 每天日活跃用户100万，每人一天平均100条: 100万*100条=10000万条
 * 每条日志1k左右，每天1亿条: 100000000 / 1024 / 1024 ≈ 100G
 * 数仓ODS层采用LZO + parquet存储: 100G压缩为10G
 * 数仓DWD层采用LZO + parquet存储: 10G左右
 * 数仓DWS层轻度聚合存储(为了快速运算，不压缩): 50G左右
 * 数仓ADS层数据量很小，可忽略不计
 * 保存3个副本: 70G * 3 = 210G
 * 半年内不扩容服务器来算: 210 * 180天 ≈ 37T
 * 预留20% ~ 30% Buff = 37T / 0.7 = 53T

### Kafa中数据
 * 每天约100G数据 * 2个副本 = 200G
 * 保存7天 * 200G = 1400G
 * 预留30%buf=1400G/0.7 ≈ 2T

### Flume中默认缓存的数据比较小
 * 暂时忽略不计

### 业务数据
 * 每天活跃用户100万，每天下单的用户10万，每人每天产生的业务数据10条，每条日志1k左右: 10万 * 10条 * 1k ≈ 1G
 * 数仓分层存储: 1G * 3 = 3G
 * 保存是三个副本: 3G * 3 = 9G
 * 半年内不扩容服务器: 9G * 180天 ≈ 1.6T
 * 预留20%~30%Buf = 1.6T / 0.7 = 2T

### 集群总规模
53T + 2T + 2T = 57T

### 集群规模总结
约8T * 10台服务器


| 1     | 2     | 3     | 4     | 5     | 6     | 7     | 8     | 9     | 10    |
| ----- | ----- | ----- | ----- | ----- | ----- | ----- | ----- | ----- | ----- |
| nn    | nn    | dn    | dn    | dn    | dn    | dn    | dn    | dn    | dn    |
|       |       | rm    | rm    | nm    | nm    | nm    | nm    | nm    | nm    |
|       |       | nm    | nm    |       |       |       |       |       |       |
|       |       |       |       |       |       |       | zk    | zk    | zk    |
|       |       |       |       |       |       |       | kafka | kafka | kafka |
|       |       |       |       |       |       |       | Flume | Flume | flume |
|       |       |       |       |       | flume | flume |       |       |       |
|       |       | Hbase | Hbase | Hbase |       |       |       |       |       |
| hive  | hive  |       |       |       |       |       |       |       |       |
| mysql | mysql |       |       |       |       |       |       |       |       |
| spark | spark | spark | spark | spark | spark | spark | spark | spark | spark |
|       |       |       |       |       | ES    | ES    |       |       |       |

# 人员配置参考
### 整体架构
属于研发部，技术总监下面有各个项目组，我们属于大数据组，其他还有后端项目组，前端组、测试组等。总监上面就是副总等级别了。其他的还有产品运营部等。

### 部门的职责等级、晋升规则
职级就分初级，中级，高级。晋升规则不一定，看公司效益和职位空缺。
京东：T1、T2应届生；T3 14k左右   T4 18K左右  T5  24k左右
阿里：p5、p6、p7、p8


### 人员配置参考
小型公司（3人左右）：组长1人，剩余组员无明确分工，并且可能兼顾javaEE和前端。
中小型公司（3~6人左右）：组长1人，离线2人左右，实时1人左右（离线一般多于实时），组长兼顾和javaEE、前端。
中型公司（5~10人左右）：组长1人，离线3~5人左右（离线处理、数仓），实时2人左右，组长和技术大牛兼顾和javaEE、前端。。
中大型公司（5~20人左右）：组长1人，离线5~10人（离线处理、数仓），实时5人左右，JavaEE1人左右（负责对接JavaEE业务），前端1人（有或者没有人单独负责前端）。（发展比较良好的中大型公司可能大数据部门已经细化拆分，分成多个大数据组，分别负责不同业务）
上面只是参考配置，因为公司之间差异很大，例如ofo大数据部门只有5个人左右，因此根据所选公司规模确定一个合理范围，在面试前必须将这个人员配置考虑清楚，回答时要非常确定。

