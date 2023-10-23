# Tinyproxy 

官方github地址[https://github.com/tinyproxy/tinyproxy](https://github.com/tinyproxy/tinyproxy)

dockerhub地址（非官方）[https://hub.docker.com/repository/docker/monokal/tinyproxy](https://hub.docker.com/repository/docker/monokal/tinyproxy)

```shell
docker run -d --name=tinyproxy \
-p 8888:8888 \
-e BASIC_AUTH_USER=<username> \
--env BASIC_AUTH_PASSWORD=<password> \
monokal/tinyproxy ANY
```



# Squid

## 配置

官方配置文件示例：[http://wiki.squid-cache.org/ConfigExamples/](http://wiki.squid-cache.org/ConfigExamples/)

### squid.conf

```bash
http_port 3128

auth_param basic program /usr/lib/squid/basic_ncsa_auth /etc/squid/passwd
auth_param basic realm proxy
acl authenticated proxy_auth REQUIRED
http_access allow authenticated

# 最后拒绝所有其它请求
http_access deny all
```



### passwd

生成密码

- 使用crypt算法（不推荐，因为安全性较低）：

  ```shell
  htpasswd -nb -d username password
  ```

- 使用MD5算法：

  ```shell
  htpasswd -nb -m username password
  ```

- 使用BCrypt算法：

  ```shell
  htpasswd -nb -B username password
  ```

- 在线生成：

  - 国内：[https://tool.oschina.net/htpasswd](https://tool.oschina.net/htpasswd)
  - 国外：[https://hostingcanada.org/htpasswd-generator/](https://hostingcanada.org/htpasswd-generator/)

将生成好的密码保存到passwords文件即可，生成的内容如：`username:$apr1$ruca84Hq$mbjdMZBAG.KWn7vfN/SNK0`



## docker（推荐）

```shell
mkdir /home/squid

# 编辑配置文件，参考配置
vim /home/squid/squid.conf
vim /home/squid/passwd

docker run -d --name=squid \
-p 3128:3128 \
-v /home/squid/squid.conf:/etc/squid/squid.conf \
-v /home/squid/passwd:/etc/squid/passwd \
ubuntu/squid
```



## centos

```shell
yum install -y squid
```



## 常用命令

```shell
# 启动
squid -s

# 重新加载配置
squid -k reconfig

# 停止
squid -k shutdown

# 验证配置是否正确
squid -k parse
```



# 使用示例

hutool  HttpUtil发请求

```java
import cn.hutool.http.HttpRequest;
import cn.hutool.http.HttpUtil;
import java.net.Authenticator;
import java.net.InetSocketAddress;
import java.net.PasswordAuthentication;
import java.net.Proxy;

public class TestProxy {
    public static void main(String[] args) {
        HttpRequest httpRequest = HttpUtil.createGet("https://api.openai.com/v1/models");
        // 设置代理账号密码
        final String userName = "代理用户名";
        final String password = "代理密码";
        System.setProperty("jdk.http.auth.tunneling.disabledSchemes", "");
        Authenticator.setDefault(
                new Authenticator() {
                    @Override
                    public PasswordAuthentication getPasswordAuthentication() {
                        if (getRequestorType() == RequestorType.PROXY) {
                            return new PasswordAuthentication(userName, password.toCharArray());
                        }
                        return null;
                    }
                }
        );
        // 创建代理地址和端口
        InetSocketAddress address = new InetSocketAddress("代理ip", 代理port);
        // 创建代理对象
        Proxy proxy = new Proxy(Proxy.Type.HTTP, address);
        // 设置代理
        httpRequest.setProxy(proxy);

        System.out.println(httpRequest.execute().body());
    }
}
```



