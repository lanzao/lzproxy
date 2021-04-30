# LZProxy 

通常情况下ssh登录服务器都需要知道服务器的地址（外网IP和端口），但也有一些场景需要通过SSH去登录和管理没有公网IP的设备，lzproxy项目实现的功能是使用ssh协议登录无外网IP的私网设备。

本项目代码仅提供整体流程代码，并未在安全性（例如TLS验证）、鲁棒性等方面进行完善，主要是为了保证项目代码足够精简，使主流程清晰易懂。

声明：本项目代码仅供学习参考，不得用于商业用途，由此带来损失本人概不负责。

如果这些代码对你能有所帮助，那这一切就还是非常值得的。
TODO：打赏二维码

# Why
在网上搜了一番，ssh over websocket的实现有不少，但都是正向登录的（需要知道目标机器的ip地址），但反向登录的（登录私网机器）的几乎没有（我没有找到），因此我将实现的Demo开源出来，以供大家交流学习。

huproxy（https://github.com/google/huproxy）项目实现了类似的简单的功能，它和lzproxy有几个比较大的区别：

- lzproxy支持使用一个独立的proxy server来代理和维护多个不同边缘设备的连接，而huproxy实现的是单个连接版本，每个连接对应一个server和一个外网监听地址。
- lzproxy支持从任意可以访问到proxy server地址的地方通过ssh登录到相应的私网设备，而不一定要在proxy server上执行ssh。


# Components
包括3个二进制文件：

- proxyserver: SSH的代理服务器，用来接收、建立和维护和所有其他设备的连接，本质是一个http server，部署在Server端。
- client_connector: 用于连接proxyserver，部署在设备端。
- sshclient: 用于SSH登录相应设备的客户端，可以部署在服务端或者其他设备端（能访问到proxyserver即可）。

# How to build?
```
make build
```
成功执行后将在output目录下产生相应二进制文件：
```
output
├── client_connector
├── proxyserver
└── sshclient
```

# Setup and run
## 需求
假设有：

- 服务器ServerA：有外网IP，其他设备可访问到
- 边缘设备DeviceA：无外网IP，但是可以访问外网的设备，（例如机顶盒、路由器等）。

要实现从ServerA SSH登录 DeviceA、DeviceB。

## 1. 部署二进制

```
# to serverA
scp output/proxyserver root@${serverA_ip}:/home/lzproxy/
scp output/sshclient root@${serverA_ip}:/home/lzproxy/

# to DeviceA
scp output/client_connector root@${DeviceA_ip}:/home/lzproxy/
```

## 2. 拉起服务

```
# 在ServerA上拉起proxyserver
/home/lzproxy/proxyserver -listen=${IP}:${Port}

# 在边缘设备DeviceA拉起client_connector
/home/lzproxy/client_connector -server="${IP}:${Port}" -name="x2"
```

## 3. ssh登录DeviceA
在任意一台能够访问${IP}:${Port}的linux设备上执行命令，例如在ServerA上执行：

```
ssh -o 'ProxyCommand=/home/lzproxy/sshclient -target="x2" -server="1.71.153.179:8086"' x2
```
这是会要求你输入密码，输入完成后即可登录。

也可以通过信任关系实现免密登录：提前将ServerA的/home/${user}/.ssh/id_rsa.pub文件的内容添加到DeviceA的/home/${user}/.ssh/authorized_keys中，这样即可实现免密登录。
