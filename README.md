
# SpringBoot应用整合ELK实现日志收集

> ELK即Elasticsearch、Logstash、Kibana,组合起来可以搭建线上日志系统，本文主要讲解使用ELK来收集SpringBoot应用产生的日志。

## 学习前需要了解的内容

- [开发者必备Docker命令](https://mp.weixin.qq.com/s/d_CuljDTJq680NTndAay8g)

- [使用Docker Compose部署SpringBoot应用](https://mp.weixin.qq.com/s/iMl9bJ4SxUsNHBbiS5VUcw)

- [SpringBoot应用中使用AOP记录接口访问日志](https://mp.weixin.qq.com/s/mNujRjejQ1bITveFI6gkcg)

## ELK中各个服务的作用

- Elasticsearch:用于存储收集到的日志信息；
- Logstash:用于收集日志，SpringBoot应用整合了Logstash以后会把日志发送给Logstash,Logstash再把日志转发给Elasticsearch；
- Kibana:通过Web端的可视化界面来查看日志。

## 使用Docker Compose 搭建ELK环境

### 需要下载的Docker镜像

```shell
docker pull elasticsearch:6.4.0
docker pull logstash:6.4.0
docker pull kibana:6.4.0
```

### 搭建前准备

- elasticsearch 启动成功需要特殊配置，具体[参考](https://www.cnblogs.com/balloon72/p/13177872.html)中的elasticsearch部分；

- docker-compose.yml文件地址:https://github.com/macrozheng/mall-learning/blob/master/mall-tiny-elk/src/main/docker/docker-compose.yml

- logstash-springboot.conf配置文件地址：https://github.com/macrozheng/mall-learning/blob/master/mall-tiny-elk/src/main/docker/logstash-springboot.conf

### 开始搭建

#### 创建一个存放logstash配置的目录并上传配置文件

##### 创建配置文件存放目录并上传配置文件到该目录

```shell
# 进入根目录
[root@192 /]# pwd
/

# 创建存放logstash文件夹
mkdir -p /mydata/logstash

[root@192 logstash]# /mydata/logstash
[root@192 logstash]# pwd
/mydata/logstash
[root@192 logstash]# touch logstash-springboot.conf
[root@192 logstash]# ll

 logstash-springboot.conf

```

##### logstash-springboot.conf文件内容
```
input {
  tcp {
    mode => "server"
    host => "0.0.0.0"
    port => 4560
    codec => json_lines
  }
}
output {
  elasticsearch {
    hosts => "es:9200"
    index => "springboot-logstash-%{+YYYY.MM.dd}"
  }
}
```
#### 配置Elasticsearch
##### 配置参数并且创建目录
```shell script
# 需要设置系统内核参数，否则会因为内存不足无法启动。
# 改变设置
sysctl -w vm.max_map_count=262144
# 使之立即生效
sysctl -p

# 创建目录
mkdir -p /mydata/elasticsearch/data/
# 配置权限
chmod -R 777 /mydata/elasticsearch/data
``` 

#### 使用docker-compose.yml脚本启动ELK服务

##### docker-compose.yml内容
```yml
version: '3'
services:
  elasticsearch:
    image: elasticsearch:6.4.0
    container_name: elasticsearch
    environment:
      - "cluster.name=elasticsearch" #设置集群名称为elasticsearch
      - "discovery.type=single-node" #以单一节点模式启动
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m" #设置使用jvm内存大小
    volumes:
      - /mydata/elasticsearch/plugins:/usr/share/elasticsearch/plugins #插件文件挂载
      - /mydata/elasticsearch/data:/usr/share/elasticsearch/data #数据文件挂载
    ports:
      - 9200:9200
      - 9300:9300
  kibana:
    image: kibana:6.4.0
    container_name: kibana
    links:
      - elasticsearch:es #可以用es这个域名访问elasticsearch服务
    depends_on:
      - elasticsearch #kibana在elasticsearch启动之后再启动
    environment:
      - "elasticsearch.hosts=http://es:9200" #设置访问elasticsearch的地址
    ports:
      - 5601:5601
  logstash:
    image: logstash:6.4.0
    container_name: logstash
    volumes:
      - /mydata/logstash/logstash-springboot.conf:/usr/share/logstash/pipeline/logstash.conf #挂载logstash的配置文件
    depends_on:
      - elasticsearch #kibana在elasticsearch启动之后再启动
    links:
      - elasticsearch:es #可以用es这个域名访问elasticsearch服务
    ports:
      - 4560:4560
```



##### 上传到linux服务器并使用docker-compose命令运行
```shell
docker-compose up -d
```
注意：Elasticsearch启动可能需要好几分钟，要耐心等待。

![tech_screen_01.png](https://upload-images.jianshu.io/upload_images/4994935-ab850f3b7127738f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### 在logstash中安装json_lines插件
```shell
# 进入logstash容器
docker exec -it logstash /bin/bash
# 进入bin目录
cd /bin/
# 安装插件
logstash-plugin install logstash-codec-json_lines
# 退出容器
exit
# 重启logstash服务
docker restart logstash
```
#### 开启防火墙并在kibana中查看
```shell
systemctl stop firewalld
```
访问地址：http://192.168.3.101:5601

![tech_screen_02.png](https://upload-images.jianshu.io/upload_images/4994935-edcd2e4fc9a755bb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## SpringBoot应用集成Logstash

### 在pom.xml中添加logstash-logback-encoder依赖
```xml
<!--集成logstash-->
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>5.3</version>
</dependency>
```

### 添加配置文件logback-spring.xml让logback的日志输出到logstash

> 注意appender节点下的destination需要改成你自己的logstash服务地址，比如我的是：192.168.3.101:4560 。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE configuration>
<configuration>
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>
    <include resource="org/springframework/boot/logging/logback/console-appender.xml"/>
    <!--应用名称-->
    <property name="APP_NAME" value="mall-admin"/>
    <!--日志文件保存路径-->
    <property name="LOG_FILE_PATH" value="${LOG_FILE:-${LOG_PATH:-${LOG_TEMP:-${java.io.tmpdir:-/tmp}}}/logs}"/>
    <contextName>${APP_NAME}</contextName>
    <!--每天记录日志到文件appender-->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${LOG_FILE_PATH}/${APP_NAME}-%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>${FILE_LOG_PATTERN}</pattern>
        </encoder>
    </appender>
    <!--输出到logstash的appender-->
    <appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
        <!--可以访问的logstash日志收集端口-->
        <destination>192.168.3.101:4560</destination>
        <encoder charset="UTF-8" class="net.logstash.logback.encoder.LogstashEncoder"/>
    </appender>
    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="FILE"/>
        <appender-ref ref="LOGSTASH"/>
    </root>
</configuration>
```


![image.png](https://upload-images.jianshu.io/upload_images/4994935-1140ad0cbd70b6c8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 运行Springboot应用

![tech_screen_03.png](https://upload-images.jianshu.io/upload_images/4994935-7ecd42de9d24857f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 在kibana中查看日志信息

### 创建index pattern

![tech_screen_04.png](https://upload-images.jianshu.io/upload_images/4994935-d1d769849b738fb7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![tech_screen_05.png](https://upload-images.jianshu.io/upload_images/4994935-225f51890a04b934.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![tech_screen_06.png](https://upload-images.jianshu.io/upload_images/4994935-bb76e997bf91927a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 查看收集的日志

![tech_screen_07.png](https://upload-images.jianshu.io/upload_images/4994935-449ee99c190bc96e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 调用接口进行测试

![tech_screen_08.png](https://upload-images.jianshu.io/upload_images/4994935-aba33e193007ae09.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![tech_screen_09.png](https://upload-images.jianshu.io/upload_images/4994935-7d6fdab62925e849.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 制造一个异常并查看

#### 修改获取所有品牌列表接口

![tech_screen_10.png](https://upload-images.jianshu.io/upload_images/4994935-b42ac4b5ea8eedd9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 调用该接口并查看日志

![tech_screen_11.png](https://upload-images.jianshu.io/upload_images/4994935-1b5a93fa44204c3a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 总结
搭建了ELK日志收集系统之后，我们如果要查看SpringBoot应用的日志信息，就不需要查看日志文件了，直接在Kibana中查看即可。

