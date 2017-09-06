# docker-elk
---
项目来源：

* [prima/docker-filebeat](https://github.com/primait/docker-filebeat)：`filebeat`的`docker`镜像
* [deviantony/docker-elk]()：`elasticsearch + kibana + logstash`的镜像及配置，主要参考该项目，说明文档非常详细，issue中作者也有很好地解答。
* [基于ELK+Filebeat搭建日志中心和日志解析](https://github.com/jasonGeng88/blog/blob/master/201703/elk.md)：参考该项目进行`filebeat`的配置以及日志解析的模式
* [Beats官网：Filebeat Prospectors](https://www.elastic.co/guide/en/beats/filebeat/current/configuration-filebeat-options.html)：参考进行`filebeat`的配置
* [logback与logstash的配置文件](https://gist.github.com/kyungw00k/e7b3cee94d9c669e5586)：参考`logstash`中`filter`的`grok`的正则表达式的`pattern`
* [Grok Debugger](http://grokdebug.herokuapp.com/patterns#)：`Grok`正则表达式的学习

---
### 简介
todo




---
### 安装与部署
`elasticsearch + kibana + logstash`安装在一台机器上，`filebeat`可以安装在多台机器上（有日志需要收集的机器上）。

###### elk的安装

进入到docker-elk/elk目录下，执行以下命令：

`docker-compose up -d`

这样就在后台启动了elk，如果不想后台，想看到启动运行情况，可以去掉`-d`参数，执行以下命令：

`docker-compose up`


该命令会默认在当前目录寻找`docker-compose.yml`文件，并以该文件制定的参数进行启动服务。

建议先看一遍该目录下的README说明，原作者讲解的很详细。

打开该文件，其中的几个参数含义是：`build`指定`Dockerfile`位置，`volumes`指定容器内目录与宿主机目录的对应关系，`ports`指定容器端口与宿主机端口的对应关系，`environment`用于设定系统变量。

* 如果想要更改elasticsearch的数据持久化目录，则修改`docker-compose.yml`文件下elasticsearch节点下配置，将`./storage`替换为你想存放elasticsearch的数据的目录即可；如果不需要持久化，将该行配置删除即可。（注意：由于在Elasticsearch镜像中使用了elasticsearch用户，所以，需要将持久化目录的拥有者设置为1000：chown -R 1000 ./storage ）
```
- ./storage:/usr/share/elasticsearch/data
```

* logstash的配置文件主要是docker-elk/elk/logstash/pipeline下的logstash.conf。日志的过滤（filter）在此配置。该文件中，if[type]== "nginx"中type是在filebeat的设置的。即filebeat在收集日志文件往logstash传输时，可以给日志设置type进行分类（当然不止这一种分类方式）。

* 如果出现连接问题，请使用`curl http://localhost:9200`类似curl命令进行测试，排除端口未开放导致的网络连接问题。centos7及以上的防火墙不再使用iptables，而是使用filewall。同时，如果你是云主机，注意查看配置的安全组（只有安全组和防火墙同时允许通过，你的连接才可以通过）。



###### filebeat的安装

进入docker-elk/filebeat目录下，运行
```
docker-compose up
```

即可启动服务。

默认配置的日志目录为docker-elk/filebeat/logs目录（即实际项目输出的日志所在的目录），打开docker-elk/filebeat目录下的docker服务配置文件docker-compose.yml，修改第10行
```
      - ./logs:/logs
```
将`./logs`替换为实际的日志位置，假如你的日志输出在/var/log/java目录下，则修改为：
```
      - /var/log/java:/logs 
```
。

filebeat还可以为每个传输的日志（同一个路径下的文件）加一些标签，通过field属性设置，可以参考配置文件中的写法：增加了app_id和type两个属性。这样在日志达到了logstash之后，就可以根据这些标签进行区分是哪里来的日志。

### 日志解析

* Java日志
如果你可以修改日志的输出格式，且使用的logback作为项目的日志组件，则可以将日志的pattern修改为：
```
 <encoder>
            <pattern>[%d{yyyy-MM-dd HH:mm:ss.SSS}] [${HOSTNAME}] [%thread] %level %logger{36}@%method:%line - %msg%n</pattern>
        </encoder>
```
实际输出如下：
```
[2017-09-05 21:55:43.223] [bogon] [http-nio-8090-exec-5] DEBUG o.s.web.servlet.DispatcherServlet@processRequest:1000 - Successfully completed request
```
则filebeat.yml多行合并可以写为
```  
   multiline: # 多行处理    
        pattern: '^\['
        match: after
        negate: true    
```
对应的logstash的filter中grok正则表达式为：
```
grok {
                    match => { "message" => "(?m)\[%{TIMESTAMP_ISO8601:timestamp}\] \[%{HOSTNAME:host}\] \[%{DATA:thread}\] %{LOGLEVEL:logLevel} %{DATA:class}@%{DATA:method}:%{DATA:line} \- %{GREEDYDATA:message}" }
            }
```
。

如果不能修改项目的日志输出样式，请学习grok的正则表达式，进行解析。

* nginx日志

* mysql慢日志