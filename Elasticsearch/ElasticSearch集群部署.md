# ElasticSearch集群部署
## 基础环境&准备工作
1. 三台Linux主机, 操作系统Centos（也可一台主机配置不同端口, 操作系统也不作限制)
2. 安装docker, docker-compose
3. 修改虚拟内存

临时修改

```bash
sysctl -w vm.max_map_count=262144
```

写入到配置文件

```bash
echo ‘vm.max_map_count=262144’ >> /etc/sysctl.conf
```

### 下载相关文件
```bash
git clone https://github.com/deviantony/docker-elk.git
cd docker-elk
# 切换到tls分支
git checkout tls
```
当前目录结构如下
```bash
.
├── docker-compose.yml
├── docker-stack.yml
├── elasticsearch
├── extensions
├── kibana
├── LICENSE
├── logstash
├── README.md
└── tls
```

### 生成证书

接着上一步, 进入`tls`目录, 可以看到`tls`目录已经有现成的证书, 根据README可知这些证书都是可以直接使用的, 有效期是10年，<u>但是不建议生产环境使用这些现成的证书</u>, 所以在生产环境需要自己生成证书, 后面我们会用自己生成的证书替换掉`tls`目录中的这些文件。
```bash
.
├── ca
│   └── ca.p12
├── elasticsearch
│   ├── elasticsearch.p12
│   ├── http.p12
│   ├── README.txt
│   └── sample-elasticsearch.yml
├── instances.yml
├── kibana
│   ├── elasticsearch-ca.pem
│   ├── README.txt
│   └── sample-kibana.yml
└── README.md
```

执行下面命令，会在当前目录生成一个certificate-bundle.zip的压缩包
```bash
docker run -it \
  -v ${PWD}:/usr/share/elasticsearch/tls \
  docker.elastic.co/elasticsearch/elasticsearch:7.16.0 \
  bin/elasticsearch-certutil cert \
    --days 3650 \
    --keep-ca-key \
    --in tls/instances.yml \
    --out tls/certificate-bundle.zip
```

```bash
Enter password for Generated CA : <None> # CA证书的密码, 除非有必要, 建议留空
Enter password for elasticsearch/elasticsearch.p12 : <None> # CA证书的密码, 除非有必要, 建议留空

Certificates written to /usr/share/elasticsearch/tls/certificate-bundle.zip
```

覆盖解压certificate-bundle.zip

```bash
unzip -o certificate-bundle.zip
```
```bash
docker run -it \
  -v ${PWD}:/usr/share/elasticsearch/tls \
  docker.elastic.co/elasticsearch/elasticsearch:7.16.0 \
  bin/elasticsearch-certutil http
```
```bash
## Do you wish to generate a Certificate Signing Request (CSR)?
# 
Generate a CSR? [y/N]n 

## Do you have an existing Certificate Authority (CA) key-pair that
# 是否使用已有的CA证书
Use an existing CA? [y/N]y


## What is the path to your CA?
# 已有CA证书路径, 这里指向上一步生成的证书ca证书
CA Path: /usr/share/elasticsearch/tls/ca/ca.p12


## How long should your certificates be valid?
# 证书有效期, 默认5年, 可以自定义，这里选择生成10年
For how long should your certificate be valid? [5y] 10y

## Do you wish to generate one certificate per node?
# 是否为每个节点生成证书, 选择否
Generate a certificate per node? [y/N]n

## Which hostnames will be used to connect to your nodes?
# 节点主机名, 
elasticsearch
localhost

You entered the following hostnames.

 - elasticsearch
 - localhost

Is this correct [Y/n]y

## Which IP addresses will be used to connect to your nodes?
# 节点IP
<None>

You did not enter any IP addresses.

Is this correct [Y/n]y

## Other certificate options
# 是否更改以上信息

Do you wish to change any of these options? [y/N]n

## What password do you want for your private key(s)?
# 私钥密码, 留空
If you wish to use a blank password, simply press <enter> at the prompt below.
Provide a password for the “http.p12” file:  [<ENTER> for none]

## Where should we save the generated files?
# 生成证书目录 
What filename should be used for the output zip file? [/usr/share/elasticsearch/elasticsearch-ssl-http.zip] tls/elasticsearch-ssl-http.zip
```

覆盖解压elasticsearch-ssl-http.zip

