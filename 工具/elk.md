# 简介

EKL，分别为`ElasticSearch`、`Logstash`、`Kibana`

- **ElasticSearch:** 分布式搜索引擎。具有高可伸缩、高可靠、易管理等特点。可以用于全文检索、结构化检索和分析，并能将这三者结合起来。Elasticsearch 是用Java 基于 Lucene 开发，现在使用最广的开源搜索引擎之一，Wikipedia 、StackOverflow、Github 等都基于它来构建自己的搜索引擎。
- **Logstash:** 数据收集处理引擎。支持动态的从各种数据源搜集数据，并对数据进行过滤、分析、丰富、统一格式等操作，然后存储以供后续使用。
- **Kibana:** 可视化化平台。它能够搜索、展示存储在 Elasticsearch 中索引数据。使用它可以很方便的用图表、表格、地图展示和分析数据。

ps: 版本统一使用同一个，可以在docker hub查询，这里使用`7.17.3`，8.X 版本的elasticsearch存在问题，无法挂载elasticsearch.yml



# ElasticSearch

```shell
# 拉取镜像
docker pull elasticsearch:7.17.3

# 创建挂载目录
mkdir -p /home/elk/elasticsearch/{config,data,logs}

# 赋予权限 docker中elasticsearch的用户UID是1000.
chown -R 1000:1000 /home/elk/elasticsearch

# 创建配置文件
cat <<EOF | sudo tee /home/elk/elasticsearch/config/elasticsearch.yml
cluster.name: "docker-cluster"
network.host: 0.0.0.0
http.cors.enabled: true
http.cors.allow-origin: "*"
EOF

# 启动 由于9000-9999 为项目端口区间，所以将其映射到 9200和9300
docker run -d --restart=always --name elasticsearch -p 9200:9200 -p 9300:9300 \
-e ES_JAVA_OPTS="-Xms512m -Xmx1g" -e "discovery.type=single-node" \
-e "ELASTIC_PASSWORD=es@123" -e "xpack.security.enabled=true" \
-v /home/elk/elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml \
-v /home/elk/elasticsearch/data:/usr/share/elasticsearch/data \
-v /home/elk/elasticsearch/logs:/usr/share/elasticsearch/logs \
elasticsearch:7.17.3
```



# Kibana

- shell

  ```shell
  # 拉取镜像
  docker pull kibana:7.17.3
  
  # 创建目录
  mkdir -p /home/elk/kibana
  
  # 创建配置文件
  cat <<EOF | sudo tee /home/elk/kibana/kibana.yml
  i18n.locale: "zh-CN"
  server.name: kibana
  server.host: "0.0.0.0"
  server.publicBaseUrl: <改为对外访问的url>
  elasticsearch.hosts: ["http://elasticsearch:9200"]
  elasticsearch.username: elastic
  elasticsearch.password: es@123
  xpack.monitoring.ui.container.elasticsearch.enabled: true
  EOF
  
  # 启动
  docker run -d --restart=always --name kibana \
  --log-driver json-file --log-opt max-size=100m --log-opt max-file=2 \
  -v /home/elk/kibana/kibana.yml:/usr/share/kibana/config/kibana.yml \
  -p 5601:5601 --link elasticsearch:elasticsearch kibana:7.17.3
  ```

- 浏览器输入 ip:5601

# Logstash

- shell

  ```shell
  # 拉取镜像
  docker pull logstash:7.17.3
  # 创建目录
  mkdir -p /home/elk/logstash/conf.d
  
  # 创建配置文件并配置内容，具体配置在下面
  vim /home/elk/logstash/logstash.yml
  
  # 创建日志相关配置，具体配置在下面
  vim /home/elk/logstash/conf.d/logstash.conf
  
  
  # 启动
  docker run -d --name=logstash --restart=always  \
  --log-driver json-file --log-opt max-size=100m --log-opt max-file=2 \
  -e ES_JAVA_OPTS="-Xms256m -Xmx512m" \
  -v /home/elk/logstash/logstash.yml:/usr/share/logstash/config/logstash.yml \
  -v /home/elk/logstash/conf.d:/usr/share/logstash/config/conf.d \
  -p 5044:5044 --link elasticsearch:elasticsearch \
  logstash:7.17.3
  
  # 查看容器ip，给后端服务使用
  # docker inspect --format="{{.NetworkSettings.IPAddress}}" logstash
  ```

- logstash.yml

  ```yaml
  http.host: "0.0.0.0"
  path.config: /usr/share/logstash/config/conf.d/*.conf
  path.logs: /var/log/logstash
  ```
  
- logstash.conf

  下面的配置是多项目和多环境配置到一起
  
  ```logstash
  input {
      tcp {
          mode => "server"
          host => "0.0.0.0"
          port => 5044
          codec => json_lines
      }
  }
  
  output {
       if "free" in [service] {
           elasticsearch {
               hosts => ["es的ip:port"]
               index => "logstash-free-%{env}"
               user => "elastic"
               password => "es@123"
           }
       }
  }
  ```
  



# ElasticSearch使用

**下面所有操作仅供参考**

浏览器打开: `ip:5601`

用户名: `elastic`

密码: `es@123`

## 索引模式

注意：有索引数据才能创建索引模式

创建索引模式: `logstash-free`

Management  -> Stack Management -> Kibana -> 索引模式 -> 创建索引模式

## 索引管理

> 只保留30天内的数据

选中`logstash-free` -> 添加生命周期策略 -> 30-days-default



## 创建角色

- 名称: `free-logs`

- 索引权限: 
  - 索引: `logstash-free`
  - 权限: `read`

- Kibana权限:

  - 工作区: `默认`
  - 所有功能的权限: `Customize`
  - 定制功能权限: `Analytics -> Discover -> Read`

  

## 创建用户

略



## 其它设置

Management  -> Stack Management -> 高级设置

- 常规
  - 日期格式: `YYYY-MM-DD HH:mm:ss`
  - 周内日: `Monday`
  - 用于设置日期格式的时区: `Asia/Shanghai`
  - 纳秒格式的日期: `YYYY-MM-DD HH:mm:ss.SSSSSSSSS`
  - 默认路由: `/app/discover`



# spring boot配置

- 添加依赖

  ```xml
  <dependency>
      <groupId>net.logstash.logback</groupId>
      <artifactId>logstash-logback-encoder</artifactId>
  </dependency>
  ```

- logback.xml新增appender

  ```xml
  <appender name="logstash" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
      <destination>${free.log.logstash.host_port}</destination>
      <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
          <level>DEBUG</level>
      </filter>
      <encoder charset="UTF-8" class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
          <providers>
              <pattern>
                  <omitEmptyFields>true</omitEmptyFields>
                  <pattern>
                      ${free.log.encoder.pattern.logstash}
                  </pattern>
              </pattern>
          </providers>
      </encoder>
  </appender>
  ```

- pattern参考: [https://github.com/spring-cloud-samples/sleuth-documentation-apps/blob/main/service1/src/main/resources/logback-spring.xml](https://github.com/spring-cloud-samples/sleuth-documentation-apps/blob/main/service1/src/main/resources/logback-spring.xml)



