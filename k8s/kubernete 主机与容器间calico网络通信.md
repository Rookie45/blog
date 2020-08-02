**关键词**：*kubernete, calico*

calico是kubernete中的一种网络解决方案，下面主要介绍kubernete中使用calico方案在如下几种情况的如何通信的

1.容器到宿主机的通信

2.同一宿主机内两个容器通信

3.同一网段内两台宿主机上的容器间通信

4.不同网段内两台宿主机上的容器间通信

容器到宿主机的通信，是通过veth pair设备实现两者之间的通信。veth pair被创建出来后，总是以两张网卡（veth peer）形式成对出现，且从其中一张网卡发出去的数据包，会直接出现在另一张网卡上。下面具体看一下怎么回事：

首先我们进入kubernete环境里的某个容器里
```
$ kubectl get pod -n nginx
NAME                                READY     STATUS    RESTARTS   AGE 
nginx-deployment-54d85cb6ff-ktjf8   1/1       Running   0          2h
$ kubectl exec -it nginx-deployment-54d85cb6ff-ktjf8 bash -n nginx
```
查看容器的网络信息以及路由信息
```
root@nginx-deployment-54d85cb6ff-ktjf8:/# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
3: eth0@if101: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1440 qdisc noqueue state UP
    link/ether d2:5c:fe:ca:17:4b brd ff:ff:ff:ff:ff:ff
    inet 177.177.114.10/32 scope global eth0
       valid_lft forever preferred_lft forever
root@nginx-deployment-54d85cb6ff-ktjf8:/# ip route
default via 169.254.1.1 dev eth0
169.254.1.1 dev eth0 scope link
root@nginx-deployment-54d85cb6ff-ktjf8:/# ping 192.168.100.100
64 bytes from 192.168.100.100: icmp_seq=1 ttl=64 time=0.121ms
64 bytes from 192.168.100.100: icmp_seq=1 ttl=64 time=0.088ms
...
```
举例中的宿主机ip是192.168.100.100，可以看到容器并没有到宿主机路由，但能ping通宿主机。当数据包发送的目的ip并不在路由表中时，这类数据包统统交给默认网关处理，在这个例子里就是169.254.1.1。通常一台机器设置的网关ip所属的设备，是与该机器二层相通的，通过MAC寻址通信，二层相通一般的情况是网关设备ip与机器在相同网段，此处明显不在一个网段，那是怎么通的呢？

事实上，一台机器要访问网关设备时，首先会通过ARP获得网关的MAC地址，然后将目标MAC变成网关MAC，而网关IP地址不会在任何网络包头里出现，换句话说，这个网关IP没人关心是多少，只要找到对应的MAC，响应ARP就可以。那在这个环境里，网关的MAC地址知道吗？
```
root@nginx-deployment-54d85cb6ff-ktjf8:/# ip neigh
169.254.1.1 dev eth0 lladdr ee:ee:ee:ee:ee:ee  REACHABLE
```
通过ip neigh命令查看本地ARP缓存，发现网关MAC地址是知道的，也就是说数据包能传给网关，那么数据包就从eth0口出去了，数据包的MAC地址就变成了ee:ee:ee:ee:ee:ee。前面有说过，veth pair设备从一张网卡出去的流量，会出现在成对的另一张网卡上，此刻容器内的eth0出去的流量走给了MAC地址为ee:ee:ee:ee:ee:ee的网关设备，实际这个网关设备就是成对的另一张网卡，而这张网卡被放在宿主机上，那么怎么知道对应到宿主机上的哪张(虚拟)网卡呢
```
root@nginx-deployment-54d85cb6ff-ktjf8:/# cat /sys/devices/virtual/net/eth0/iflink
400
```
通过查看/sys/devices/virtual/net/eth0/iflink的值，发现(虚拟)网卡编号400，退出到宿主机上，通过ip addr命令找到编号400的网卡名
```
$ ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: enp61s0f0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 54:1f:e2:de:b3:2b brd ff:ff:ff:ff:ff:ff 
    inet 192.168.100.100/16 brd 192.168.255.255 scope global noprefixroute enp61s0f0
       valid_lft forever preferred_lft forever
...
400: cali9a4ed7b64a6@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 16
    inet6 fe80::ecee:eeff:feee:eeee/64 scope link
       valid_lft forever preferred_lft forever
...
```
找到400的网卡calicba2f87f6bb，这就是放到宿主机的另一张网卡。我们通过tcpdump命令抓取这个端口进来的icmp包（ping命令产生的是icmp包），确认容器发出的数据包从这个网卡出来的。
```
$ tcpdump icmp -i cali9a4ed7b64a6 -e -nn
16:40:23.196284 d2:5c:fe:ca:17:4b > ee:ee:ee:ee:ee:ee ethertype IPv4 (0x0800), length 98: 177.177.114.10 > 192.168.100.100: ICMP echo request, id 115, seq 5, length 64
16:40:23.196354 ee:ee:ee:ee:ee:ee > d2:5c:fe:ca:17:4b ethertype IPv4 (0x0800), length 98: 192.168.100.100 > 177.177.114.10: ICMP echo request, id 115, seq 5, length 64
...
```
总结
+  **容器与宿主机之间的通信靠veth pair设备，veth pair可以比喻为一根通的管子两端，从一头进的数据包会从另一头出现**
+  **Calico利用了网卡的代理ARP功能，当ARP请求目标跨网段时(比如上面的169.254.1.1)，网关设备收到此ARP请求，会用自己的MAC地址返回给请求者，这便是代理 ARP（Proxy ARP）**
问题:
使用kubernete的场景不会是单机情况，往往都是集群模式，那肯定涉及跨主机的容器间访问问题，这部分如何处理的呢？