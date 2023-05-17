# 搭建xxl-job-admin

[github地址](https://github.com/xuxueli/xxl-job)



## 数据库配置

创建数据库

库名：xxl_job

编码：utf8mb4

初始化脚本：[tables_xxl_job.sql](https://github.com/xuxueli/xxl-job/blob/master/doc/db/tables_xxl_job.sql)



## 通过源码部署

- 源码：[https://github.com/xuxueli/xxl-job](https://github.com/xuxueli/xxl-job)

- 配置：`xxl-job-admin/application.properties`

- maven进行打包




## docker 部署

如需指定版本，先查找tag [https://hub.docker.com/r/xuxueli/xxl-job-admin/](https://hub.docker.com/r/xuxueli/xxl-job-admin/)

```shell
docker pull xuxueli/xxl-job-admin:2.3.1

# 日志挂载 -v /var/log/xxl-job:/data/applogs
docker run -d --restart=always --name xxl-job-admin \
-p 9999:8080 \
-e JAVA_OPTS="-Xmx512m -Xms256m" \
-e PARAMS="\
--server.servlet.context-path=/ \
--spring.datasource.url=jdbc:mysql://172.16.11.185:3306/xxl_job?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&serverTimezone=Asia/Shanghai \
--spring.datasource.username=root \
--spring.datasource.password=ZPqmqiCFmW85" \
xuxueli/xxl-job-admin:2.3.1
```

浏览器打开：`http://ip:9999` 进行访问

admin/123456



# 配置任务

## spring boot 集成

- 依赖

  ```xml
  <dependency>
      <groupId>com.xuxueli</groupId>
      <artifactId>xxl-job-core</artifactId>
  </dependency>
  ```

- 添加配置

  ```java
  import org.springframework.context.annotation.Import;
  
  import java.lang.annotation.ElementType;
  import java.lang.annotation.Retention;
  import java.lang.annotation.RetentionPolicy;
  import java.lang.annotation.Target;
  
  @Target({ElementType.TYPE})
  @Retention(RetentionPolicy.RUNTIME)
  @Import(XxlJobConfig.class)
  public @interface EnableXxlJob {
  
  }
  ```

  ```java
  import com.xxl.job.core.executor.impl.XxlJobSpringExecutor;
  import lombok.Data;
  import lombok.extern.slf4j.Slf4j;
  import org.springframework.boot.context.properties.ConfigurationProperties;
  import org.springframework.context.annotation.Bean;
  import org.springframework.context.annotation.Profile;
  
  @Data
  @Slf4j
  @Profile("!local")
  @ConfigurationProperties(prefix = "xxl.job")
  public class XxlJobConfig {
  
      @Bean
      @Profile("!local")
      public XxlJobSpringExecutor xxlJobSpringExecutor() {
          log.info(">>>>>>>>>>> xxl-job config init, config:{}", this);
          XxlJobSpringExecutor xxlJobSpringExecutor = new XxlJobSpringExecutor();
  
          {
              xxlJobSpringExecutor.setAdminAddresses(adminAddresses);
              xxlJobSpringExecutor.setAppname(executor.appName);
              xxlJobSpringExecutor.setAddress(executor.address);
              xxlJobSpringExecutor.setIp(executor.ip);
              xxlJobSpringExecutor.setPort(executor.port);
              xxlJobSpringExecutor.setAccessToken(accessToken);
              xxlJobSpringExecutor.setLogPath(executor.logPath);
              xxlJobSpringExecutor.setLogRetentionDays(executor.logRetentionDays);
          }
  
          return xxlJobSpringExecutor;
      }
  
  
      private String adminAddresses;
  
      private String accessToken;
  
      private ExecutorConfig executor;
  
      @Data
      public static class ExecutorConfig {
  
          private String appName;
  
          private String address;
  
          private String ip;
  
          private int port;
  
          private String logPath;
  
          private int logRetentionDays;
      }
  
  }
  ```

  

## application.yaml

```yaml
xxl:
  job:
    admin-addresses: http://192.168.0.60:9999
    accessToken: default_token
    executor:
      app-name: ${spring.profiles.active}-${spring.application.name}
```



## admin进行配置

打开xxl-job的地址，进行配置，可以参考官方文档