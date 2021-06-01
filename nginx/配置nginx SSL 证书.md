配置nginx SSL 证书

进入 nginx.conf目录

在nginx下新建cert文件夹

![1584352533839](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\1584352533839.png)

新建文件夹，设置文件夹名为网站域名（方便管理）

![1584352586317](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\1584352586317.png)

把从阿里云下载好的ssl证书解压（*.key    *.pem），放在对应的域名文件夹下

![1584352644791](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\1584352644791.png)

配置nginx.conf

![1584352725168](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\1584352725168.png)