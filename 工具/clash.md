### 安装

下载地址：[https://github.com/Dreamacro/clash/releases](https://github.com/Dreamacro/clash/releases)

```shell
# 解压文件
gzip -d clash-linux-amd64-vX.Y.Z.gz

# 移动
mv clash-linux-amd64-vX /usr/local/bin/clash

# 赋可执行权限
chmod +x /usr/local/bin/clash

# 配置service
vim /etc/systemd/system/clash.service
```

将以下内容复制到该文件

```shell
[Unit]
Description=Clash Service
After=network.target

[Service]
Type=simple
Restart=always
ExecStart=/usr/local/bin/clash -d /etc/clash

[Install]
WantedBy=multi-user.target
```



重新加载配置，设置开机自启

```shell
sudo systemctl daemon-reload

sudo systemctl enable clash

sudo systemctl start clash
```



### yacd（推荐）

非官方web-ui，简单，只需要下载压缩包即可

下载地址：[https://github.com/haishanh/yacd/releases](https://github.com/haishanh/yacd/releases)

```shell
# 先创建web-ui目录，解压到web-ui
# -C选项指定了输出目录。--strip-components=1参数指示tar命令在解压缩时去掉路径中的第一个组件，从而实现将输出目录重命名的效果。
mkdir web-ui & tar xf yacd.tar.xz -C web-ui --strip-components=1
```



### clash-dashboard（不推荐）

官方web-ui，比较麻烦

github地址：[https://github.com/Dreamacro/clash-dashboard](https://github.com/Dreamacro/clash-dashboard)

需要切换到`gh-pages`分支，master分支需要自己编译

```
git clone -b gh-pages https://github.com/Dreamacro/clash-dashboard.git
```



### 配置

根据上面配置的service，指定了clash工作配置目录为/etc/clash，启动clash后会自动生成一个config.yaml文件

[官方配置文档](https://github.com/Dreamacro/clash/wiki/Configuration)

参考配置

```yaml
# http和socks共用端口
mixed-port: 7890
external-controller: 0.0.0.0:7899
external-ui: <web-ui路径>
allow-lan: true
mode: Rule
log-level: info
```





