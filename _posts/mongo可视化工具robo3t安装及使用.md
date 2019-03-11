#### 安装

下载地址：https://robomongo.org/download

在网页上选择合适的版本进行下载，安装十分简单



#### 连接

连接mongodb

![企业微信截图_15523007563844](https://ws3.sinaimg.cn/large/006tKfTcgy1g0z21prydbj30j806fq38.jpg)

通过点击create可以创建新的连接，接下来可以填写连接的具体信息

![企业微信截图_15523008752484](https://ws2.sinaimg.cn/large/006tKfTcgy1g0z225f546j30ga0bb3ym.jpg)

如上图所示，主要包含connection、Autherntication、SSH、SSL和Advanced几个大的选项卡。

##### connection

connection中用来填写连接的基本信息，如连接方式、连接名称、要连接的mongo server的ip地址及端口号等。连接方式选direct connection就行。

##### authentication

authentication中主要用来填写mongod的授权信息，包括用户名、密码等，database一般填admin，表示可以访问该mongo server上的所有数据库。

![企业微信截图_15523010361109](https://ws2.sinaimg.cn/large/006tKfTcgy1g0z22ebn2bj30f50amdfu.jpg)

##### ssh

ssh主要用来打通隧道的。适用场景是，当mongo server需要先连跳板机，然后在跳板机上连接mongo时，开发过程中十分常见。

![企业微信截图_15523012358193](https://ws1.sinaimg.cn/large/006tKfTcgy1g0z22o6164j30fc0aijrd.jpg)

##### Advanced

Advanced中可以指定你默认要访问的数据库名。

![企业微信截图_15523013257053](https://ws3.sinaimg.cn/large/006tKfTcgy1g0z230fv3yj30ez0ag3yh.jpg)

但我在填写指定的数据库名后，读取指定数据库下的collection时，发现会出现如下错误，但把advanced中的数据库名改为admin，就可以正常读写了。

```json
Error: Failed to refresh 'Users'. Socket exception [CONNECT_ERROR] for couldn't connect to server 127.0.0.1:63685, connection attempt failed
```

##### 连接Mongo server

当上述信息填写完整后，可以点击Test按钮进行测试，没有问题后就可以连接mongo server，对其指定的数据库及collection进行读写了。

![企业微信截图_15523017868078](https://ws2.sinaimg.cn/large/006tKfTcgy1g0z237ok0cj306f0dtmxh.jpg)