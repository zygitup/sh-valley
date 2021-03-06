> 全部编排文件和配置文件可以访问我的[Github](https://github.com/cocowool/sh-valley/tree/master/docker-conf/elasticstack/cluster)，大家只要修改配置文件中的路径即可快速搭建一个3实例的ES集群和一个Kibana实例。

## 规划
计划创建3个ES实例组成一个集群，同时创建一个Kibana实例连接该集群。每个ES实例使用本地配置文件，方便配置文件的保存和版本管理。Kibana的配置文件也放在本地，通过文件映射的方式挂载到容器内。

总的目录结构如下：
```sh
$ tree
.
├── docker-compose.yml
├── kibana.yml
├── node1
│   └── es1.yml
├── node2
│   └── es2.yml
└── node3
    └── es3.yml

3 directories, 5 files
```

## 编排文件
主要的编排文件是`docker-compose.yml`
```yaml
version: "2.1"
services:
  es-node1:
    image: docker.elastic.co/elasticsearch/elasticsearch:6.7.0
    hostname: es-node1
    expose:         #不会将端口暴露给容器外应用
      - "9001"
    ports:          #将端口暴露到宿主机中
      - "9200:9200"
      - "9300:9300"
    volumes:
      - ~/Projects/sh-valley/docker-conf/elasticstack/cluster/node1/es1.yml:/usr/share/elasticsearch/config/elasticsearch.yml
    environment:
      - cluster.name=es-cluster
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms256m -Xmx256m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    networks:
      es-cluster-network:
        ipv4_address: 172.21.0.10
  es-node2:
    image: docker.elastic.co/elasticsearch/elasticsearch:6.7.0
    hostname: es-node2
    expose:         #不会将端口暴露给容器外应用
      - "9002"
    ports:          #将端口暴露到宿主机中
      - "9201:9201"
      - "9301:9301"
    volumes:
      - ~/Projects/sh-valley/docker-conf/elasticstack/cluster/node2/es2.yml:/usr/share/elasticsearch/config/elasticsearch.yml
    environment:
      - cluster.name=es-cluster
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms256m -Xmx256m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    networks:
      es-cluster-network:
        ipv4_address: 172.21.0.11
  es-node3:
    image: docker.elastic.co/elasticsearch/elasticsearch:6.7.0
    hostname: es-node3
    expose:         #不会将端口暴露给容器外应用
      - "9003"
    ports:          #将端口暴露到宿主机中
      - "9202:9202"
      - "9302:9302"
    volumes:
      - ~/Projects/sh-valley/docker-conf/elasticstack/cluster/node3/es3.yml:/usr/share/elasticsearch/config/elasticsearch.yml
    environment:
      - cluster.name=es-cluster
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms256m -Xmx256m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    networks:
      es-cluster-network:
        ipv4_address: 172.21.0.12
  kibana:
    image: docker.elastic.co/kibana/kibana:6.7.0
    ports:
      - "5601:5601"
    volumes:
      - ~/Projects/sh-valley/docker-conf/elasticstack/cluster/kibana.yml:/usr/share/kibana/config/kibana.yml
    environment:
      - ELASTICSEARCH_URL=http://es-node1:9200
    networks:
      - es-cluster-network
networks:
  es-cluster-network:
    driver: bridge
    ipam:
      driver: default
      config:
      - subnet: 172.21.0.0/16
        gateway: 172.21.0.1
```

ES配置文件只选取了一个示例如下：
```yaml
cluster.name: elasticsearch-cluster
node.name: es-node1
network.bind_host: 0.0.0.0
network.publish_host: 172.21.0.10
http.port: 9200
transport.tcp.port: 9300
http.cors.enabled: true
http.cors.allow-origin: "*"
node.master: true 
node.data: true  
discovery.zen.ping.unicast.hosts: ["172.21.0.10:9300","172.21.0.11:9301","172.21.0.12:9302"]
discovery.zen.minimum_master_nodes: 2
```

Kibana的配置文件如下
```yaml
server.name: kibana
server.host: "0"
elasticsearch.hosts: [ "http://es-node1:9200", "http://es-node2:9201", "http://es-node3:9202" ]
xpack.monitoring.ui.container.elasticsearch.enabled: false

```

## 启动命令
配置文件准备好之后，就可以启动集群了
```sh
$ docker-compose up -d
```

启动过程可能会比较慢，通过命令行可以看到集群节点
```sh
$ curl http://localhost:9200/_cat/nodes
172.21.0.12 51 96 29 6.53 6.43 3.72 md  - es-node3
172.21.0.11 47 96 30 6.53 6.43 3.72 mdi - es-node2
172.21.0.10 49 96 30 6.53 6.43 3.72 mdi * es-node1
```

后续使用过程中可以通过`docker-compose`命令启动、停止服务，如果不想保留相关的实例，可以使用`docker-compose down`关闭并删除容器。

![](https://images2018.cnblogs.com/blog/39469/201807/39469-20180710163655709-89635310.png)

## 参考资料
1. [elasticsearch document](https://www.elastic.co/guide/en/elasticsearch/reference/current/indexing-buffer.html)
2. [docker-compose自定义网络，固定容器ip地址](https://blog.csdn.net/hechaojie_com/article/details/83625265)
3. [docker-compose ports和expose的区别](https://blog.csdn.net/stinky_kiss/article/details/82563480)