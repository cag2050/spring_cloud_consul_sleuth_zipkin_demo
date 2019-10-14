### Spring Cloud Sleuth 集成 Zipkin，实现分布式链路追踪

### 步骤一：Zipkin 的 Docker 镜像使用
> 在Spring Cloud D版本，zipkin-server通过引入依赖的方式构建工程，自从E版本之后，这一方式改变了，采用官方的jar形式启动

1.Zipkin 在 dockerhub 上网址：https://hub.docker.com/r/openzipkin/zipkin

2.下载镜像
```
docker pull openzipkin/zipkin:2.17.2
```
3.创建容器：这种方式启动，将链路数据存在了内存中，只要zipkin-server重启之后，之前的链路数据全部查找不到了，zipkin是支持将链路数据存储在mysql、cassandra、elasticsearch中的。
```
docker run -p 9411:9411 openzipkin/zipkin:2.17.2
```
4.在浏览器上访问Web界面
```
http://localhost:9411/zipkin/
```

### 步骤二：创建 Consul 容器
1.镜像官方网址：https://hub.docker.com/_/consul

2.pull 镜像：
```
docker pull consul:1.6.0
```
3.创建容器（默认http管理端口：8500）
```
docker run -p 8500:8500 consul:1.6.0
```
4.访问管理网址
```
http://localhost:8500/
```

### 步骤三：改造服务提供者项目
1.项目github：https://github.com/cag2050/spring_cloud_consul_producer_demo
2.需要在工程的pom文件加上sleuth的起步依赖和zipkin的起步依赖
```
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-zipkin</artifactId>
</dependency>
```
3.在工程的配置文件 application.properties 需要做以下的配置：
```
#=== sleuth链路监控 ===
spring.sleuth.web.client.enabled=true
# 将采样比例设置为 1.0，也就是全部都需要。默认是 0.1
spring.sleuth.sampler.probability=1.0
# 指定了 Zipkin 服务器的地址
# 使用http请求的方式，将链路数据发送给zipkin-server
spring.zipkin.base-url=http://localhost:9411/
```

### 步骤四：改造使用 FeignClient 消费服务的服务消费者
1.项目github：https://github.com/cag2050/spring_cloud_consul_consumer3_feign_demo
2.同服务提供者一样，需要在工程的pom文件加上sleuth的起步依赖和zipkin的起步依赖，另外也需要在配置文件application.properties做相关的配置，具体同步骤二的服务提供者。

### 步骤五：链路数据验证
1. 依次执行步骤一、步骤二，启动步骤三对应的服务提供者应用，启动步骤四对应的服务消费者应用，等所有应用启动完成后，在浏览器上访问服务消费者：http://localhost:8505/hello
（如果报错，是服务与发现需要一定的时间，耐心等待几十秒），访问成功后，再次在浏览器上访问zipkin-server的页面：http://127.0.0.1:9411/zipkin/
2. 点击页面：http://127.0.0.1:9411/zipkin/ 上的"Find Traces"按钮，可以看到每次请求所消耗的时间，以及一些span的信息。
3.点击"Dependencies"文字，可以看出具体的服务依赖关系

### 步骤六：将链路数据保存在 Elasticsearch 中
1.zipkin-server支持将链路数据存储在ElasticSearch中，zipkin连接elasticsearch是从环境变量中读取的

2.从网址：https://repo1.maven.org/maven2/io/zipkin/zipkin-server/2.17.2/zipkin-server-2.17.2-exec.jar
 下载jar(已下载在本项目中；如果需要其他版本，可去相应文件夹下载；是一个打包好的springBoot应用，springBoot自带tomcat因此只要启动JAR包就可以访问了。 java -jar zipkin-server-xxx.jar 启动完后访问 http://localhost:9411 可以查看统计界面)。
 
3.下载elasticsearch镜像：`docker pull elasticsearch:7.4.0`，创建 Elasticsearch 容器：`docker run --name elasticsearch -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" elasticsearch:7.4.0`，访问网址：http://localhost:9200/

4.在项目目录下，使用下面命令启动 zipkin-server:
```
java -jar zipkin-server-2.17.2-exec.jar --STORAGE_TYPE=elasticsearch --ES_HOSTS=http://localhost:9200 --ES_INDEX=zipkin
```

5.启动完成后，然后在浏览数上访问服务消费者：http://localhost:8505/hello，再访问http://localhost:9411/zipkin/，可以看到链路数据。这时链路数据存储在 ElasticSearch 中
（可以通过谷歌浏览器扩展程序可视化界面：ElasticSearch Head 查看数据；该扩展程序下载说明：https://www.cnblogs.com/cag2050/p/10277101.html）。

### 步骤七：在 Kibana 上展示链路数据
1. 链路数据存储在ElasticSearch中，ElasticSearch可以和Kibana结合，将链路数据展示在Kibana上。
2. 下载镜像：`docker pull kibana:7.4.0`，创建容器（Kibana 默认的端口为5601；kibana出现[Kibana server is not ready yet]问题的解决：https://blog.csdn.net/qq_38796327/article/details/90480314；ELASTICSEARCH_URL中指定的elasticsearch的容器地址最好是采用本机的ip地址 + 端口）：`docker run -p 5601:5601 --link elasticsearch -e
 "ELASTICSEARCH_URL=http://192.168.1.7:9200" kibana:7.4.0`，访问网址：`http://localhost:5601`
3. 安装完成Kibana后启动，Kibana默认会向本地端口为9200的ElasticSearch读取数据。Kibana默认的端口为5601，访问Kibana的主页 http://localhost:5601
4. 在网址 http://localhost:5601 中，单击“Management”按钮，单击“Index patterns”，单击“Create index pattern”添加一个index。我们在上节ElasticSearch中写入链路数据的index配置为“zipkin
”，那么在界面填写为“zipkin-*”，单击“Next step”按钮，单击“Create index pattern”；创建完成index后，单击“Discover”，就可以在界面上展示链路数据了
5.网址：http://localhost:5601/app/kibana#/visualize?_g=() ，创建一个图，就可以在图上看到记录数。
6. 注意：Kibana 版本必须和 Elasticsearch 版本一样，否则 Kibana 会报错误：`Kibana server is not ready yet`。

### 注意
1.如果报错：
```
Fielddata is disabled on text fields by default. Set fielddata=true on [type] in order to load fielddata in memory by uninverting the inverted index. Note that this can however use significant memory. Alternatively use a keyword field instead.
```
参考官方网站：https://www.elastic.co/guide/en/elasticsearch/reference/current/fielddata.html#enable-fielddata-text-fields
 ，为字段添加 fielddata 属性。

### 参考
参考资料 | 网址
--- | ---
Spring Cloud Sleuth 之Greenwich版本全攻略 | https://juejin.im/post/5c623c195188256219175369