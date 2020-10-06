## Spring Boot 不停机动态管理数据多主多从数据源



### 1.  AbstractRoutingDataSource 可实现动态数据源切换

> 1. 包的位置:  org.springframework.jdbc.datasource.lookup.AbstractRoutingDataSource
>
> 2.  通过 determineCurrentLookupKey() 方法会在程序运行时选择已加载的数据源
> 3.  如果需要自己实现动态选择数据源，实现 AbstractRoutingDataSource 类即可，但是此时只能在已加载的数据源列表中自由切换，通过AOP可解耦实现动态切换， **并不能解决程序不停机，动态管理数据源, 比如：删除一个或全部，使用新的数据连接信息加载新数据源**



```java
public abstract class AbstractRoutingDataSource extends AbstractDataSource implements InitializingBean {
    @Nullable
    private Map<Object, Object> targetDataSources;
    @Nullable
    private Object defaultTargetDataSource;
    private boolean lenientFallback = true;
    private DataSourceLookup dataSourceLookup = new JndiDataSourceLookup();
    @Nullable
    private Map<Object, DataSource> resolvedDataSources;
    @Nullable
    private DataSource resolvedDefaultDataSource;

    public AbstractRoutingDataSource() {
    }

    public void setTargetDataSources(Map<Object, Object> targetDataSources) {
        this.targetDataSources = targetDataSources;
    }

    public void setDefaultTargetDataSource(Object defaultTargetDataSource) {
        this.defaultTargetDataSource = defaultTargetDataSource;
    }

    public void setLenientFallback(boolean lenientFallback) {
        this.lenientFallback = lenientFallback;
    }

    public void setDataSourceLookup(@Nullable DataSourceLookup dataSourceLookup) {
        this.dataSourceLookup = (DataSourceLookup)(dataSourceLookup != null ? dataSourceLookup : new JndiDataSourceLookup());
    }

    public void afterPropertiesSet() {
        if (this.targetDataSources == null) {
            throw new IllegalArgumentException("Property 'targetDataSources' is required");
        } else {
            this.resolvedDataSources = new HashMap(this.targetDataSources.size());
            this.targetDataSources.forEach((key, value) -> {
                Object lookupKey = this.resolveSpecifiedLookupKey(key);
                DataSource dataSource = this.resolveSpecifiedDataSource(value);
                this.resolvedDataSources.put(lookupKey, dataSource);
            });
            if (this.defaultTargetDataSource != null) {
                this.resolvedDefaultDataSource = this.resolveSpecifiedDataSource(this.defaultTargetDataSource);
            }

        }
    }

    protected Object resolveSpecifiedLookupKey(Object lookupKey) {
        return lookupKey;
    }

    protected DataSource resolveSpecifiedDataSource(Object dataSource) throws IllegalArgumentException {
        if (dataSource instanceof DataSource) {
            return (DataSource)dataSource;
        } else if (dataSource instanceof String) {
            return this.dataSourceLookup.getDataSource((String)dataSource);
        } else {
            throw new IllegalArgumentException("Illegal data source value - only [javax.sql.DataSource] and String supported: " + dataSource);
        }
    }

    public Connection getConnection() throws SQLException {
        return this.determineTargetDataSource().getConnection();
    }

    public Connection getConnection(String username, String password) throws SQLException {
        return this.determineTargetDataSource().getConnection(username, password);
    }

    public <T> T unwrap(Class<T> iface) throws SQLException {
        return iface.isInstance(this) ? this : this.determineTargetDataSource().unwrap(iface);
    }

    public boolean isWrapperFor(Class<?> iface) throws SQLException {
        return iface.isInstance(this) || this.determineTargetDataSource().isWrapperFor(iface);
    }

    protected DataSource determineTargetDataSource() {
        Assert.notNull(this.resolvedDataSources, "DataSource router not initialized");
        Object lookupKey = this.determineCurrentLookupKey();
        DataSource dataSource = (DataSource)this.resolvedDataSources.get(lookupKey);
        if (dataSource == null && (this.lenientFallback || lookupKey == null)) {
            dataSource = this.resolvedDefaultDataSource;
        }

        if (dataSource == null) {
            throw new IllegalStateException("Cannot determine target DataSource for lookup key [" + lookupKey + "]");
        } else {
            return dataSource;
        }
    }

    @Nullable
    protected abstract Object determineCurrentLookupKey();
}

```





