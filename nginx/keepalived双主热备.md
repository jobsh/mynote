# keepalived双主热备

## 原理

![1582724064007](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\1582724064007.png)

* 好处：这样避免了备用机只有在主机（MASTER）停机的时候才工作的资源浪费的问题。
* 核心：设置两个vip（虚拟ip）让每个虚拟ip都有一个nginx主机可以映射。（实际上是有两组路由）
* DNS: 实现双主热备必不可少的，通过DNS来选择vip，DNS是在云服务器里面的。

## 实现

### keepalived.conf配置

#### 原来MASTER配置

```
vrrp_instance VI_1 {
    # MASTER:主节点；BACKUP:备用节点 
    state MASTER
    # 当前实例绑定的网卡，每台服务器网卡不一定一样，需要ip addr查看
    interface ens33
    # 保证主备节点一致
    virtual_router_id 51
    # 优先级/权重，谁的优先级越高，MASTER挂掉后，谁就竞选为MASTER
    priority 100
    # 主备之间同步检查的时间间隔
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.248.120
    }
    track_script {
        check_nginx_alive   # 追踪 nginx 脚本
    }
}

# 实现双主热备
vrrp_instance VI_2 {
    # MASTER:主节点；BACKUP:备用节点 
    state BACKUP
    # 当前实例绑定的网卡，每台服务器网卡不一定一样，需要ip addr查看
    interface ens33
    # 保证主备节点一致,修改为52，实现第二组路由
    virtual_router_id 52
    # 优先级/权重，谁的优先级越高，MASTER挂掉后，谁就竞选为MASTER
    priority 80
    # 主备之间同步检查的时间间隔
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
    192.168.248.121
    }
}

```

#### 原来备用机配置

```
vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 51
    priority 80
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.248.120
    }
}

# 实现双主热备
vrrp_instance VI_2 {
    # MASTER:主节点；BACKUP:备用节点
    state MASTER
    # 当前实例绑定的网卡，每台服务器网卡不一定一样，需要ip addr查看
    interface ens33
    # 保证主备节点一致,修改为52，实现第二组路由
    virtual_router_id 52
    # 优先级/权重，谁的优先级越高，MASTER挂掉后，谁就竞选为MASTER
    priority 100
    # 主备之间同步检查的时间间隔
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.248.121
    }
}

```

## 测试

当一台服务器（keepalived）挂掉后，另一台会和另一个vip绑定，从而有了两个vip，都是MASTER

![1582726055000](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\1582726055000.png)