```bash
unzip -o elasticsearch-ssl-http.zip
Archive:  elasticsearch-ssl-http.zip
  inflating: elasticsearch/README.txt
  inflating: elasticsearch/http.p12
  inflating: elasticsearch/sample-elasticsearch.yml
  inflating: kibana/README.txt
  inflating: kibana/elasticsearch-ca.pem
  inflating: kibana/sample-kibana.yml
```

当前`tls`目录结构, 目录结构跟之前并没有什么变化, 但是证书已经更新。

```bash
.
├── ca
│   └── ca.p12
├── elasticsearch
│   ├── elasticsearch.p12
│   ├── http.p12
│   ├── README.txt
│   └── sample-elasticsearch.yml
├── instances.yml
├── kibana
│   ├── elasticsearch-ca.pem
│   ├── README.txt
│   └── sample-kibana.yml
└── README.md
```
## 单机（单节点）模式

### 修改配置文件

#### docker-compose.yml

根据服务器配置和实际需求修改`ES_JAVA_OPTS`, `ELASTIC_PASSWORD`,`LS_JAVA_OPTS`的值

```yaml
version: ‘3.2’

services:
  elasticsearch:
    build:
      context: elasticsearch/
      args:
        ELK_VERSION: $ELK_VERSION
    volumes:
      - type: bind
        source: ./elasticsearch/config/elasticsearch.yml
        target: /usr/share/elasticsearch/config/elasticsearch.yml
        read_only: true
      - type: volume
        source: elasticsearch
        target: /usr/share/elasticsearch/data
      # (!) TLS certificates. Generate using instructions from tls/README.md.
      - type: bind
        source: ./tls/elasticsearch/elasticsearch.p12
        target: /usr/share/elasticsearch/config/elasticsearch.p12
        read_only: true
      - type: bind
        source: ./tls/elasticsearch/http.p12
        target: /usr/share/elasticsearch/config/http.p12
        read_only: true
    ports:
      - “9200:9200”
      - “9300:9300”
    environment:
      TZ: Asia/Shanghai
      ES_JAVA_OPTS: “-Xmx8g -Xms8g”
      ELASTIC_PASSWORD: mypassword
      discovery.type: single-node
    networks:
      - elk

  logstash:
    build:
      context: logstash/
      args:
        ELK_VERSION: $ELK_VERSION
    volumes:
      - type: bind
        source: ./logstash/config/logstash.yml
        target: /usr/share/logstash/config/logstash.yml
        read_only: true
      - type: bind
        source: ./logstash/pipeline
        target: /usr/share/logstash/pipeline
        read_only: true
      # (!) CA certificate. Generate using instructions from tls/README.md
      - type: bind
        source: ./tls/kibana/elasticsearch-ca.pem
        target: /usr/share/logstash/config/elasticsearch-ca.pem
        read_only: true
    ports:
      - “5044:5044”
      - “5000:5000/tcp”
      - “5000:5000/udp”
      - “9600:9600”
    environment:
      TZ: Asia/Shanghai
      LS_JAVA_OPTS: “-Xmx4g -Xms4g”
    networks:
      - elk
    depends_on:
      - elasticsearch

  kibana:
    build:
      context: kibana/
      args:
        ELK_VERSION: $ELK_VERSION
    volumes:
      - type: bind
        source: ./kibana/config/kibana.yml
        target: /usr/share/kibana/config/kibana.yml
        read_only: true
      # (!) CA certificate. Generate using instructions from tls/README.md.
      - type: bind
        source: ./tls/kibana/elasticsearch-ca.pem
        target: /usr/share/kibana/config/elasticsearch-ca.pem
        read_only: true
    environment:
      TZ: Asia/Shanghai
    ports:
      - “5601:5601”
    networks:
      - elk
    depends_on:
      - elasticsearch

networks:
  elk:
    driver: bridge

volumes:
  elasticsearch:
```

#### elasticsearch/config/elasticsearch.yml

建议修改`xpack.license.self_generated.type: trial`为`xpack.license.self_generated.type: basic`, 这里直接选择使用基础免费的License。

```yaml
cluster.name: docker-cluster
network.host: 0.0.0.0
xpack.license.self_generated.type: basic
xpack.security.enabled: true
xpack.monitoring.collection.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: elasticsearch.p12
xpack.security.transport.ssl.truststore.path: elasticsearch.p12
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: http.p12
```