### 2.  整合 dynamic-datasource-spring-boot-starter

> dynamic-datasource-spring-boot-starter 是一个基于springboot的快速集成多数据源的启动器
>
> github:  <https://github.com/baomidou/dynamic-datasource-spring-boot-starter>
>
> 文档： <https://github.com/baomidou/dynamic-datasource-spring-boot-starter/wiki
>
> 网址:  https://baomidou.gitee.io/dynamic-datasource-doc/
>
> 它与mybatis-plus是一个生态圈里的，很容易集成mybatis-plus，无缝整合



####  2.1 特性

> 1. 支持 **数据源分组** ，适用于多种场景 纯粹多库 读写分离 一主多从 混合模式。
> 2. 支持数据库敏感配置信息 **加密** ENC()。
> 3. 支持每个数据库独立初始化表结构schema和数据库database。
> 4. 支持 **自定义注解** ，需继承DS(3.2.0+)。
> 5. 提供对Druid，Mybatis-Plus，P6sy，Jndi的快速集成。
> 6. 简化Druid和HikariCp配置，提供 **全局参数配置** 。配置一次，全局通用。
> 7. 提供 **自定义数据源来源** 方案。
> 8. 提供项目启动后 **动态增加移除数据源** 方案。
> 9. 提供Mybatis环境下的 **纯读写分离** 方案。
> 10. 提供使用 **spel动态参数** 解析数据源方案。内置spel，session，header，支持自定义。
> 11. 支持 **多层数据源嵌套切换** 。（ServiceA >>> ServiceB >>> ServiceC）。
> 12. 提供对shiro，sharding-jdbc,quartz等第三方库集成的方案,注意事项和示例。
> 13. 提供 **基于seata的分布式事务方案。** 附：不支持原生spring事务。



#### 2. 2 导入依赖

```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>dynamic-datasource-spring-boot-starter</artifactId>
    <version>3.2.1</version>
</dependency>
```



#### 2.3 配置

```yaml
spring:
  datasource:
    dynamic:
      primary: master #设置默认的数据源或者数据源组,默认值即为master
      strict: false #设置严格模式,默认false不启动. 启动后在未匹配到指定数据源时候会抛出异常,不启动则使用默认数据源.
      datasource:
        master:
          url: jdbc:mysql://xx.xx.xx.xx:3306/dynamic
          username: root
          password: 123456
          driver-class-name: com.mysql.jdbc.Driver # 3.2.0开始支持SPI可省略此配置
        slave_1:
          url: jdbc:mysql://xx.xx.xx.xx:3307/dynamic
          username: root
          password: 123456
          driver-class-name: com.mysql.jdbc.Driver
        slave_2:
          url: ENC(xxxxx) # 内置加密,使用请查看详细文档
          username: ENC(xxxxx)
          password: ENC(xxxxx)
          driver-class-name: com.mysql.jdbc.Driver
          schema: db/schema.sql # 配置则生效,自动初始化表结构
          data: db/data.sql # 配置则生效,自动初始化数据
          continue-on-error: true # 默认true,初始化失败是否继续
          separator: ";" # sql默认分号分隔符
          
       #......省略
       #以上会配置一个默认库master，一个组slave下有两个子库slave_1,slave_2
```



```
# 多主多从                      纯粹多库（记得设置primary）                   混合配置
spring:                               spring:                               spring:
  datasource:                           datasource:                           datasource:
    dynamic:                              dynamic:                              dynamic:
      datasource:                           datasource:                           datasource:
        master_1:                             mysql:                                master:
        master_2:                             oracle:                               slave_1:
        slave_1:                              sqlserver:                            slave_2:
        slave_2:                              postgresql:                           oracle_1:
        slave_3:                              h2:                                   oracle_2:
```



#### 2.4 使用注解 **@DS** 切换数据源

| 注解          | 结果                                     |
| ------------- | ---------------------------------------- |
| 没有@DS       | 默认数据源                               |
| @DS("dsName") | dsName可以为组名也可以为具体某个库的名称 |

