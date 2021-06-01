CentOS7 下软件安装和卸载

## mariadb

1. ```bash
   rpm -qa | grep mariadb
   ```

2. ```bash
   rpm -e --nodeps 上面的包
   ```

3.  **查询是否安装了mysql**

   ```
   rpm -qa|grep mysql
   ```

4. **卸载mysql （下面是卸载mysql的库，防止产生冲突，mysql也是类似卸载方式）**

   ```
   rpm -e --nodeps mysql-libs-5.1.*
   卸载之后，记得：
   find / -name mysql
   删除查询出来的所有东西
   ```

5. centos7安装mysql

   ```
   wget http://dev.mysql.com/get/mysql-community-release-el7-5.noarch.rpm
     #rpm -ivh mysql-community-release-el7-5.noarch.rpm
    #yum install mysql-community-server
   成功安装之后重启mysql服务
   #service mysqld restart
   初次安装mysql是root账户是没有密码的
   设置密码的方法
   #mysql -uroot
   mysql> set password for ‘root’@‘localhost’ = password('mypasswd');
   mysql> exit
   ```

## Mysql主从同步配置

1. 主配置log-bin,指定文件的名字

2. 主配置 server-id默认为1；从配置 server-id与主不能重复,**需要重启服务器！**

   ```properties
   log-bin=imooc_mysql
   server-id=1
   ```

   注意：一定要放到属于[mysql-id]的节点里面

3. 主节点创建备份账户并授权 REPLICATION SLAVE

   `create user 'repl'@'%' identified by 'Imooc@123456';`

   `grant replication slave on *.* to 'repl'@'%';`

   `flush privileges;`

4. 主节点进行锁表 FLUSH TABLES WITH READ LOCK;

   锁表是为了进行备份的，把主节点的数据备份到从数据库当中

   备份的时候需要锁表，这样所有的写操作就不会落到主数据库中了；

   锁表后把主库的数据备份到从库，先查一下主库bin-log的位置并记录下来，然后在从库中进行主从的配置，因为主从的配置要指定bin-log的位置，然后才能把主库的请求放开，这个时候会读取bin-log的位置，这样锁表之后的产生的数据就可以同步到从库了

![1593433428560](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\1593433428560.png)

​        这时候插入数据可以看到被阻塞了

5. 主找到log-bin的位置 SHOW MASTER STATUS;

   ![1593435152302](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\1593435152302.png)

   需要记住Position这个值，后面从库配置的时候要配置bin-log的位置，也就是从这个位置往后读取bin-log日志，进行锁表后产生的数据的同步，这个位置之前的数据，使用下面的命令同步过去。

6. 主备份数据
   mysqldump --all-databases --master-data > dbdump.db

   这行语句需要在myql外面执行，千万不能退出mysql

   ![1593435716505](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\1593435716505.png)

7. 在从数据库节点上执行  scp root@192.168.248.172:~/dbdump.db .  注意后面有个 . 代表当前目录

   把刚才主数据库节点上的dbdump.db下载到从数据库节点

8. 把dbdump.db文件加载到从数据库中

    mysql < dbdump.db -u root -p

9. 主数据库表进行解锁   unlock tables;

   主数据库锁住的插入语句，执行成功，下面需要同步到从数据库

10. 在从上设置主从同步

    ```bash
    mysql> change master to
           master_host='192.168.248.172',
           master_user='repl',
           master_password='Imooc@123456',
           master_log_file='imooc_mysql.000001',
           master_log_pos=892;
    Query OK, 0 rows affected
    ```

    master_host：主库ip地址

    master_user：上面新创建的user

    master_password： 上面新创建的user对应的密码

    master_log_file：SHOW MASTER STATUS;查询出的File名

    master_log_pos：SHOW MASTER STATUS;查询出的Position

11. 从数据库执行    START SLAVE;

## Haproxy安装

### yum安装

1. yum list|grep haproxy

   ![1593492033323](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\1593492033323.png)

2. yum install -y haproxy.x86_64

Haproxy配置

## Haproxy + mycat + mysql

修改haproxy的配置文件：/etc/haproxy/haproxy.cfg