#### kibana/config/kibana.yml

`elasticsearch.password`要跟docker-compose.yml中的配置的   `ELASTIC_PASSWORD`一致。

```yaml
server.name: kibana
server.host: 0.0.0.0
elasticsearch.hosts:
  - ‘https://elasticsearch:9200’
monitoring.ui.container.elasticsearch.enabled: true
server.basePath: /kibana
server.publicBaseUrl: ‘http://elk.com/kibana’
i18n.locale: zh-CN
elasticsearch.username: elastic
elasticsearch.password: mypassword
elasticsearch.ssl.certificateAuthorities:
  - config/elasticsearch-ca.pem
```

### 启动

到docker-compose.yml文件所在目录，执行`docker-compose up -d`启动相应服务。

## 集群模式

这里创建一个ELK集群, 在节点1部署Elasticsearch, Logstash, Kibana; 在节点2和节点3部署Elasticsearch。三个Elasticsearch都为master节点, 都可以存储数据。

| Hostname | IP        | Service                         |
| -------- | --------- | ------------------------------- |
| master-1 | 10.0.0.10 | Elasticsearch, Logstash, Kibana |
| master-2 | 10.0.0.11 | Elasticsearch                   |
| master-3 | 10.0.0.12 | Elasticsearch                   |


### 修改配置文件

#### docker-compose.yml

根据服务器配置和实际需求修改`ES_JAVA_OPTS`, `ELASTIC_PASSWORD`,`LS_JAVA_OPTS`的值

* 节点1
```yaml
version: ’3.2’

services:
  elasticsearch:
    build:
      context: elasticsearch/
      args:
        ELK_VERSION: $ELK_VERSION
    volumes:
      - type: bind
        source: ./elasticsearch/config/elasticsearch.yml
        target: /usr/share/elasticsearch/config/elasticsearch.yml
        read_only: true
      - type: volume
        source: elasticsearch
        target: /usr/share/elasticsearch/data
      # (!) TLS certificates. Generate using instructions from tls/README.md.
      - type: bind
        source: ./tls/elasticsearch/elasticsearch.p12
        target: /usr/share/elasticsearch/config/elasticsearch.p12
        read_only: true
      - type: bind
        source: ./tls/elasticsearch/http.p12
        target: /usr/share/elasticsearch/config/http.p12
        read_only: true
    ports:
      - “9200:9200”
      - “9300:9300”
    environment:
      TZ: Asia/Shanghai
      ES_JAVA_OPTS: “-Xmx8g -Xms8g”
      ELASTIC_PASSWORD: mypassword
    networks:
      - elk

  logstash:
    build:
      context: logstash/
      args:
        ELK_VERSION: $ELK_VERSION
    volumes:
      - type: bind
        source: ./logstash/config/logstash.yml
        target: /usr/share/logstash/config/logstash.yml
        read_only: true
      - type: bind
        source: ./logstash/pipeline
        target: /usr/share/logstash/pipeline
        read_only: true
      # (!) CA certificate. Generate using instructions from tls/README.md
      - type: bind
        source: ./tls/kibana/elasticsearch-ca.pem
        target: /usr/share/logstash/config/elasticsearch-ca.pem
        read_only: true
    ports:
      - “5044:5044”
      - “5000:5000/tcp”
      - “5000:5000/udp”
      - “9600:9600”
    environment:
      TZ: Asia/Shanghai
      LS_JAVA_OPTS: “-Xmx4g -Xms4g”
    networks:
      - elk
    depends_on:
      - elasticsearch

  kibana:
    build:
      context: kibana/
      args:
        ELK_VERSION: $ELK_VERSION
    volumes:
      - type: bind
        source: ./kibana/config/kibana.yml
        target: /usr/share/kibana/config/kibana.yml
        read_only: true
      # (!) CA certificate. Generate using instructions from tls/README.md.
      - type: bind
        source: ./tls/kibana/elasticsearch-ca.pem
        target: /usr/share/kibana/config/elasticsearch-ca.pem
        read_only: true
    environment:
      TZ: Asia/Shanghai
    ports:
      - “5601:5601“ ”但是”
    networks:
      - elk
    depends_on:
      - elasticsearch

networks:
  elk:
    driver: bridge

volumes:
  elasticsearch:
```