> 说明：如果是多主多从，如没有注解默认使用多主库列表，使用默认轮训在主库列表中依次选择主库



```java
@Service
@DS("slave")
public class UserServiceImpl implements UserService {

  @Autowired
  private JdbcTemplate jdbcTemplate;

  public List selectAll() {
    return  jdbcTemplate.queryForList("select * from user");
  }
  
  @Override
  @DS("slave_1")
  public List selectByCondition() {
    return  jdbcTemplate.queryForList("select * from user where age >10");
  }
}
```



#### 2.5 自定义，实现动态管理数据源

>  基于dynamic-datasource-spring-boot-starter添加自定义的数据源管理功能，此处只作功能实现



```java
package com.microservice.controller;

import com.alibaba.druid.pool.DruidDataSource;
import com.baomidou.dynamic.datasource.DynamicGroupDataSource;
import com.baomidou.dynamic.datasource.DynamicRoutingDataSource;
import com.baomidou.dynamic.datasource.creator.DataSourceCreator;
import com.baomidou.dynamic.datasource.spring.boot.autoconfigure.DataSourceProperty;
import com.google.common.collect.Lists;
import com.microservice.bean.DataSourceProperties;
import com.microservice.dto.DruidDataSourceDto;
import io.swagger.annotations.ApiOperation;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.BeanUtils;
import org.springframework.beans.BeansException;
import org.springframework.util.CollectionUtils;
import org.springframework.util.StringUtils;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.*;

import javax.annotation.Resource;
import javax.sql.DataSource;
import java.util.*;
import java.util.stream.Collectors;

/**
 * @author zw
 * @date 2020-11-21
 * <p>
 */
@RestController
@RequestMapping(value = "/dataSource")
@Slf4j
public class DataSourceDynamicController {

    @Resource
    private DataSource dataSource;

    @Resource
    private DataSourceCreator dataSourceCreator;

    @PostMapping(value = "/add")
    @ApiOperation("动态添加数据源")
    public String add(@Validated @RequestBody DataSourceProperties properties) {
        if (Objects.isNull(properties)) {
            return "参数不合法";
        }

        String poolName = properties.getPoolName();
        try {
            if (StringUtils.isEmpty(poolName)) {
                return "参数不合法";
            }
            List<DruidDataSourceDto> list = this.list();
            long count = list.parallelStream().filter(f -> Objects.deepEquals(poolName, f.getDataSourceName()))
                    .count();
            if (count > 0) {
                return "已存在数据源: " + poolName;
            }
            DataSourceProperty dataSourceProperty = new DataSourceProperty();
            BeanUtils.copyProperties(properties, dataSourceProperty);

            DynamicRoutingDataSource ds = (DynamicRoutingDataSource) dataSource;
            DataSource dataSource = dataSourceCreator.createDataSource(dataSourceProperty);
            ds.addDataSource(poolName, dataSource);
        } catch (BeansException e) {
            log.error(">>> 动态添加数据源失败, 原因: {}", e.getLocalizedMessage(), e);
            return "添加失败, 原因: " + e.getLocalizedMessage();
        }

        List<DruidDataSourceDto> list = this.list();
        long count = list.parallelStream().filter(f -> Objects.deepEquals(poolName, f.getDataSourceName()))
                .count();
        return count == 1 ? "添加成功" : "添加失败";
    }

    @DeleteMapping(value = "/delete")
    @ApiOperation("动态删除数据源")
    public String remove(String name) {
        List<DruidDataSourceDto> list = this.list();
        if (CollectionUtils.isEmpty(list)) {
            return "未发现数据库";
        }

        long master = list.stream()
                .filter(f -> !Objects.deepEquals(name.toLowerCase().trim(), f.getDataSourceName()) && Objects.deepEquals("master", f.getGroupName()))
                .count();
        if (master <= 0) {
            return String.format("一旦删除: %s, 将不存在主数据库，请先配置[新主数据库], 再删除：%s", name, name);
        }

        // 验证合法性
        DruidDataSourceDto druidDataSourceDto = list.stream()
                .filter(f -> Objects.deepEquals(name.toLowerCase().trim(), f.getDataSourceName()))
                .findFirst()
                .orElseGet((() -> null));
        if (Objects.isNull(druidDataSourceDto)) {
            return "未发现数据库: " + name;
        }

        DynamicRoutingDataSource ds = (DynamicRoutingDataSource) dataSource;
        // 先停掉数据库， 再删除此条记录 ,删除操作自动联动操作
        ds.removeDataSource(name.toLowerCase().trim());

        // 再次验证合法性
        list = this.list();
        druidDataSourceDto = list.stream()
                .filter(f -> Objects.deepEquals(name.toLowerCase().trim(), f.getDataSourceName()))
                .findFirst()
                .orElseGet((() -> null));
        if (Objects.isNull(druidDataSourceDto)) {
            return "删除成功";
        }
        return "删除数据库: " + name + ", 失败";
    }

    @GetMapping(value = "/list")
    @ApiOperation("动态查询所有数据源")
    public List<DruidDataSourceDto> list() {
        DynamicRoutingDataSource dynamicRoutingDataSource = (DynamicRoutingDataSource) dataSource;
        Map<String, DynamicGroupDataSource> currentGroupDataSources = dynamicRoutingDataSource.getCurrentGroupDataSources();
        Map<String, DataSource> currentDataSources = dynamicRoutingDataSource.getCurrentDataSources();

        List<DruidDataSourceDto> ddss = new ArrayList<>(currentDataSources.size());

        // 找出各自的组
        currentDataSources.forEach((dataSourceName, value1) -> {
            // 数据库
            DruidDataSource dataSource = (DruidDataSource) value1;

            if (CollectionUtils.isEmpty(currentGroupDataSources)) {
                // 组成数据
                DruidDataSourceDto dataSourceDto = new DruidDataSourceDto();
                dataSourceDto.setGroupName(null);
                dataSourceDto.setDataSourceName(dataSourceName);
                dataSourceDto.setDbType(dataSource.getDbType());
                dataSourceDto.setDriverClass(dataSource.getDriverClassName());
                dataSourceDto.setUserName(dataSource.getUsername());
                dataSourceDto.setJdbcUrl(dataSource.getRawJdbcUrl());

                ddss.add(dataSourceDto);
            } else {
                currentGroupDataSources.forEach((key, value) -> {
                    String groupName = value.getGroupName();
                    List<DataSource> dss = value.getDataSources();
                    List<DataSource> collect = dss.stream()
                            .filter(f -> Objects.deepEquals(((DruidDataSource) f).getRawJdbcUrl(), dataSource.getRawJdbcUrl()))
                            .collect(Collectors.toList());
                    if (!CollectionUtils.isEmpty(collect)) {
                        collect.forEach(source -> {
                            // 组成数据
                            DruidDataSource dataSource1 = (DruidDataSource) source;

                            DruidDataSourceDto dataSourceDto = new DruidDataSourceDto();
                            dataSourceDto.setGroupName(groupName);
                            dataSourceDto.setDataSourceName(dataSourceName);
                            dataSourceDto.setDbType(dataSource1.getDbType());
                            dataSourceDto.setDriverClass(dataSource1.getDriverClassName());
                            dataSourceDto.setUserName(dataSource.getUsername());
                            dataSourceDto.setJdbcUrl(dataSource1.getRawJdbcUrl());

                            ddss.add(dataSourceDto);
                        });
                    }
                });
            }
        });
        if (CollectionUtils.isEmpty(ddss)) {
            return Lists.newArrayList();
        }
        return ddss.parallelStream()
                .sorted(Comparator.comparing(DruidDataSourceDto::getGroupName))
                .sorted(Comparator.comparing(DruidDataSourceDto::getDataSourceName))
                .sorted(Comparator.comparing(DruidDataSourceDto::getDbType))
                .sorted(Comparator.comparing(DruidDataSourceDto::getUserName))
                .collect(Collectors.toList());
    }
}
```



