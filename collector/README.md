## Kafka海量日志收集实战_架构设计讲解

- 安装包

![image.png](https://upload-images.jianshu.io/upload_images/4994935-54f23055728dc8e1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 解压到/user/local

![image.png](https://upload-images.jianshu.io/upload_images/4994935-50dae08121a9e78d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


- ELK技术栈的架构示意图

![image.png](https://upload-images.jianshu.io/upload_images/4994935-0443641a0eeeba05.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> Kafka做缓冲，为Broker；Filebeat为Producer；Logstash为Comsumer

- 海量日志收集实战架构设计图

![image.png](https://upload-images.jianshu.io/upload_images/4994935-c1324564ad1081ab.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## Kafka海量日志收集实战_log4j2日志输出实战

- Log4j2：日志输出、日志分级、日志过滤、MDC线程变量

1. maven配置

```
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.1.5.RELEASE</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>
	<groupId>com.frxs</groupId>
	<artifactId>collector</artifactId>
	<version>1.0.0</version>
	<name>collector</name>
	<description>collector</description>
	
  <dependencies>
      	<dependency>
          	<groupId>org.springframework.boot</groupId>
          	<artifactId>spring-boot-starter-web</artifactId>
          	<!-- 	排除spring-boot-starter-logging -->
          	<exclusions>
              	<exclusion>
                  	<groupId>org.springframework.boot</groupId>
                  	<artifactId>spring-boot-starter-logging</artifactId>
              	</exclusion>
          	</exclusions>
      	</dependency>
      	<dependency>
           	<groupId>org.springframework.boot</groupId>
           	<artifactId>spring-boot-starter-test</artifactId>
           	<scope>test</scope>
      	</dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency> 		
		<!-- log4j2 -->
		<dependency>
		    <groupId>org.springframework.boot</groupId>
		    <artifactId>spring-boot-starter-log4j2</artifactId>
		</dependency> 
	  	<dependency>
	   		<groupId>com.lmax</groupId>
	   		<artifactId>disruptor</artifactId>
	   		<version>3.3.4</version>
	  	</dependency>		
		
      	<dependency>
          	<groupId>com.alibaba</groupId>
          	<artifactId>fastjson</artifactId>
          	<version>1.2.58</version>
      	</dependency>	
      	
	</dependencies>	
	
	<build>
		<finalName>collector</finalName>
    	<!-- 打包时包含properties、xml -->
    	<resources>
             <resource>  
                <directory>src/main/java</directory>  
                <includes>  
                    <include>**/*.properties</include>  
                    <include>**/*.xml</include>  
                </includes>  
                <!-- 是否替换资源中的属性-->  
                <filtering>true</filtering>  
            </resource>  
            <resource>  
                <directory>src/main/resources</directory>  
            </resource>      	
    	</resources>		
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
				<configuration>
					<mainClass>com.bfxy.collector.Application</mainClass>
				</configuration>
			</plugin>
		</plugins>
	</build> 	

</project>
```

2. application.yml

```
spring:
  application:
    name: kafka=log-collect
  http:
    encoding:
      charset: utf-8
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: GMT+8
    default-property-inclusion: non_null
```
3. IndexController

```
@Slf4j
@RestController("index")
public class IndexController {
    @GetMapping("")
    public String index(){
        log.info("我是一条info日志.");
        log.warn("我是一条warn日志.");
        log.error("我是一条error日志.");
        return "idx";
    }
}
```

4. log4j2.xml

```
<?xml version="1.0" encoding="utf-8" ?>
<Configuration status="INFO" schema="Log4J-V2.0.xsd" monitorInterval="600">
    <Properties>
        <Property name="LOG_HOME">logs</Property>
        <Property name="FILE_NAME">kafka-log-collect</Property>
        <Property name="patternLayout">[%d{yyyy-MM-dd'T'HH:mm:ss.SSSZZ}] [%level{length=5}] [%thread-%tid] [%logger] [%X{hostName}] [%X{ip}] [%X{applicationName}] [%F, %L, %C, %M] [%m] ## '%ex'%n</Property>
    </Properties>
    <Appenders>
        <Console name="CONSOLE" target="SYSTEM_OUT">
            <PatternLayout pattern="${patternLayout}"/>
        </Console>
        <!--全量日志-->
        <RollingRandomAccessFile name="appAppender" fileName="${LOG_HOME}/app-${FILE_NAME}.log" filePattern="${LOG_HOME}/app-${FILE_NAME}-%d{yyyy-MM-dd}-%i.log">
            <PatternLayout pattern="${patternLayout}"/>
            <Policies>
                <TimeBasedTriggeringPolicy interval="1"/>
                <SizeBasedTriggeringPolicy size="500M"/>
            </Policies>
            <DefaultRolloverStrategy max="20"/>
        </RollingRandomAccessFile>
        <!--错误日志,warn级别以上-->
        <RollingRandomAccessFile name="errorAppender" fileName="${LOG_HOME}/error-${FILE_NAME}.log" filePattern="${LOG_HOME}/error-${FILE_NAME}-%d{yyyy-MM-dd}-%i.log">
            <PatternLayout pattern="${patternLayout}"/>
            <Filters>
                <ThresholdFilter level="warn" onMatch="ACCEPT" onMismatch="DENY"/>
            </Filters>
            <Policies>
                <TimeBasedTriggeringPolicy interval="1"/>
                <SizeBasedTriggeringPolicy size="500M"/>
            </Policies>
            <DefaultRolloverStrategy max="20"/>
        </RollingRandomAccessFile>
    </Appenders>
    <Loggers>
        <!--业务相关 异步logger-->
        <AsyncLogger name="com.tuyrk.*" level="info" includeLocation="true">
            <AppenderRef ref="appAppender"/>
        </AsyncLogger>
        <AsyncLogger name="com.tuyrk.*" level="info" includeLocation="true">
            <AppenderRef ref="errorAppender"/>
        </AsyncLogger>
        <Root level="info">
            <Appender-ref ref="CONSOLE"/>
            <Appender-ref ref="appAppender"/>
            <Appender-ref ref="errorAppender"/>
        </Root>
    </Loggers>
</Configuration>
```

## Kafka海量日志收集实战_log4j2日志输出实战

```
@Component
public class InputMDC implements EnvironmentAware {
    private static Environment environment;
    @Override
    public void setEnvironment(Environment environment) {
        InputMDC.environment = environment;
    }

    public static void putMDC() {
        MDC.put("hostName", NetUtils.getLocalHostname());
        MDC.put("ip", NetUtils.getMacAddressString());
        MDC.put("applicationName", environment.getProperty("spring.application.name"));
    }
}
```

---

```
@Slf4j
@RestController
@RequestMapping("index")
public class IndexController {
    @GetMapping("err")
    public String err() {
        InputMDC.putMDC();
        try {
            int a = 1 / 0;
        } catch (Exception e) {
            log.error("算术异常", e);
        }
        return "err";
    }
}

```

> http://localhost:8081/kafka/index

> http://localhost:8081/kafka/index/err

---

## 安装zookeeper

- 下载

```
$ wget https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/zookeeper-3.6.1/apache-zookeeper-3.6.1-bin.tar.gz
$ tar -zxvf apache-zookeeper-3.6.1-bin.tar.gz -C /user/local

cd /usr/local/zookeeper/

# 创建目录
mdkir data
# 进入配置文件目录
cd conf/

cp zoo_sample.cfg zoo.cfg

vim zoo.cfg

# 修改
dataDir=/usr/local/zookeeper/data

```
- 如图修改

![image.png](https://upload-images.jianshu.io/upload_images/4994935-cf80d53f48da47cd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 运行

```
zkServer.sh start

zkServer.sh status

```
## 安装kafka
#### 安装步骤

```

## 解压命令： 
tar -zxvf kafka_2.12-2.1.0.tgz -C /usr/local/ 

## 改名命令： 
mv kafka_2.12-2.1.0/ kafka_2.12

## 进入解压后的目录，修改server.properties文件： 
vim /usr/local/kafka_2.12/config/server.properties 

# 单机版无需下面配置,只需修改
log.dirs=/usr/local/kafka_2.12/kafka-logs 

## 集群修改配置： 
broker.id=0 
port=9092 
host.name=192.168.11.51 
advertised.host.name=192.168.11.51 
log.dirs=/usr/local/kafka_2.12/kafka-logs 
num.partitions=2 
zookeeper.connect=192.168.11.111:2181,192.168.11.112:2181,192.168.11.113:2181 

## 建立日志文件夹： 
mkdir /usr/local/kafka_2.12/kafka-logs 

##启动kafka： 
/usr/local/kafka_2.12/bin/kafka-server-start.sh /usr/local/kafka_2.12/config/server.properties &

```

#### kafka常用命令

```
## 简单操作： 
#（1）创建topic主题命令：（创建名为test的topic， 1个分区分别存放数据，数据备份总共1份） 
kafka-topics.sh --zookeeper 192.168.11.111:2181 --create --topic topic1 --partitions 1 --replication-f 

## --zookeeper 为zk服务列表
## --create 命令后 --topic 为创建topic 并指定 topic name 
## --partitions 为指定分区数量 
## --replication-factor 为指定副本集数量 

#（2）查看topic列表命令： 
kafka-topics.sh --zookeeper 192.168.11.111:2181 --list 

#（3）kafka命令发送数据：（然后我们就可以编写数据发送出去了） 
kafka-console-producer.sh --broker-list 192.168.11.51:9092 --topic topic1 

#（4）kafka命令接受数据：（然后我们就可以看到消费的信息了） 
kafka-console-consumer.sh --bootstrap-server 192.168.11.51:9092 --topic topic1 --from-beginning 

#（5）删除topic命令： 
kafka-topics.sh --zookeeper 192.168.11.111:2181 --delete --topic topic1
```


## Kafka海量日志收集实战_filebeat日志收集实战

#### FileBeat：安装入门、配置文件、对接Kafka、实战应用

- filebeat安装

```
# 解压到/user/local
tar -zxvf filebeat-6.6.0-linux-x86_64.tar.gz -C /usr/local/


cd /usr/local/filebeat-6.6.0

# 修改配置
vim filebeat.yml

```

- 修改配置 

```
filebeat.prospectors:
- input_type: log
  paths: # 日志路径
    ## app-服务名称.log，为什么写死？防止发生轮转抓取历史数据
    - /Users/tuyuankun/Develop/1297-kafka/logs/app-kafka-log-collect.log
  # 定义写入ES时的_type值
  document_type: "app-log"
  multiline:
    # pattern: '^\s*(\d{4}|\d{2})\-(\d{2}|[a-zA-Z]{3})\-(\d{2}|\d{4})' # 指定匹配的表达式（配置）
    pattern: '^\['      # 指定配置的表达式（匹配以"{开头的字符串）
    nagate: true        # 是否匹配到
    match: after        # 合并到上一行的末尾
    max_lines: 2000     # 最大的行数
    timeout: 2s         # 如果在规定时间没有新的日志事件就不等待后面的日志，把已收集到的就推送到其他地方
  fields:
    logbiz: kafka-log-collect   # 应用名称
    logtopic: app-kafka-log-collect # 按服务划分用作kafka topic
    evn: dev

- input_type: log
  paths: # 日志路径
    - /Users/tuyuankun/Develop/1297-kafka/logs/error-kafka-log-collect.log
  document_type: "error-log"
  multiline:
    # pattern: '^\s*(\d{4}|\d{2})\-(\d{2}|[a-zA-Z]{3})\-(\d{2}|\d{4})' # 指定匹配的表达式（配置）
    pattern: '^\['      # 指定配置的表达式（匹配以"{开头的字符串）
    nagate: true        # 是否匹配到
    match: after        # 合并到上一行的末尾
    max_lines: 2000     # 最大的行数
    timeout: 2s         # 如果在规定时间没有新的日志事件就不等待后面的日志，把已收集到的就推送到其他地方
  fields:
    logbiz: kafka-log-collect   # 应用名称
    logtopic: error-kafka-log-collect # 按服务划分用作kafka topic
    evn: dev

output.kafka:
  enabled: true
  hosts: ["localhost:9092"]
  topic: '%{[fields.logtopic]}'
  partition.hash:
    reachable_only: true
  compression: gzip
  max_message_bytes: 1000000
  required_acks: 1
logging.to_files: true
```

- 检查配置是否正确

```

## 检查配置是否正确
cd /usr/local/filebeat-6.6.0
./filebeat -c filebeat.yml -configtest
## Config OK

## 启动filebeat
/usr/local/filebeat-6.6.0/filebeat &
ps -ef | grep filebeat

```
## Kafka海量日志收集实战_filebeat日志收集实战-2

#### 前提启动zookeeper kafka

- 创建两个主题

```
./kafka-topics.sh --create --zookeeper 127.0.0.1:2181 --replication-factor 1 --partitions 1 --topic app-log-collector

./kafka-topics.sh --create --zookeeper 127.0.0.1:2181 --replication-factor 1 --partitions 1 --topic error-log-collector

# 查看topic情况
kafka-topics --zookeeper localhost:2181 --describe --topic app-log-collector

# 查看kafka是否有数据

# 查看kafka配置文件的log.dirs属性，即获取kafka日志文件夹路径
cat /usr/local/etc/kafka/server.properties 
# log.dirs=/usr/local/var/lib/kafka-logs
# 进入partition对应的文件夹下查看
cd /usr/local/var/lib/kafka-logs/app-kafka-log-collect-0/
```


## Kafka海量日志收集实战_logstash日志过滤实战-1

#### logstash：安装入门、ELK环境搭建、基础语法、实战应用

- 安装配置
```
# 解压安装
tar -zxvf logstash-6.6.0.tar.gz -C /usr/local/

## conf下配置文件说明：
# logstash配置文件：/config/logstash.yml
# JVM参数文件：/config/jvm.options
#  日志格式配置文件：log4j2.properties
#  制作Linux服务参数：/config/startup.options


vim /usr/local/logstash-6.6.0/config/logstash.yml
## 增加workers工作线程数 可以有效的提升logstash性能
pipeline.workers: 16 ## 测试环境默认就行

# 下个步骤
cd /usr/local/logstash-6.6.0/
# 创建文件夹
mkdir script
# 创建配置文件 
touch logstash-script.conf


## 启动logstash
nohup /usr/local/logstash-6.6.0/bin/logstash -f /usr/local/logstash-6.6.0/script/logstash-script.conf &
```

- 修改配置

```

## logstash-script.conf
## multiline 插件也可以用于其他类似的堆栈式信息，比如，Linux的内核日志
input {
  kafka {
    ## app-log-服务名称
    topics_pattern => "app-kafka-log-.*" ## 接收哪些topic下的消息
    bootstrap_servers => "localhost:9092" ## kafka server
    codec => json
    consumer_threads => 1 ## 增加consumer的并行消费线程数，一个partition对应一个consumer_thread
    decorate_events => true
    # auto_offset_rest => "latest"
    group_id => "app-kafka-log-group" ## kafka组
  }

  kafka {
    ## error-log-服务名称
    topics_pattern => "error-kafka-log-.*"
    bootstrap_servers => "localhost:9092"
    codec => json
    consumer_threads => 4
    decorate_events => true
    # auto_offset_rest => "latest"
    group_id => "error-kafka-log-group"
  }
}

filter {
  ## 时区转换
  ruby {
    code => "event.set('index_time', event.timestamp.time.localtime.strftime('%Y.%m.%d'))"
  }

  if "app-kafka-log-collect" in [fields][logtopic] { ## `[fields][logtopic]`为filebeat配置文件`filebeat.yml`的属性
    grok {
      ## 表达式
      match => ["message", "\[%{NOTSPACE:currentDateTime}\] \[%{NOTSPACE:level}\] \[%{NOTSPACE:thread-id}\] \[%{NOTSPACE:class}\] \[%{DATA:hostName}\] \[%{DATA:ip}\] \[%{DATA:applicationName}\] \[%{DATA:location}\] \[%{DATA:messageInfo}\] ## (\'\'|%{QUOTEDSTRING:throwable})"]
    }
  }

  if "error-kafka-log-collect" in [fields][logtopic] {
    grok {
      ## 表达式
      match => ["message", "\[%{NOTSPACE:currentDateTime}\] \[%{NOTSPACE:level}\] \[%{NOTSPACE:thread-id}\] \[%{NOTSPACE:class}\] \[%{DATA:hostName}\] \[%{DATA:ip}\] \[%{DATA:applicationName}\] \[%{DATA:location}\] \[%{DATA:messageInfo}\] ## (\'\'|%{QUOTEDSTRING:throwable})"]
    }
  }
}

## 测试输出到控制台
output {
  stdout {codec => rubydebug}
}
```

## 安装ES Kibana

#### es 安装相关
```
tar -zxvf elasticsearch-6.6.0.tar.gz -C /usr/local/
# 创建用户组和用户
groupadd elsearch
# 设置密码
useradd elsearch -g elsearch -p 123456
# 修改权限
chown -R elsearch:elsearch elasticsearch-6.6.0

# 切换用户并启动 elasticsearch
su elsearch

# 前台启动，接 ctrl + c 停止elasticsearch服务
./elasticsearch 
# 后台启动
./elasticsearch -d 

#本地 curl 测试
curl 127.0.0.1:9200
```

- 修改配置文件

```
vim elasticsearch.yml

path.data: /usr/local/elasticsearch-6.6.0/data
path.logs: /usr/local/elasticsearch-6.6.0/logs
network.host: 0.0.0.0

# 切换到root用户下去修改配置如下

vim /etc/security/limits.conf
* soft nofile 65536
* hard nofile 131072
* soft nproc 2048
* hard nproc 4096

vim /etc/sysctl.conf
# 添加以下内容
vm.max_map_count=262145
sysctl -p 刷新一下

# 再次启动
```

---

#### 安装 kibana

```
tar -zxvf kibana-6.6.0-linux-x86_64.tar.gz -C /usr/local/

cd kibana-6.6.0-linux-x86_64/
cd config/

vim kibana.yml

server.host: "192.168.245.133"
elasticsearch.username: "elsearch"
elasticsearch.password: "123456"
i18n.locale: "zh-CN"

#启动 kibana
cd kibana-6.6.0-linux-x86_64/bin/
 # 前台启动，接 ctrl + c 停止
./kibana 
# 后台启动
./kibana &  

```

- 启动ElasticSearch、Kibana

```
192.168.245.133:5601
```
#### 日志发送到kafka

```
## logstash-script.conf
## elasticsearch
output {
  if "app-kafka-log-collect" in [fields][logtopic] {
    ## es插件
    elasticsearch {
      # es服务地址
      hosts => ["localhost:9200"]
      # 用户名密码
      user => "elastic"
      password => "123456"
      ## 索引名，+号开头的，就会自动认为后面是时间格式：
      ## javalog-app-server-2019.01.23
      index => "app-log-%{[fields][logbiz]}-%{index_time}"
      ## 是否嗅探集群ip：一般设置true；http://localhost:9200/_nodes/http?pretty
      ## 通过嗅探机制进行es集群负载均衡发日志消息
      sniffing => true
      ## logstash默认自带一个mapping模板，进行模板覆盖
      template_overwrite => true
    }
  }
  if "error-kafka-log-collect" in [fields][logtopic] {
    elasticsearch {
      hosts => ["localhost:9200"]
      user => "elastic"
      password => "123456"
      index => "error-log-%{[fields][logbiz]}-%{index_time}"
      sniffing => true
      template_overwrite => true
    }
  }
}
```

## 部署项目产生日志

#### springboot java -jar xxx 没有主清单属性


![image.png](https://upload-images.jianshu.io/upload_images/4994935-d3289ce5519f4c94.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![image.png](https://upload-images.jianshu.io/upload_images/4994935-09eecf24e8b55902.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


```
<build>
		<finalName>collector</finalName>
    	<!-- 打包时包含properties、xml -->
    	<resources>
             <resource>  
                <directory>src/main/java</directory>  
                <includes>  
                    <include>**/*.properties</include>  
                    <include>**/*.xml</include>  
                </includes>  
                <!-- 是否替换资源中的属性-->  
                <filtering>true</filtering>  
            </resource>  
            <resource>  
                <directory>src/main/resources</directory>  
            </resource>      	
    	</resources>		
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
				<configuration>
					<mainClass>com.bfxy.collector.Application</mainClass>
				</configuration>
			</plugin>
		</plugins>
	</build> 	
	
```



## Kafka海量日志收集实战_watcher监控告警实战-1

```
@Data
public class AccurateWatcherMessage {
    private String title;
    private String application;
    private String level;
    private String body;
    private String executionTime;
}
```
---

```
@PostMapping("accurateWatch")
public String accurateWatch(AccurateWatcherMessage accurateWatcherMessage) {
    System.out.println("-------警告内容-------" + accurateWatcherMessage);
    return "is watched" + accurateWatcherMessage;
}
```

## Kafka海量日志收集实战_watcher监控告警实战-2

#### 创建Watcher之前手动指定创建模板

```
// PUT _template/error-log-
{
  "template": "error-log-*",
  "order": 0,
  "settings": {
    "index": {
      "refresh_interval": "5s"
    }
  },
  "mappings": {
    "dynamic_templates": [
      {
        "message_field": {
          "match_mapping_type": "string",
          "path_match": "message",
          "mapping": {
            "norms": false,
            "type": "text",
            "analyzer": "ik_max_word",
            "search_analyzer": "ik_max_word"
          }
        }
      },
      {
        "throwable_field": {
          "match_mapping_type": "string",
          "path_match": "throwable",
          "mapping": {
            "norms": false,
            "type": "text",
            "analyzer": "ik_max_word",
            "search_analyzer": "ik_max_word"
          }
        }
      },
      {
        "string_field": {
          "match_mapping_type": "string",
          "path_match": "*",
          "mapping": {
            "norms": false,
            "type": "text",
            "analyzer": "ik_max_word",
            "search_analyzer": "ik_max_word",
            "fields": {
              "keyword": {
                "type": "keyword"
              }
            }
          }
        }
      }
    ],
    "properties": {
      "hostName": {
        "type": "keyword"
      },
      "ip": {
        "type": "ip"
      },
      "level": {
        "type": "keyword"
      },
      "currentDateTime": {
        "type": "date"
      }
    }
  }
}

```
#### 创建一个Watcher，比如定义一个trigger，每5秒钟看一下input里的数据

```
// PUT _xpack/watcher/watch/error_log_collector_watcher
{
  "trigger": {
    "schedule": {
      "interval": "5s"
    }
  },
  "input": {
    "search": {
      "request": {
        "indices": ["<error-log-{now+8h/d}>"],
        "body": {
          "size": 0,
          "query": {
            "bool": {
              "must": [
                {
                  "term": {"level": "ERROR"}
                }
              ],
              "filter": {
                "range": {
                  "currentDateTime": {
                    "gt": "now-30s", "lt": "now"
                  }
                }
              }
            }
          }
        }
      }
    }
  },
  "condition": {
    "compare": {
      "ctx.payload.hits.total": {
        "gt": 0
      }
    }
  },
  "transform": {
    "search": {
      "request": {
        "indices": ["error-log-{now+8h/d}"],
        "body": {
          "size": 1,
          "query": {
            "bool": {
              "must": [
                {
                  "term": {"level": "ERROR"}
                }
              ],
              "filter": {
                "range": {
                  "currentDateTime": {
                    "gt": "now-30s", "lt": "now"
                  }
                }
              }
            }
          },
          "sort": [
            {
              "currentDateTime": {
                "order": "desc"
              }
            }
          ]
        }
      }
    }
  },
  "actions": {
    "test_error": {
      "webhook": {
        "method": "POST",
        "url": "http://localhost:8081/kafka/index/accurateWatch",
        "body": "{\"title\": \"异常错误告警\", \"applicationName\": \"{{#ctx.payload.hits.hits}}{{_source.applicationName}}{{/ctx.payload.hits.hits}}\", \"level\": \"告警级别P1\", \"body\": \"{{#ctx.payload.hits.hits}}{{_source.messageInfo}}{{/ctx.payload.hits.hits}}\", \"executionTime\": \"{{#ctx.payload.hits.hits}}{{_source.currentDateTime}}{{/ctx.payload.hits.hits}}\"}"
      }
    }
  }
}
```

### 总结图

![image.png](https://upload-images.jianshu.io/upload_images/4994935-ed92cf120efc8232.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


####  [参考文章](https://www.tuyrk.cn/imooc/1297-MQ-Kafka/04-kafka-log-collect/)
