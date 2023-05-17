# 简介

[Github地址](https://github.com/fatedier/frp)

frp 是一个专注于内网穿透的高性能的反向代理应用，支持 TCP、UDP、HTTP、HTTPS 等多种协议。可以将内网服务以安全、便捷的方式通过具有公网 IP 节点的中转暴露到公网。



# frps（服务端）

## 配置(frps.ini)

[配置参考](https://gofrp.org/docs/reference/server-configures/)

```ini
[common]
bind_port = 7000
# 允许代理绑定的服务端端口
allow_ports = 20080,20443

# 开启鉴权
authenticate_heartbeats = true
authenticate_new_work_conns = true
token = free@123
```



## 安装

### 压缩包

下载文件: [https://github.com/fatedier/frp/releases](https://github.com/fatedier/frp/releases)

- 找到适合自己系统的压缩包下载
- 解压
- 运行`./frps -c ./frps.ini`



### docker（推荐）

```shell
# 拉取镜像
docker pull snowdreamtech/frps

# 添加配置文件（可参考上面的配置）
vim /home/frp/frps.ini

# 运行
docker run -d --restart=always --name frps --network host -v /home/frp/frps.ini:/etc/frp/frps.ini snowdreamtech/frps
```



# frpc（客户端）

## 配置(frpc.ini)

[配置参考](https://gofrp.org/docs/reference/client-configures/)

```ini
[common]
server_addr = <frps的ip>
server_port = 7000
tls_enable = true
authenticate_heartbeats = true
authenticate_new_work_conns = true
token = free@123

[free-http]
type = tcp
local_ip = 127.0.0.1
local_port = 80
remote_port = 20080

[free-https]
type = tcp
local_ip = 127.0.0.1
local_port = 443
remote_port = 20443

[docker]
type = tcp
local_ip = 127.0.0.1
local_port = 9080
remote_port = 9080
```



## 安装

### 压缩包

下载文件: [https://github.com/fatedier/frp/releases](https://github.com/fatedier/frp/releases)

- 找到适合自己系统的压缩包下载
- 解压
- 运行`./frpc -c ./frpc.ini`



### docker（推荐）

```shell
# 拉取镜像
docker pull snowdreamtech/frpc

# 添加配置文件（可参考上面的配置）
vim /home/frp/frpc.ini

# 运行
docker run -d --restart=always --name frpc --network host -v /home/frp/frpc.ini:/etc/frp/frpc.ini snowdreamtech/frpc
```

