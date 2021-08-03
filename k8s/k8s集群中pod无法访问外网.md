# k8s笔记

## k8s集群中pod无法访问外网

### 问题

使用`kubeadm init --pod-network-cidr=10.224.0.0/16 --service-cidr=10.96.0.0/12`创建了一个k8s集群，之后使用`kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml`安装了网络插件，集群成功创建。

创建一个busybox pod，进入到pod中，通过ping测试能否访问外网，结果发现不能。

```bash
instance-master::root ➜  ~ kubectl run -i -t busybox --image=busybox
If you don't see a command prompt, try pressing enter.
/ # ping -c 4 google.com
PING google.com (142.250.204.46): 56 data bytes
^C
--- google.com ping statistics ---
4 packets transmitted, 0 packets received, 100% packet loss
/ # exit
Session ended, resume using 'kubectl attach busybox -c busybox -i -t' command when the pod is running
```

### 分析

查看当前节点pod网段10.224.0.0/24。

```bash
instance-master::root ➜  ~ kubectl describe node node-name | grep PodCIDR
PodCIDR:                      10.224.0.0/24
PodCIDRs:                     10.224.0.0/24
```

查看当前防火墙策略，找不到与10.224.0.0/24相关的策略，却看到了10.244.0.0/24相关的策略。

```bash
instance-master::root ➜  ~ iptables -t nat -nvL POSTROUTING --line
Chain POSTROUTING (policy ACCEPT 16 packets, 960 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1    10562  637K KUBE-POSTROUTING  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes postrouting rules */
2        0     0 MASQUERADE  all  --  *      !docker0  172.17.0.0/16        0.0.0.0/0
3    10875  656K POSTROUTING_direct  all  --  *      *       0.0.0.0/0            0.0.0.0/0
4    10875  656K POSTROUTING_ZONES_SOURCE  all  --  *      *       0.0.0.0/0            0.0.0.0/0
5    10875  656K POSTROUTING_ZONES  all  --  *      *       0.0.0.0/0            0.0.0.0/0
6        0     0 RETURN     all  --  *      *       10.244.0.0/16        10.244.0.0/16
7        0     0 MASQUERADE  all  --  *      *       10.244.0.0/16       !224.0.0.0/4
8      586 35307 RETURN     all  --  *      *      !10.244.0.0/16        10.224.0.0/24
9        0     0 MASQUERADE  all  --  *      *      !10.244.0.0/16        10.244.0.0/16
```

针对网段10.224.0.0/24添加相应的防火墙策略。

```bash
instance-master::root ➜  ~ iptables -t nat -I POSTROUTING -s 10.224.0.0/24 -j MASQUERADE
instance-master::root ➜  ~ iptables -t nat -nvL POSTROUTING --line
Chain POSTROUTING (policy ACCEPT 16 packets, 960 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1        4   240 MASQUERADE  all  --  *      *       10.224.0.0/24        0.0.0.0/0
2    10562  637K KUBE-POSTROUTING  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes postrouting rules */
3        0     0 MASQUERADE  all  --  *      !docker0  172.17.0.0/16        0.0.0.0/0
4    10875  656K POSTROUTING_direct  all  --  *      *       0.0.0.0/0            0.0.0.0/0
5    10875  656K POSTROUTING_ZONES_SOURCE  all  --  *      *       0.0.0.0/0            0.0.0.0/0
6    10875  656K POSTROUTING_ZONES  all  --  *      *       0.0.0.0/0            0.0.0.0/0
7        0     0 RETURN     all  --  *      *       10.244.0.0/16        10.244.0.0/16
8        0     0 MASQUERADE  all  --  *      *       10.244.0.0/16       !224.0.0.0/4
9      586 35307 RETURN     all  --  *      *      !10.244.0.0/16        10.224.0.0/24
10       0     0 MASQUERADE  all  --  *      *      !10.244.0.0/16        10.244.0.0/16
```

再次进入pod中进行测试，可以正常与外网进行通信。

```bash
instance-master::root ➜  ~ kubectl attach busybox -c busybox -i -t
If you don't see a command prompt, try pressing enter.
/ # ping -c 4 google.com
PING google.com (142.250.204.46): 56 data bytes
64 bytes from 142.250.204.46: seq=0 ttl=121 time=1.599 ms
64 bytes from 142.250.204.46: seq=1 ttl=121 time=1.453 ms
64 bytes from 142.250.204.46: seq=2 ttl=121 time=1.484 ms
64 bytes from 142.250.204.46: seq=3 ttl=121 time=1.527 ms

--- google.com ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 1.453/1.515/1.599 ms
/ # exit
Session ended, resume using 'kubectl attach busybox -c busybox -i -t' command when the pod is running
# 保存防火墙规则
instance-master::root ➜  ~ iptables-save
```

### 原因

Kubeadm初始化k8s集群指定的`--pod-network-cidr`网段与网络插件flannel配置的`"Network"`网段不一致，会导致出现网络不通的情况。

### 解决方法

#### 1. 临时解决

通过在每台节点配置相应的防火墙策略，实现网络通信。

查看当前节点PodCIDR，将node- name改成自己的主机名。

```bash
instance-master::root ➜  ~ kubectl describe node node-name | grep PodCIDR
PodCIDR:                      10.224.0.0/24
PodCIDRs:                     10.224.0.0/24
```

根据当前节点PodCIDR配置防火墙规则。

```bash
iptables -t nat -I POSTROUTING -s 10.224.0.0/24 -j MASQUERADE
```

保存防火墙规则。

```bash
instance-master::root ➜  ~ iptables-save
```

#### 2.永久解决

修改flannel的网段与当前节点PodCIDR一样。

读取configmap内容并保存到本地（也可以到官方下载对应版本的[kube-flannel.yml](https://github.com/flannel-io/flannel/blob/master/Documentation/kube-flannel.yml)）。

```bash
kubectl get configmap kube-flannel-cfg -n kube-system -o yaml > kube-flannel.yaml
```

编辑kube-flannel.yaml，将：

```yaml
...
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
...
```

修改为

```yaml
...
  net-conf.json: |
    {
      "Network": "10.224.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
...
```

应用配置

```bash
kubectl apply -f flannel.yaml
```

更新配置之后重启集群。

#### 3.初始化集群的过程中

在初始化集群和安装网络插件的时候，保证`--pod-network-cidr`网段与网络插件flannel配置的`"Network"`网段一致。