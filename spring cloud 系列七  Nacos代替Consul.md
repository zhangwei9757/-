# Spring Cloud 微服务系列七： Nacos代替Consul



>  **提示：本文默认您已使用 注册中心 或 已知晓微服务注册中心组件的使用**

## 1.  Nacos 介绍

>  官网： https://nacos.io/zh-cn/docs/what-is-nacos.html 
>
>  Nacos 致力于帮助您发现、配置和管理微服务。Nacos 提供了一组简单易用的特性集，帮助您快速实现动态服务发现、服务配置、服务元数据及流量管理。
>
>  Nacos 帮助您更敏捷和容易地构建、交付和管理微服务平台。 Nacos 是构建以“服务”为中心的现代应用架构 (例如微服务范式、云原生范式) 的服务基础设施。

![nacosMap](https://nacos.io/img/nacosMap.jpg)

![1533045871534-e64b8031-008c-4dfc-b6e8-12a597a003fb](https://cdn.nlark.com/lark/0/2018/png/11189/1533045871534-e64b8031-008c-4dfc-b6e8-12a597a003fb.png)



## 2.   Nacos 服务端集群部署 + 密码配置化 



### 1. 下载启动包

> 下载网址：  https://github.com/alibaba/nacos/releases 
>
> [nacos-server-1.3.2.tar.gz](https://github.com/alibaba/nacos/releases/download/1.3.2/nacos-server-1.3.2.tar.gz)71.3 MB
>
> [nacos-server-1.3.2.zip](https://github.com/alibaba/nacos/releases/download/1.3.2/nacos-server-1.3.2.zip)71.3 MB
>
> [Source code(zip)](https://github.com/alibaba/nacos/archive/1.3.2.zip)
>
> [Source code(tar.gz)](https://github.com/alibaba/nacos/archive/1.3.2.tar.gz)
>
> 目前稳定版是1.1.4，但是可能下载特别慢，这里提供网盘下载链接(nacos-server-1.1.4.tar.gz)：
>
> 链接:https://pan.baidu.com/s/1yQqAmLG8wVVWpBjE0yXMHw 
>
> 提取码:z9b3



### 2. 解压指定环境的压缩包

> nacos-server-版本.zip
>
> nacos-server-版本.tar.gz



### 3. 启动

> startup.cmd -m standalone   (standalone代表着单机模式运行，非集群模式): 



### 4. 访问

> 测试地址  http://127.0.0.1:8848/nacos  成功进入表示环境搭建成功
>
> 输入默认用户名密码： nacos/nacos 



### 5.  集群部署

#### 5.1 nacos\conf\ 下有一个文件  nacos-mysql.sql 

``` sql
CREATE DATABASE nacos;
USE nacos;

/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = config_info   */
/******************************************/
CREATE TABLE `config_info` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
  `data_id` varchar(255) NOT NULL COMMENT 'data_id',
  `group_id` varchar(255) DEFAULT NULL,
  `content` longtext NOT NULL COMMENT 'content',
  `md5` varchar(32) DEFAULT NULL COMMENT 'md5',
  `gmt_create` datetime NOT NULL DEFAULT '2010-05-05 00:00:00' COMMENT '创建时间',
  `gmt_modified` datetime NOT NULL DEFAULT '2010-05-05 00:00:00' COMMENT '修改时间',
  `src_user` text COMMENT 'source user',
  `src_ip` varchar(20) DEFAULT NULL COMMENT 'source ip',
  `app_name` varchar(128) DEFAULT NULL,
  `tenant_id` varchar(128) DEFAULT '' COMMENT '租户字段',
  `c_desc` varchar(256) DEFAULT NULL,
  `c_use` varchar(64) DEFAULT NULL,
  `effect` varchar(64) DEFAULT NULL,
  `type` varchar(64) DEFAULT NULL,
  `c_schema` text,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_configinfo_datagrouptenant` (`data_id`,`group_id`,`tenant_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='config_info';

/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = config_info_aggr   */
/******************************************/
CREATE TABLE `config_info_aggr` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
  `data_id` varchar(255) NOT NULL COMMENT 'data_id',
  `group_id` varchar(255) NOT NULL COMMENT 'group_id',
  `datum_id` varchar(255) NOT NULL COMMENT 'datum_id',
  `content` longtext NOT NULL COMMENT '内容',
  `gmt_modified` datetime NOT NULL COMMENT '修改时间',
  `app_name` varchar(128) DEFAULT NULL,
  `tenant_id` varchar(128) DEFAULT '' COMMENT '租户字段',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_configinfoaggr_datagrouptenantdatum` (`data_id`,`group_id`,`tenant_id`,`datum_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='增加租户字段';


/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = config_info_beta   */
/******************************************/
CREATE TABLE `config_info_beta` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
  `data_id` varchar(255) NOT NULL COMMENT 'data_id',
  `group_id` varchar(128) NOT NULL COMMENT 'group_id',
  `app_name` varchar(128) DEFAULT NULL COMMENT 'app_name',
  `content` longtext NOT NULL COMMENT 'content',
  `beta_ips` varchar(1024) DEFAULT NULL COMMENT 'betaIps',
  `md5` varchar(32) DEFAULT NULL COMMENT 'md5',
  `gmt_create` datetime NOT NULL DEFAULT '2010-05-05 00:00:00' COMMENT '创建时间',
  `gmt_modified` datetime NOT NULL DEFAULT '2010-05-05 00:00:00' COMMENT '修改时间',
  `src_user` text COMMENT 'source user',
  `src_ip` varchar(20) DEFAULT NULL COMMENT 'source ip',
  `tenant_id` varchar(128) DEFAULT '' COMMENT '租户字段',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_configinfobeta_datagrouptenant` (`data_id`,`group_id`,`tenant_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='config_info_beta';

/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = config_info_tag   */
/******************************************/
CREATE TABLE `config_info_tag` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
  `data_id` varchar(255) NOT NULL COMMENT 'data_id',
  `group_id` varchar(128) NOT NULL COMMENT 'group_id',
  `tenant_id` varchar(128) DEFAULT '' COMMENT 'tenant_id',
  `tag_id` varchar(128) NOT NULL COMMENT 'tag_id',
  `app_name` varchar(128) DEFAULT NULL COMMENT 'app_name',
  `content` longtext NOT NULL COMMENT 'content',
  `md5` varchar(32) DEFAULT NULL COMMENT 'md5',
  `gmt_create` datetime NOT NULL DEFAULT '2010-05-05 00:00:00' COMMENT '创建时间',
  `gmt_modified` datetime NOT NULL DEFAULT '2010-05-05 00:00:00' COMMENT '修改时间',
  `src_user` text COMMENT 'source user',
  `src_ip` varchar(20) DEFAULT NULL COMMENT 'source ip',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_configinfotag_datagrouptenanttag` (`data_id`,`group_id`,`tenant_id`,`tag_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='config_info_tag';

/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = config_tags_relation   */
/******************************************/
CREATE TABLE `config_tags_relation` (
  `id` bigint(20) NOT NULL COMMENT 'id',
  `tag_name` varchar(128) NOT NULL COMMENT 'tag_name',
  `tag_type` varchar(64) DEFAULT NULL COMMENT 'tag_type',
  `data_id` varchar(255) NOT NULL COMMENT 'data_id',
  `group_id` varchar(128) NOT NULL COMMENT 'group_id',
  `tenant_id` varchar(128) DEFAULT '' COMMENT 'tenant_id',
  `nid` bigint(20) NOT NULL AUTO_INCREMENT,
  PRIMARY KEY (`nid`),
  UNIQUE KEY `uk_configtagrelation_configidtag` (`id`,`tag_name`,`tag_type`),
  KEY `idx_tenant_id` (`tenant_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='config_tag_relation';

/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = group_capacity   */
/******************************************/
CREATE TABLE `group_capacity` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `group_id` varchar(128) NOT NULL DEFAULT '' COMMENT 'Group ID，空字符表示整个集群',
  `quota` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '配额，0表示使用默认值',
  `usage` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '使用量',
  `max_size` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '单个配置大小上限，单位为字节，0表示使用默认值',
  `max_aggr_count` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '聚合子配置最大个数，，0表示使用默认值',
  `max_aggr_size` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '单个聚合数据的子配置大小上限，单位为字节，0表示使用默认值',
  `max_history_count` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '最大变更历史数量',
  `gmt_create` datetime NOT NULL DEFAULT '2010-05-05 00:00:00' COMMENT '创建时间',
  `gmt_modified` datetime NOT NULL DEFAULT '2010-05-05 00:00:00' COMMENT '修改时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_group_id` (`group_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='集群、各Group容量信息表';

/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = his_config_info   */
/******************************************/
CREATE TABLE `his_config_info` (
  `id` bigint(64) unsigned NOT NULL,
  `nid` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `data_id` varchar(255) NOT NULL,
  `group_id` varchar(128) NOT NULL,
  `app_name` varchar(128) DEFAULT NULL COMMENT 'app_name',
  `content` longtext NOT NULL,
  `md5` varchar(32) DEFAULT NULL,
  `gmt_create` datetime NOT NULL DEFAULT '2010-05-05 00:00:00',
  `gmt_modified` datetime NOT NULL DEFAULT '2010-05-05 00:00:00',
  `src_user` text,
  `src_ip` varchar(20) DEFAULT NULL,
  `op_type` char(10) DEFAULT NULL,
  `tenant_id` varchar(128) DEFAULT '' COMMENT '租户字段',
  PRIMARY KEY (`nid`),
  KEY `idx_gmt_create` (`gmt_create`),
  KEY `idx_gmt_modified` (`gmt_modified`),
  KEY `idx_did` (`data_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='多租户改造';


/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = tenant_capacity   */
/******************************************/
CREATE TABLE `tenant_capacity` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `tenant_id` varchar(128) NOT NULL DEFAULT '' COMMENT 'Tenant ID',
  `quota` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '配额，0表示使用默认值',
  `usage` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '使用量',
  `max_size` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '单个配置大小上限，单位为字节，0表示使用默认值',
  `max_aggr_count` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '聚合子配置最大个数',
  `max_aggr_size` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '单个聚合数据的子配置大小上限，单位为字节，0表示使用默认值',
  `max_history_count` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '最大变更历史数量',
  `gmt_create` datetime NOT NULL DEFAULT '2010-05-05 00:00:00' COMMENT '创建时间',
  `gmt_modified` datetime NOT NULL DEFAULT '2010-05-05 00:00:00' COMMENT '修改时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_tenant_id` (`tenant_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='租户容量信息表';


CREATE TABLE `tenant_info` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
  `kp` varchar(128) NOT NULL COMMENT 'kp',
  `tenant_id` varchar(128) default '' COMMENT 'tenant_id',
  `tenant_name` varchar(128) default '' COMMENT 'tenant_name',
  `tenant_desc` varchar(256) DEFAULT NULL COMMENT 'tenant_desc',
  `create_source` varchar(32) DEFAULT NULL COMMENT 'create_source',
  `gmt_create` bigint(20) NOT NULL COMMENT '创建时间',
  `gmt_modified` bigint(20) NOT NULL COMMENT '修改时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_tenant_info_kptenantid` (`kp`,`tenant_id`),
  KEY `idx_tenant_id` (`tenant_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='tenant_info';

CREATE TABLE users (
	username varchar(50) NOT NULL PRIMARY KEY,
	password varchar(500) NOT NULL,
	enabled boolean NOT NULL
);

CREATE TABLE roles (
	username varchar(50) NOT NULL,
	role varchar(50) NOT NULL
);

/*****************************************************************/
/*   此处是使用spring security 的 BCryptPasswordEncoder 加密方式   */
/*****************************************************************/
INSERT INTO users (username, password, enabled) VALUES ('nacos', '$2a$10$EuWPZHzz32dJN7jexM34MOeYirDdFAZm2kuWj7VEOJhhZkDrxfvUu', TRUE);

INSERT INTO roles (username, role) VALUES ('nacos', 'ROLE_ADMIN');
```



> 查询结果如下:
>
> username  password                                                      enabled  
> --------  ------------------------------------------------------------  ---------
> nacos     $2a$10$EuWPZHzz32dJN7jexM34MOeYirDdFAZm2kuWj7VEOJhhZkDrxfvUu          1



#### 5.2  nacos\conf\ 下有一个文件   nacos/conf/application.properties  

添加如下配置：(配置成功后重启即可，此时修改密码也可永久生效)

```pro
spring.datasource.platform=mysql
db.num=1
db.url.0=jdbc:mysql://localhost:3306/nacos?useSSL=false&serverTimezone=UTC&serverTimezone=Asia/Shanghai
db.user=root
db.password=123456
```



### 6. nacos集群的意义：

>微服务最重要的就是注册中心，起着生产与消费之间联系的作用，那么nacos宕机将造成程序错误！
>所以要将nacos集群搭建，以防宕机 

```json
1. 复制多份nacos文件，最好以端口号命名方便查看 
```

```json
2. 修改配置 nacos/conf/cluster.conf 这个文件本来没有的，通过复制 cluster.conf.example  文件的副本改名得来 

192.168.203.1:8848
192.168.203.1:8849
# 因为是模拟多主机集群，这里用的是ip+端口的形式。
# 两个记录为集群的主机ip和nacos端口
# 不要写localhost
```

```json
3. 修改配置 application.properties ( 两个nacos都要有该文件，内容相同 )
server.port=8848 / 8849
spring.datasource.platform=mysql
db.num=1
db.url.0=jdbc:mysql://localhost:3306/nacos?>useSSL=false&serverTimezone=UTC&serverTimezone=Asia/Shanghai	
db.user=root
db.password=123456
```

```json
4. 修改启动模式为： cluster ( nacos\bin\starup.cmd *27行* , 如是集群模式不用修改)
set MODE="cluster" 
```





### 7. 启动

> 分别配置好二个项目配置文件后，启动各自startup文件，如下结果表示成功

```cmd

         ,--.
       ,--.'|
   ,--,:  : |                                           Nacos 1.1.4
,`--.'`|  ' :                       ,---.               Running in cluster mode, All function modules
|   :  :  | |                      '   ,'\   .--.--.    Port: 8849
:   |   \ | :  ,--.--.     ,---.  /   /   | /  /    '   Pid: 1676
|   : '  '; | /       \   /     \.   ; ,. :|  :  /`./   Console: http://192.168.203.1:8849/nacos/index.html
'   ' ;.    ;.--.  .-. | /    / ''   | |: :|  :  ;_
|   | | \   | \__\/: . ..    ' / '   | .; : \  \    `.      https://nacos.io
'   : |  ; .' ," .--.; |'   ; :__|   :    |  `----.   \
|   | '`--'  /  /  ,.  |'   | '.'|\   \  /  /  /`--'  /
'   : |     ;  :   .'   \   :    : `----'  '--'.     /
;   |.'     |  ,     .-./\   \  /            `--'---'
'---'        `--`---'     `----'

2020-09-13 15:22:15,188 INFO The server IP list of Nacos is [192.168.203.1:8848, 192.168.203.1:8849]

2020-09-13 15:22:16,198 INFO Nacos is starting...

2020-09-13 15:22:17,200 INFO Nacos is starting...

2020-09-13 15:22:18,201 INFO Nacos is starting...

2020-09-13 15:22:19,203 INFO Nacos is starting...

2020-09-13 15:22:20,205 INFO Nacos is starting...

2020-09-13 15:22:21,206 INFO Nacos is starting...

2020-09-13 15:22:21,669 INFO Nacos Log files: D:\nacos\nacos-8849\nacos/logs/

2020-09-13 15:22:21,669 INFO Nacos Conf files: D:\nacos\nacos-8849\nacos/conf/

2020-09-13 15:22:21,670 INFO Nacos Data files: D:\nacos\nacos-8849\nacos/data/

2020-09-13 15:22:21,670 INFO Nacos started successfully in cluster mode.

```



### 8. 验证页面效果

> 页面左侧菜单 集群管理-> 节点列表 如下效果表示集群成功部署，如有问题可重复操作第6步骤

- [集群管理](http://127.0.0.1:8848/nacos/)
  - 节点列表

### 节点列表|public

节点Ip

查询

| 节点Ip             | 节点状态 | 集群任期 | Leader止时(ms) | 心跳止时(ms) |
| :----------------- | :------- | :------- | :------------- | :----------- |
| 192.168.203.1:8848 | LEADER   | 2        | 16911          | 5000         |
| 192.168.203.1:8849 | FOLLOWER | 2        | 15544          | 5000         |



### 9. 验证是否真的配置成功，且密码可配置化

> 1. 直接修改 mysql 下nacos 库下 `users`下nacos用户的密码为: (123)
>
>    $2a$10$.62erVHP8ojd5jEc7PHBKuO48jbojtxJCUwefxDTzuIcacJwtnUm.
>
> 2. 验证集群
>
>    192.168.203.1:8848/nacos
>
>    192.168.203.1:8849/nacos
>
>    使用nacos/123 是否可以成功登录



## 3.   [Nacos与Spring Cloud一起使用](https://nacos.io/zh-cn/docs/use-nacos-with-springcloud.html) 



### 3.1. 在Spring Cloud项目的pom.xml文件中添加依赖项spring-cloud-starter-alibaba-nacos-discovery。 

>  https://mvnrepository.com/artifact/com.alibaba.cloud/spring-cloud-starter-alibaba-nacos-discovery  可查看版本

``` xml
 <dependency>
     <groupId>com.alibaba.cloud</groupId>
     <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
     <version>2.1.0.RELEASE</version>
 </dependency>
```



### 3.2. 将Nacos服务器地址配置添加到文件/src/main/resources/application.properties。 

```properties
spring:
  application:
    name: test-server
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848  
```



### 3. 3 使用@EnableDiscoveryClient注解打开服务注册和发现。 

```java
 @SpringBootApplication
 // 此处导入@EnableDiscoveryClient注解即可
 @EnableDiscoveryClient
 public class TestApplication {

 	public static void main(String[] args) {
 		SpringApplication.run(TestApplication.class, args);
 	}
 }
```





## 4.  服务注册成功效果

| 服务名      | 分组名称      | 集群数目 | 实例数 | 健康实例数 | 触发保护阈值 |         操作         |
| :---------- | :------------ | :------- | :----- | :--------- | :----------- | :------------------: |
| test-server | DEFAULT_GROUP | 1        | 1      | 1          | false        | 详情\|示例代码\|删除 |





## 5.  更多配置项

| 配置项目   | 键                                             | 默认值                  | 描述                                                         |
| ---------- | ---------------------------------------------- | ----------------------- | ------------------------------------------------------------ |
| 服务器地址 | spring.cloud.nacos.discovery.server-addr       |                         |                                                              |
| 服务       | spring.cloud.nacos.discovery.service           | spring.application.name | 服务ID到注册表                                               |
| 重量       | spring.cloud.nacos.discovery.weight            | 1个                     | 值从1到100，值越大，权重越大                                 |
| ip         | spring.cloud.nacos.discovery.ip                |                         | 注册表的IP地址，最高优先级                                   |
| 网络接口   | spring.cloud.nacos.discovery.network-interface |                         | 未配置IP时，注册的IP地址为该网络接口对应的IP地址。如果未配置此项，则默认使用第一个网络接口的地址。 |
| 港口       | spring.cloud.nacos.discovery.port              | -1                      | 端口到注册表，无需配置即可自动检测                           |
| 名字空间   | spring.cloud.nacos.discovery.namespace         |                         | 常见方案之一是分离不同环境的配置，例如测试环境的开发和生产环境的资源隔离。 |
| 快捷键     | spring.cloud.nacos.discovery.access-key        |                         |                                                              |
| 密钥       | spring.cloud.nacos.discovery.secret-key        |                         |                                                              |
| 元数据     | spring.cloud.nacos.discovery.metadata          |                         | 扩展数据，使用地图格式进行配置                               |
| 日志名称   | spring.cloud.nacos.discovery.log名称           |                         |                                                              |
| 终点       | spring.cloud.nacos.discovery.endpoint          |                         | 服务的域名，通过该域名可以动态获取服务器地址。               |
| 整合功能区 | ribbon.nacos.enabled                           | 真正                    |                                                              |