```shell
global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    #
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    #
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    log         127.0.0.1 local2

    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    tcp
    log                     global
    option                  tcplog
    option                  dontlognull
    option http-server-close
    # option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

#---------------------------------------------------------------------
# main frontend which proxys to the backends
#---------------------------------------------------------------------
frontend  main *:5000
    #acl url_static       path_beg       -i /static /images /javascript /stylesheets
    #acl url_static       path_end       -i .jpg .gif .png .css .js

    #use_backend static          if url_static
    default_backend             app

#---------------------------------------------------------------------
# static backend for serving up images, stylesheets and such
#---------------------------------------------------------------------
backend static
    balance     roundrobin
    server      static 127.0.0.1:4331 check

#---------------------------------------------------------------------
# round robin balancing between the various backends
#---------------------------------------------------------------------
backend app
    balance     roundrobin
    server  app1 192.168.248.171:3306 check
    server  app2 192.168.248.173:3306 check
    #server  app3 127.0.0.1:5003 check
    #server  app4 127.0.0.1:5004 check

```

default ： 由原来的http改为tcp

将有关http相关的配置注释掉

```
# option forwardfor       except 127.0.0.0/8
```

关于frontend的配置：我们使用default_backend，把use_backend static这一行注释掉，否则启动会报错：

```
frontend  main *:5000
    #acl url_static       path_beg       -i /static /images /javascript /stylesheets
    #acl url_static       path_end       -i .jpg .gif .png .css .js

    #use_backend static          if url_static
    default_backend             app

```

核心配置: 我们配置了两台mycat分别在171和173服务器上，mycat连接mysql的默认端口为8066

```
backend app
    balance     roundrobin
    server  app1 192.168.248.171:8066 check
    server  app2 192.168.248.173:8066 check
    #server  app3 127.0.0.1:5003 check
    #server  app4 127.0.0.1:5004 check
```

启动haproxy：

`haproxy -f <haproxy 配置文件路径>` 默认配置文件路径为：/etc/haproxy/haproxy.cfg

## keepalived + haproxy + mycat

使用keepalived 解决haproxy单点的问题，做到一个haproxy节点挂了。可以切换到另一台

![1593505180226](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\1593505180226.png)

1. 安装keepalived

   yum install -y keepalived.x86_64

2. 修改配置文件   vim /etc/keepalived/keepalived.conf

   1. 主

      ```shell
      global_defs {
         notification_email {
           acassen@firewall.loc
           failover@firewall.loc
           sysadmin@firewall.loc
         }
         notification_email_from Alexandre.Cassen@firewall.loc
         smtp_server 192.168.200.1
         smtp_connect_timeout 30
         router_id LVS_DEVEL
         vrrp_skip_check_adv_addr
         #vrrp_strict  一定要注释掉，不然无法使用虚拟ip
         vrrp_garp_interval 0
         vrrp_gna_interval 0
      }
      
      # 用于检测haproxy进程是否宕掉
      vrrp_script chk_mycat {
               script "killall -0 haproxy"
               interval 2
      }
      vrrp_instance VI_1 {
      	# 配置为master节点，只有配置为Master才能参加竞选，成为MASTER
          state MASTER
          # 通过ip addr 查看网卡
          interface ens33
          # 路由id，MASTER和BACKUP要相同
          virtual_router_id 51
          priority 100
          advert_int 1
          authentication {
              auth_type PASS
              auth_pass 1111
          }
          # 虚拟ip ，连接时就连接虚拟ip
          virtual_ipaddress {
              192.168.248.170
          }
          track_script {
              chk_mycat
          }
      }
      
      # 真实主机配置
      # 第一行填的时虚拟ip 并指定一个端口，只要不冲突即可,navicate连接就是192.168.248.170:6000
      virtual_server 192.168.248.170 6000 {
          delay_loop 6
          lb_algo rr
          lb_kind NAT
          persistence_timeout 50
          protocol TCP
      	# 配置真实地址，这里就是haproxy的地址，默认端口为5000
          real_server 192.168.248.171 5000 {
              weight 1
          }
      }
      
      
      ```

   2. 从节点

      和上面类似，需要修改MSATER => BACKUP

      修改真实主机地址

3. 安装killall -0命令所需插件： yum install -y psmisc.x86_64