* 节点2 && 节点3

```yaml
version: ‘3.2’

services:
  elasticsearch:
    build:
      context: elasticsearch/
      args:
        ELK_VERSION: $ELK_VERSION
    volumes:
      - type: bind
        source: ./elasticsearch/config/elasticsearch.yml
        target: /usr/share/elasticsearch/config/elasticsearch.yml
        read_only: true
      - type: volume
        source: elasticsearch
        target: /usr/share/elasticsearch/data
      # (!) TLS certificates. Generate using instructions from tls/README.md.
      - type: bind
        source: ./tls/elasticsearch/elasticsearch.p12
        target: /usr/share/elasticsearch/config/elasticsearch.p12
        read_only: true
      - type: bind
        source: ./tls/elasticsearch/http.p12
        target: /usr/share/elasticsearch/config/http.p12
        read_only: true
    ports:
      - “9200:9200”
      - “9300:9300”
    environment:
      TZ: Asia/Shanghai
      ES_JAVA_OPTS: “-Xmx8g -Xms8g”
      ELASTIC_PASSWORD: mypassword
    networks:
      - elk

networks:
  elk:
    driver: bridge

volumes:
  elasticsearch:
```

#### elasticsearch/config/elasticsearch.yml
分别修改三个节点的Elasticsearch配置文件, 主要是修改其中的`node.name`和`network.publish_host`, 具体内容如下

* 节点1

```yaml
cluster.name: es-cluster
node.name: master-1
cluster.initial_master_nodes:
  - master-1
  - master-2
  - master-3
node.master: true
node.data: true
node.ingest: true
network.host: 0.0.0.0
network.publish_host: 10.0.0.10
http.port: 9200
transport.port: 9300
discovery.seed_hosts:
  - ’10.0.0.10:9300’
  - ’10.0.0.11:9300’
  - ’10.0.0.12:9300’
xpack.license.self_generated.type: basic
xpack.security.enabled: true
xpack.monitoring.collection.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: elasticsearch.p12
xpack.security.transport.ssl.truststore.path: elasticsearch.p12
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: http.p12
```

* 节点2

```yaml
cluster.name: es-cluster
node.name: master-2
cluster.initial_master_nodes:
  - master-1
  - master-2
  - master-3
node.master: true
node.data: true
node.ingest: true
network.host: 0.0.0.0
network.publish_host: 10.0.0.11
http.port: 9200
transport.port: 9300
discovery.seed_hosts:
  - ’10.0.0.10:9300’
  - ’10.0.0.11:9300’
  - ’10.0.0.12:9300’
xpack.license.self_generated.type: basic
xpack.security.enabled: true
xpack.monitoring.collection.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: elasticsearch.p12
xpack.security.transport.ssl.truststore.path: elasticsearch.p12
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: http.p12
```

* 节点3

```yaml
cluster.name: es-cluster
node.name: master-3
cluster.initial_master_nodes:
  - master-1
  - master-2
  - master-3
node.master: true
node.data: true
node.ingest: true
network.host: 0.0.0.0
network.publish_host: 10.0.0.12
http.port: 9200
transport.port: 9300
discovery.seed_hosts:
  - ’10.0.0.10:9300’
  - ’10.0.0.11:9300’
  - ’10.0.0.12:9300’
xpack.license.self_generated.type: basic
xpack.security.enabled: true
xpack.monitoring.collection.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: elasticsearch.p12
xpack.security.transport.ssl.truststore.path: elasticsearch.p12
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: http.p12
```

#### kibana/config/kibana.yml

`elasticsearch.password`要跟docker-compose.yml中的配置的   `ELASTIC_PASSWORD`一致。

```yaml
server.name: kibana
server.host: 0.0.0.0
elasticsearch.hosts:
  - ‘https://elasticsearch:9200’
monitoring.ui.container.elasticsearch.enabled: true
server.basePath: /kibana
server.publicBaseUrl: ‘http://elk.com/kibana’
i18n.locale: zh-CN
elasticsearch.username: elastic
elasticsearch.password: mypassword
elasticsearch.ssl.certificateAuthorities:
  - config/elasticsearch-ca.pem
```

### 启动

到三台主机docker-compose.yml文件所在目录，执行`docker-compose up -d`启动相应服务。