### 3. 验证功能

> 源码： https://github.com/zhangwei9757/springcloud-zhangwei/tree/master/microservice-dynamic-datasource-spring-boot-starter



#### 3.1 查询项目启动时已初始化的数据源列表

```json
http://localhost:8000/dataSource/list
```



```json
[
  {
    "groupName": "master",
    "dataSourceName": "master_1",
    "userName": "root",
    "jdbcUrl": "jdbc:mysql://127.0.0.1:3306/vhr?useUnicode=true&characterEncoding=UTF-8&serverTimezone=Asia/Shanghai&zeroDateTimeBehavior=convertToNull&useSSL=false",
    "driverClass": "com.mysql.jdbc.Driver",
    "dbType": "mysql"
  },
  {
    "groupName": "slave",
    "dataSourceName": "slave_1",
    "userName": "root",
    "jdbcUrl": "jdbc:mysql://127.0.0.1:3306/nacos?useUnicode=true&characterEncoding=UTF-8&serverTimezone=Asia/Shanghai&zeroDateTimeBehavior=convertToNull&useSSL=false",
    "driverClass": "com.mysql.jdbc.Driver",
    "dbType": "mysql"
  },
  {
    "groupName": "slave",
    "dataSourceName": "slave_2",
    "userName": "root",
    "jdbcUrl": "jdbc:mysql://127.0.0.1:3306/questions?useUnicode=true&characterEncoding=UTF-8&serverTimezone=Asia/Shanghai&zeroDateTimeBehavior=convertToNull&useSSL=false",
    "driverClass": "com.mysql.jdbc.Driver",
    "dbType": "mysql"
  }
]
```



