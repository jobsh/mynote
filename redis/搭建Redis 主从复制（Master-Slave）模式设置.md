# Pre-Working

1. 采用一主二从的模式
2. 创建三台虚拟机
3. 虚拟机ip映射
   1. Master: 192.168.248.111
   2. Slave1: 192.168.248.113
   3. Slave2: 192.168.248.114

# redis 配置

> 在没有配置前，三台节点均为master节点，可以进入redis-cli，通过info replication命令查看

* slave1 配置

  ​	可以通过 /REPLICATION 找到主从复制相关的配置模块，然后分别进行如下操作

  1. 进入redis.conf 

  2. 在redis.conf 中配置附属的Master节点，replicaof 192.168.248.111 6379

  3. 配置Master节点的密码：masterauth <password>

  4. 配置slave节点只读：replica-serve-stale-data yes 默认为yes ，通过此配置实现读写主从节点读写分离

  5. 重启redis-slave1

  6. 如图所示。即为配置成功：由一开始的master变为了slave，并且Master_host为192.168.248.111

     ![1583116987371](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\1583116987371.png)

* slave2 配置

  ​	

​	