## 前言

在Spring boot 2的版本中默认使用了 Hikari数据库连接池，号称是世界上最快的连接池技术。**在我们项目中使用了自定义数据源的方式初始化数据库连接，问题：关于Hikari相关的配置`maximum-pool-size`并未能生效**,现总结问题如下

## 有问题的配置方式

yml文件

```yaml
spring:
  datasource:
    db0:
      url: jdbc:mysql://localhost:3306/cockpit_data
      username: root
      password: 123456
      hikari:
      	minimum-idle: 1
      	maximum-pool-size: 10

    db1:
      url: jdbc:mysql://localhost:3306/cockpit
      username: root
      password: 123456
      hikari:
      	minimum-idle: 1
      	maximum-pool-size: 10
```

`DataSourceConfig`

```JAVA
@Configuration
public class DataSourceConfig {

    @Bean
    @Primary
    @ConfigurationProperties(prefix = "spring.datasource.db1")
    public DataSource primaryDataSource() {
        DataSourceProperties dataSourceProperties = db1Properties();
        return dataSourceProperties.initializeDataSourceBuilder().build();
        return DataSourceBuilder.create().type(HikariDataSource.class).build();
    }

    @Bean(name = "odsDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.db0")
    public DataSource odsDataSources() {
        return db0Properties().initializeDataSourceBuilder().build();
    }

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.db0")
    public DataSourceProperties db0Properties() {
        return new DataSourceProperties();
    }

    @Bean(name = "db1DataSourceProperties")
    @ConfigurationProperties(prefix = "spring.datasource.db1")
    public DataSourceProperties db1Properties() {
        return new DataSourceProperties();
        return DataSourceBuilder.create().type(HikariDataSource.class).build();
    }

}
```

上面的方式我采用的是  ` db0Properties().initializeDataSourceBuilder()`方式创建数据源，看源码发现，这中方式与Hikari的配置类，HikariConfig没有任何关联。

![image-20220419104121574](https://img-blog.csdnimg.cn/img_convert/bbfa55d4eba539d03a016bad39b43876.png)

## 正确的配置方式

关注`HikariConfig`类

```JAVA
public class HikariConfig implements HikariConfigMXBean{
   private String jdbcUrl;
   private volatile int maxPoolSize;
   private volatile int minIdle;
   private volatile String username;
   private volatile String password;
    ....
}
```

所以我们可以应该按照如下的配置，对应上HikariConfig类中的属性即可

```yaml
spring:
  datasource:
    db0:
      jdbc-url: jdbc:mysql://localhost:3306/cockpit_data
      username: root
      password: 123456
      minimum-idle: 1
      maximum-pool-size: 2

    db1:
      url: jdbc:mysql://localhost:3306/cockpit
      username: root
      password: 123456
      minimum-idle: 1
      maximum-pool-size: 2
```

`DataSourceConfig`

```java
@Configuration
public class DataSourceConfig {

    @Bean
    @Primary
    @ConfigurationProperties(prefix = "spring.datasource.db1")
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create().type(HikariDataSource.class).build();
    }

    @Bean(name = "odsDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.db0")
    public DataSource odsDataSources() {
        return DataSourceBuilder.create().type(HikariDataSource.class).build();
    }
}
```

Debug,看看运行的是设置的 maximum-pool-size

![image-20220419105143406](https://img-blog.csdnimg.cn/img_convert/51efa66adf1da8ea243188a333d0c616.png)