#### 3.2 删除一个数据源

```json
http://localhost:8000/dataSource/add?name=slave_1
```



```shell
2020-11-21 18:37:58.687 INFO  [http-nio-8000-exec-5] --- c.alibaba.druid.pool.DruidDataSource Line:1825 - {dataSource-3} closed
2020-11-21 18:37:58.688 INFO  [http-nio-8000-exec-5] --- c.b.d.d.DynamicRoutingDataSource Line:190 - dynamic-datasource - remove the database named [slave_1] success

```



#### 3.3 添加一个数据源

```json
http://localhost:8000/dataSource/add
```



```json
{
  "driverClassName": "com.mysql.jdbc.Driver",
  "druid": {
    "asyncInit": true,
    "breakAfterAcquireFailure": true,
    "clearFiltersEnable": true,
    "connectionErrorRetryAttempts": 0,
    "connectionProperties": {},
    "failFast": true,
    "filters": "stat,wall",
    "initConnectionSqls": "",
    "initGlobalVariants": true,
    "initVariants": true,
    "initialSize": 3,
    "keepAlive": true,
    "killWhenSocketReadTimeout": true,
    "logAbandoned": true,
    "maxActive": 8,
    "maxEvictableIdleTimeMillis": 30000,
    "maxPoolPreparedStatementPerConnectionSize": 0,
    "maxWait": -1,
    "maxWaitThreadCount": 0,
    "minEvictableIdleTimeMillis": 30000,
    "minIdle": 2,
    "notFullTimeoutRetryCount": 0,
    "phyTimeoutMillis": 0,
    "poolPreparedStatements": true,
    "proxyFilters": [],
    "publicKey": "",
    "queryTimeout": 0,
    "removeAbandoned": true,
    "removeAbandonedTimeoutMillis": 0,
    "resetStatEnable": true,
    "sharePreparedStatements": true,
    "slf4j": {
      "enable": true,
      "statementExecutableSqlLogEnable": true
    },
    "stat": {
      "logSlowSql": true,
      "mergeSql": true,
      "slowSqlMillis": 0
    },
    "statSqlMaxSize": 0,
    "testOnBorrow": false,
    "testOnReturn": false,
    "testWhileIdle": true,
    "timeBetweenEvictionRunsMillis": 0,
    "timeBetweenLogStatsMillis": 0,
    "transactionQueryTimeout": 0,
    "useGlobalDataSourceStat": true,
    "useUnfairLock": true,
    "validationQuery": "select 1",
    "validationQueryTimeout": -1,
    "wall": {
      "alterTableAllow": true,
      "blockAllow": true,
      "callAllow": true,
      "caseConditionConstAllow": true,
      "commentAllow": true,
      "commitAllow": true,
      "completeInsertValuesCheck": true,
      "conditionAndAlwayFalseAllow": true,
      "conditionAndAlwayTrueAllow": true,
      "conditionDoubleConstAllow": true,
      "conditionLikeTrueAllow": true,
      "conditionOpBitwseAllow": true,
      "conditionOpXorAllow": true,
      "constArithmeticAllow": true,
      "createTableAllow": true,
      "deleteAllow": true,
      "deleteWhereAlwayTrueCheck": true,
      "deleteWhereNoneCheck": true,
      "describeAllow": true,
      "dir": "",
      "doPrivilegedAllow": true,
      "dropTableAllow": true,
      "functionCheck": true,
      "hintAllow": true,
      "insertAllow": true,
      "insertValuesCheckSize": 0,
      "intersectAllow": true,
      "limitZeroAllow": true,
      "lockTableAllow": true,
      "mergeAllow": true,
      "metadataAllow": true,
      "minusAllow": true,
      "multiStatementAllow": true,
      "mustParameterized": true,
      "noneBaseStatementAllow": true,
      "objectCheck": true,
      "renameTableAllow": true,
      "replaceAllow": true,
      "rollbackAllow": true,
      "schemaCheck": true,
      "selectAllColumnAllow": true,
      "selectAllow": true,
      "selectExceptCheck": true,
      "selectHavingAlwayTrueCheck": true,
      "selectIntersectCheck": true,
      "selectIntoAllow": true,
      "selectIntoOutfileAllow": true,
      "selectLimit": 0,
      "selectMinusCheck": true,
      "selectUnionCheck": true,
      "selectWhereAlwayTrueCheck": true,
      "setAllow": true,
      "showAllow": true,
      "startTransactionAllow": true,
      "strictSyntaxCheck": true,
      "tableCheck": true,
      "tenantColumn": "",
      "tenantTablePattern": "",
      "truncateAllow": true,
      "updateAllow": true,
      "updateWhereAlayTrueCheck": true,
      "updateWhereNoneCheck": true,
      "useAllow": true,
      "variantCheck": true,
      "wrapAllow": true
    }
  },
  "password": "jzbr",
  "poolName": "master_2",
  "url": "jdbc:mysql://127.0.0.1:3306/vhr?useUnicode=true&characterEncoding=UTF-8&serverTimezone=Asia/Shanghai&zeroDateTimeBehavior=convertToNull&useSSL=false",
  "username": "root"
}
```



