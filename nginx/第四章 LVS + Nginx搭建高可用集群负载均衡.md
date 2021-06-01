# 第四章 LVS + Nginx搭建高可用集群负载均衡

## LVS简介

The Linux Virtual Server is a highly scalable and highly available server built on a cluster of real servers, with the [load balancer](http://kb.linuxvirtualserver.org/wiki/Load_balancer) running on the Linux operating system. The architecture of the server cluster is fully transparent to end users, and the users interact as if it were a single high-performance virtual server. For more information, click [here](http://www.linux-vs.org/whatis.html).

​														引自—— <http://www.linux-vs.org/index.html>

## 为什么要使用LVS+Nginx？

* LVS效率更高
  * LVS 基于四层（传输层），即ip+端口号转发，而nginx是基于七层的负载均衡，nginx收到请求报文后还需要对报文解析
  * Nginx不仅接受请求，还要响应，LVS可以只接受请求不响应，只是转发
* 单个LVS的承压能力要远高于Nginx，所有可以使用LVS充当Nginx的调度者,实现Nginx的集群

### Nginx网络拓扑图

![1582768387681](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\1582768387681.png)

### LVS网络拓扑

![1582768589631](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\1582768589631.png)

LVS的核心就是ipvs（可以把ipvs理解为苹果手机里的IOS），ipvs可以虚拟出一个vip，用户访问的请求，首先会到达vip到达负载均衡的调度器（LVS），然后由LVS根据自己的算法，挑选出一个Real Server（通常就是Nginx服务器）来响应用户的请求。



## LVS的三种模式

* LVS-NAT模式

  ![1582768946271](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\1582768946271.png)

* LVS-TUN（一种ip隧道模式）

  ![1582769277511](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\1582769277511.png)

  > TUN模式有一个硬性要求：所有的计算机节点都必须有一个网卡，这个网卡就是用于建立隧道的。计算机节点彼此的通信都会通过隧道。
  >
  > 用户所有的响应不会经过LVS，可以做到上行很小，下行很大（可以接收到所有Real Server的响应）。
  >
  > 从图可知RealServer必须要暴露在公网，这是这种模式的缺点。

* **LVS-DR模式**

  ![1582769714093](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\1582769714093.png)

  DR：Direct Reloader 直接路由模式

  不需要将RS暴露在公网，把所有的RS都“包”了起来，再通过一个Router（绑定虚拟ip）返回给用户

### 搭建LVS-DR模式,配置LVS节点与ipvsadm

#### 前期准备

1. 服务器与ip规划：

   - LVS - 1台
     - VIP（虚拟IP）：192.168.1.150
     - DIP（转发者IP/内网IP）：192.168.1.151
   - Nginx - 2台（RealServer）
     - RIP（真实IP/内网IP）：192.168.1.171
     - RIP（真实IP/内网IP）：192.168.1.172

2. 所有计算机节点关闭网络配置管理器，因为有可能会和网络接口冲突：

   ```
   systemctl stop NetworkManager 
   systemctl disable NetworkManager
   ```

------

#### 创建子接口

1. 进入到网卡配置目录，找到咱们的ens33：
   ![图片描述](http://img1.sycdn.imooc.com//climg/5dc2875b08ce651416000419.jpg)
2. 拷贝并且创建子接口：

```
cp ifcfg-ens33 ifcfg-ens33:1
* 注：`数字1`为别名，可以任取其他数字都行
```

1. 修改子接口配置： `vim ifcfg-ens33:1`
2. 配置参考如下：
   ![图片描述](http://img1.sycdn.imooc.com//climg/5dc287db0882451906340344.jpg)
   - 注：配置中的 192.168.1.150 就是咱们的vip，是提供给外网用户访问的ip地址，道理和nginx+keepalived那时讲的vip是一样的。
3. 重启网络服务，或者重启linux：
   ![图片描述](http://img1.sycdn.imooc.com//climg/5dc287fc08c55cbc15100108.jpg)
4. 重启成功后，ip addr 查看一下，你会发现多了一个ip，也就是虚拟ip（vip）
   ![图片描述](http://img1.sycdn.imooc.com//climg/5dc2887b086239f113560442.jpg)

------

#### 创建子接口 - 方式2（不推荐）

1. 创建网络接口并且绑定虚拟ip：

   ```
   ifconfig ens33:1 192.168.1.150/24
   ```

   - 配置规则如下：
     ![图片描述](http://img1.sycdn.imooc.com//climg/5dc2888d08852c0412950520.jpg)
     配置成功后，查看ip会发现新增一个192.168.1.150：
     ![图片描述](http://img1.sycdn.imooc.com//climg/5dc288a0088cf91d11070450.jpg)

> - 通过此方式创建的虚拟ip在重启后会自动消失

------

#### 安装ipvsadm

现如今的centos都是集成了LVS，所以`ipvs`是自带的，相当于苹果手机自带ios，我们只需要安装`ipvsadm`即可（ipvsadm是管理集群的工具，通过ipvs可以管理集群，查看集群等操作），命令如下：

```
yum install ipvsadm
```

#### 安装成功后，可以检测一下： ![图片描述](http://img1.sycdn.imooc.com//climg/5dc288c1086c8e5015100198.jpg) 图中显示目前版本为1.2.1，此外是一个空列表，啥都没。

- 注：关于虚拟ip在云上的事儿

1. 阿里云不支持虚拟IP，需要购买他的负载均衡服务
2. 腾讯云支持虚拟IP，但是需要额外购买，一台节点最大支持10个虚拟ip

### 搭建LVS-DR模式,为两台RS配置虚拟ip

1  cd /etc/sysconfig/network-scripts/

2  vim ifcfg-lo:1 (复制一份ifcfgf-lo)

3  将如下内容复制、粘贴到ifcfg-lo:1 

```shell
DEVICE=lo:1
# 虚拟ip
IPADDR=192.168.248.150
NETMASK=255.255.255.255
NETWORK=127.0.0.0
# If you're having problems with gated making 127.0.0.0/8 a martian,
# you can change this to something else (255.255.255.255, for example)
BROADCAST=127.255.255.255
ONBOOT=yes
NAME=loopback

4 ifup lo命令更新配置ifcfg-lo:1的配置

5 ip addr 查看lo网卡的ip
```

![1582778316233](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\1582778316233.png)

### 搭建LVS-DR模式, 为两台RS配置arp

#### ARP响应级别与通告行为的概念

1. arp-ignore：ARP响应级别（处理请求）
   - 0：只要本机配置了ip，就能响应请求
   - 1：请求的目标地址到达对应的网络接口，才会响应请求
2. arp-announce：ARP通告行为（返回响应）
   - 0：本机上任何网络接口都向外通告，所有的网卡都能接受到通告
   - 1：尽可能避免本网卡与不匹配的目标进行通告
   - 2：只在本网卡通告

------

#### 配置ARP

1. 打开sysctl.conf:

   ```
   vim /etc/sysctl.conf
   ```

2. 配置`所有网卡`、`默认网卡`以及`虚拟网卡`的arp响应级别和通告行为，分别对应：`all`，`default`，`lo`：

   ```
   net.ipv4.conf.all.arp_ignore = 1
   net.ipv4.conf.default.arp_ignore = 1
   net.ipv4.conf.lo.arp_ignore = 1
   
   net.ipv4.conf.all.arp_announce = 2
   net.ipv4.conf.default.arp_announce = 2
   net.ipv4.conf.lo.arp_announce = 2
   
   
   ```

3. 刷新配置文件：sysctl -p

4. 增加一个网关，用于接收数据报文，当有请求到本机后，会交给lo去处理：route add -host 192.168.248.150 dev lo:1

5. 查看配置是否生效：route -n

   ![1582779821036](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\1582779821036.png)

6. 防止重启失效，做如下处理，用于开机自启动：

```
	echo "route add -host 192.168.1.150 dev lo:1" >> /etc/rc.local
```

### 搭建LVS-DR模式, 使用ipvsadm配置集群规则

1. 创建LVS节点，用户访问的集群调度者

   ```
   ipvsadm -A -t 192.168.248.150:80 -s rr -p 5
   ```

   - -A：添加集群
   - -t：tcp协议
   - ip地址：设定集群的访问ip，也就是LVS的虚拟ip
   - -s：设置负载均衡的算法，rr表示轮询
   - -p：设置连接持久化的时间

2. 创建2台RS真实服务器

   ```
   ipvsadm -a -t 192.168.248.150:80 -r 192.168.248.105:80 -g
   ipvsadm -a -t 192.168.248.150:80 -r 192.168.248.106:80 -g
   ```

   - -a：添加真实服务器
   - -t：tcp协议
   - -r：真实服务器的ip地址
   - -g：设定DR模式

3. 保存到规则库，否则重启失效

   ```
   ipvsadm -S
   ```

4. 检查集群

   - 查看集群列表

     ```
     ipvsadm -Ln
     ```

   - 查看集群状态

     ```
     ipvsadm -Ln --stats
     ```

5. 其他命令：

```nginx
    # 重启ipvsadm，重启后需要重新配置
    service ipvsadm restart
    # 查看持久化连接
    ipvsadm -Ln --persistent-conn
    # 查看连接请求过期时间以及请求源ip和目标ip
    ipvsadm -Lnc
    
    # 设置tcp tcpfin udp 的过期时间（一般保持默认）
    ipvsadm --set 1 1 1
    # 查看过期时间
    ipvsadm -Ln --timeout
```

1. 更详细的帮助文档：

   ```
   ipvsadm -h
   man ipvsadm
   ```

### 搭建LVS-DR模式,验证DR模式，探讨LVS的持久化机制

### 搭建Keepalive+LVS+Nginx高可用集群负载均衡 - 配置MASTER

1、编辑keepalived.conf

```
global_defs {
   router_id keep_111
}


# 实现keepalived+LVS
vrrp_instance VI_1 {
    # MASTER:主节点；BACKUP:备用节点
    state MASTER
    # 当前实例绑定的网卡，每台服务器网卡不一定一样，需要ip addr查看
    interface ens33
    # 保证主备节点一致,修改为52，实现第二组路由
    virtual_router_id 41
    # 优先级/权重，谁的优先级越高，MASTER挂掉后，谁就竞选为MASTER
    priority 100
    # 主备之间同步检查的时间间隔
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.248.150
    }
}

virtual_server 192.168.248.150 80 { #集群所使用的VIP和端口
    delay_loop 6                    #健康检查间隔，单位为秒
    lb_algo rr                      #lvs调度算法rr|wrr|lc|wlc|lblc|sh|dh
    lb_kind DR                      #负载均衡转发规则。一般包括DR,NAT,TUN 3种
    persistence_timeout 5           #会话保持时间
    protocol TCP                    #转发协议，有TCP和UDP两种，一般用TCP，没用过UDP

    #负载均衡的真实服务器，也就是nginx节点的具体的真实地址，包括IP和端口号
    real_server 192.168.248.105 80 {
        weight 1            #默认为1,0为失效

        TCP_CHECK {                     #通过tcpcheck判断RealServer的健康状态
            connect_timeout 2           #连接超时时间
            nb_get_retry 2              #重连次数
            delay_before_retry 3        #在retry之前的重连间隔时间
            connect_port 80             #健康检查的端口
        }

    }

    #负载均衡的真实服务器，也就是nginx节点的具体的真实地址，包括IP和端口号
    real_server 192.168.248.106 80 {
        weight 1            #默认为1,0为失效

        TCP_CHECK {                     #通过tcpcheck判断RealServer的健康状态
            connect_timeout 2           #连接超时时间
            nb_get_retry 2              #重连次数
            delay_before_retry 3        #在retry之前的重连间隔时间
            connect_port 80             #健康检查的端口
        }

    }

}

```

2、重启 keepalived服务：service keepalived restart

3、ipvsadm -Ln 检查

![1582800195654](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\1582800195654.png)

### 搭建Keepalive+LVS+Nginx高可用集群负载均衡 - 配置BACKUP

1、编辑keepalived.conf

```
global_defs {
   router_id keep_111
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 41
    priority 50
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.248.150
    }
}

virtual_server 192.168.248.150 80 { #集群所使用的VIP和端口
    delay_loop 6                    #健康检查间隔，单位为秒
    lb_algo rr                      #lvs调度算法rr|wrr|lc|wlc|lblc|sh|dh
    lb_kind DR                      #负载均衡转发规则。一般包括DR,NAT,TUN 3种
    persistence_timeout 5           #会话保持时间
    protocol TCP                    #转发协议，有TCP和UDP两种，一般用TCP，没用过UDP

    #负载均衡的真实服务器，也就是nginx节点的具体的真实地址，包括IP和端口号
    real_server 192.168.248.105 80 {
        weight 1            #默认为1,0为失效

        TCP_CHECK {                     #通过tcpcheck判断RealServer的健康状态
            connect_timeout 2           #连接超时时间
            nb_get_retry 2              #重连次数
            delay_before_retry 3        #在retry之前的重连间隔时间
            connect_port 80             #健康检查的端口
        }

    }

    #负载均衡的真实服务器，也就是nginx节点的具体的真实地址，包括IP和端口号
    real_server 192.168.248.106 80 {
        weight 1            #默认为1,0为失效

        TCP_CHECK {                     #通过tcpcheck判断RealServer的健康状态
            connect_timeout 2           #连接超时时间
            nb_get_retry 2              #重连次数
            delay_before_retry 3        #在retry之前的重连间隔时间
            connect_port 80             #健康检查的端口
        }

    }

}

```

2、重启 keepalived服务：service keepalived restart

3、ipvsadm -Ln 检查

## LVS的负载均衡算法

