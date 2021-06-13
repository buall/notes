# Docker笔记

## Docker环境下配置iptables规则无效

### 场景

在Linux服务器 (192.168.33.8)用Docker部署服务，以MySQL为例，通过`-p`将容器的端口3306映射到宿主机的8006。

```bash
docker run --name mysql --restart always -p 8006:3306 -e LANG=C.UTF-8 -e MYSQL_ROOT_PASSWORD=123456 -d mysql
```

使用iptables限制连接MySQL，这里向`INPUT`链中插入一条DROP规则拒绝来自192.168.33.0/24的主机访问宿主机的8006。

```bash
 ➜  ~ iptables -I INPUT -p tcp --dport 8006 -s 192.168.33.0/24 -j DROP
 ➜  ~ iptables -nvL INPUT --line-numbers
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1        0     0 DROP       tcp  --  *      *       191.168.33.0/24      0.0.0.0/0            tcp dpt:8006
2      411 28776 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED
3        0     0 ACCEPT     icmp --  *      *       0.0.0.0/0            0.0.0.0/0
4        0     0 ACCEPT     all  --  lo     *       0.0.0.0/0            0.0.0.0/0
5        0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            state NEW tcp dpt:22
6        0     0 REJECT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            reject-with icmp-host-prohibited
```

添加规则之后，在本机进行测试仍然可以连通192.168.33.8:8006，防火墙规则没生效。

```bash
 Tearth-MBP:~ tearth  ~  telnet 192.168.33.8 8006
Trying 192.168.33.8...
Connected to example.com.
Escape character is '^]'.
J
8.0.258bKQ=+�DzLWtOAw5*%caching_sha2_password
```

在服务器查看刚添加的这条规则(num: 1)，没有任何流量经过这条规则，再次确定了这条规则没有生效。

```bash
 ➜  ~ iptables -nvL INPUT --line-numbers
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1        0     0 DROP       tcp  --  *      *       192.168.33.0/24      0.0.0.0/0            tcp dpt:8006
2     2333  163K ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED
3        0     0 ACCEPT     icmp --  *      *       0.0.0.0/0            0.0.0.0/0
4        0     0 ACCEPT     all  --  lo     *       0.0.0.0/0            0.0.0.0/0
5        0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            state NEW tcp dpt:22
6       64  4992 REJECT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            reject-with icmp-host-prohibited
```

### 原因

在 Linux 上，Docker 操纵`iptables`规则来提供网络隔离，Docker 安装了两个自定义 iptables 链`DOCKER-USER`和 的`DOCKER`，并确保传入的数据包始终首先由这两个链检查。如果通过 Docker 公开一个端口，无论防火墙配置了什么规则，这个端口都会被公开。如果希望即使在端口通过 Docker 公开时也应用这些规则，则必须将这些规则添加到 `DOCKER-USER`链中。具体内容参考官方文档[Docker and iptables](https://docs.docker.com/network/iptables/)，

### 解决

根据Docker官方文档，可以通过在`DOCKER-USER`链中添加相应的规则控制。（⚠️ 在`DOCKER-USER`中配置规则，端口要写Docker容器监听的端口3306，不能配置映射到宿主机的端口8006）

```bash
iptables -I DOCKER-USER -p tcp --dport 3306 -s 192.168.33.0/24 -j DROP
```

再次测试防火墙规则是否生效，

```bash
 Tearth-MBP:~ tearth  ~  telnet 192.168.33.8 8006
Trying 192.168.33.8...
telnet: connect to address 192.168.33.8: Operation timed out
telnet: Unable to connect to remote host
```

查看添加的规则 （num: 1)，可以看到有流量经过这条规则，有11个数据包总计688bytes，由此可以确定规则生效。

```bash
 ➜  ~ iptables -nvL DOCKER-USER --line-numbers
Chain DOCKER-USER (1 references)
num   pkts bytes target     prot opt in     out     source               destination
1       11   688 DROP       tcp  --  *      *       192.168.33.0/24      0.0.0.0/0            tcp dpt:3306
2    37631 8191K RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0
```