```shell
2020-11-21 18:39:35.164 INFO  [http-nio-8000-exec-7] --- c.alibaba.druid.pool.DruidDataSource Line:930 - {dataSource-9,master_2} inited
2020-11-21 18:39:35.165 INFO  [http-nio-8000-exec-7] --- c.b.d.d.DynamicRoutingDataSource Line:128 - dynamic-datasource - load a datasource named [master_2] success
2020-11-21 18:39:35.172 ERROR [Druid-ConnectionPool-Create-1249574468] --- c.a.druid.filter.stat.StatFilter Line:478 - slow sql 2 millis. show variables[]
2020-11-21 18:39:35.176 ERROR [Druid-ConnectionPool-Create-1249574468] --- c.a.druid.filter.stat.StatFilter Line:478 - slow sql 2 millis. show global variables[]
2020-11-21 18:39:35.188 ERROR [Druid-ConnectionPool-Create-1249574468] --- c.a.druid.filter.stat.StatFilter Line:478 - slow sql 3 millis. show variables[]
2020-11-21 18:39:35.193 ERROR [Druid-ConnectionPool-Create-1249574468] --- c.a.druid.filter.stat.StatFilter Line:478 - slow sql 2 millis. show global variables[]
2020-11-21 18:39:35.203 ERROR [Druid-ConnectionPool-Create-1249574468] --- c.a.druid.filter.stat.StatFilter Line:478 - slow sql 2 millis. show variables[]
2020-11-21 18:39:35.208 ERROR [Druid-ConnectionPool-Create-1249574468] --- c.a.druid.filter.stat.StatFilter Line:478 - slow sql 2 millis. show global variables[]

```



#### 4. 具体连接数据源，测试查询指定数据库，可自行编写代码测试

>  到此，不停机管理数据源就已完成 ，如果项目连接的数据库异常下线，可使用接口调用，添加或删除数据源