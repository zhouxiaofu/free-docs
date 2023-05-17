# 安装

如果使用的云服务器，为了节省资源，为了节省资源，可以将jenkins安装在我们本地虚拟机或者公司机房

```shell
# 找一个比较大的盘挂载jenkins_home，因为jenkins一般需要10G以上的磁盘空间
df -h

# https://hub.docker.com/r/jenkins/jenkins，找到最新的版本，docker pull latest并不一定会拉最新的

# 拉取镜像
docker pull jenkins/jenkins:<version>

# 后面两个docker的挂载是为了在 jenkins容器中使用宿主机的docker
# -u root拥有root权限，如果没有提供root权限，则会有很多问题需要人为解决
# 在jenkins集群中，50000端口用于子节点(Agent)与主节点(master)之间的通信
# Xmx根据实际进行设置
# jenkins_context，用于存储打包文件，宿主机和jenkins都可以进行访问，比如dockerfile等文件
docker run -d --restart=always --name jenkins \
-p 8899:8080 -p 50000:50000 \
-u root \
--env JAVA_OPTS=-Xmx6g \
-v /home/jenkins_context:/var/jenkins_context \
-v /home/jenkins_home:/var/jenkins_home \
-v /home/maven/repository:/home/maven/repository \
-v $(which docker):/usr/bin/docker \
-v /var/run/docker.sock:/var/run/docker.sock \
jenkins/jenkins:<version>
```



# 开始使用

## 输入初始密码，点击继续

获取初始密码：宿主机执行 `cat <挂载的jenkins_home路径>/secrets/initialAdminPassword`

或者 `docker logs -f jenkins` 在日志中有显示

## 安装插件

### Folders（必需）

该插件允许用户创建“文件夹”来组织作业。用户可以定义自定义分类法（例如，按项目类型，组织类型）。文件夹是可嵌套的，您可以在文件夹中定义视图。

### OWASP Markup Formatter(必需)

Jenkins**允许具有适当权限的用户输入各种对象的描述，例如视图、作业、构建等**。这些描述由标记格式化程序过滤。它们有两个目的： 允许用户对这些描述使用丰富的格式。

### Build Timeout(必需)

这个插件允许构建在指定的时间过去后自动终止。

### Timestamper(必需)
将时间戳添加到控制台输出

### Workspace Cleanup(必需)

该插件提供了一个构建包装器（**在构建开始前删除工作区**）和一个构建后步骤（**在构建完成时删除工作区**）。这些步骤允许您配置将在什么情况下删除哪些文件。构建后步骤也可以考虑构建状态。

### Ant

向Jenkins添加Apache Ant支持

### Gradle

该插件允许Jenkins直接调用Gradle构建脚本。

### Pipeline(必需)

Pipeline 等价于流水线任务

Jenkins Pipeline（或简称为“Pipeline”）是**一套插件，支持将持续交付管道实现和集成到 Jenkins**中。持续交付管道是您将软件从版本控制到用户和客户的过程的自动化表达。

### Pipeline: Stage View(必需)
可以查看任务每个阶段的执行视图

### Git(必需)

该插件将Git与Jenkins集成在一起。

### SSH Build Agents

允许使用SSH协议的Java实现在SSH上启动代理。

### Matrix Authorization Strategy(必需)
提供基于矩阵的安全授权策略

### PAM Authentication(必需)
向Jenkins添加Unix插座身份验证模块（PAM）支持

允许启用、禁用和配置 Jenkins 的关键功能。PAM 身份验证：添加可插入身份验证模块 (PAM) 支持。“这是**一种将多个低级身份验证方案集成到高级应用程序编程接口（API）中的机制**”

### LDAP

将LDAP身份验证添加到Jenkins

### Email Extension

此插件允许您配置电子邮件通知的各个方面。您可以自定义电子邮件的发送时间、接收人以及电子邮件的内容。

### Mailer

此插件允许您为构建结果配置电子邮件通知。

### Localization: Chinese (Simplified) (必需)

Jenkins 核心和插件的简体中文本地化。

### Locale

这个插件控制 Jenkins 的语言，如果只需要一种语言（如简体中文），则无需安装

通常，如果首选语言的翻译可用，Jenkins 会尊重浏览器的语言首选项，并在构建期间使用系统默认语言环境来获取消息。该插件允许您：

- 将系统默认语言环境覆盖为您选择的语言
- 完全忽略浏览器的语言偏好

此功能有时对多语言环境很方便。

### Credentials Binding(推荐)

允许将凭据绑定到环境变量，以便在其他构建步骤中使用。

### SSH(必需)

SSH远程执行脚本

### NodeJS Plugin(必需)

Nodejs插件将NodeJS脚本作为构建步骤

### Role-based Authorization Strategy

如果团队需要**配置权限**，则需要安装本插件

使用基于角色的策略启用用户授权。可以在全局定义角色，也可以针对正则表达式选择的特定作业或节点定义

### Config File Provider(必需)

能够提供配置文件(例如maven的settings.xml, XML, groovy，自定义文件，…)通过UI加载，这些文件将被复制到作业工作区。 



### Maven Integration(必需)

> 需要在安装好之后才能搜索进行安装，推荐插件可能没有这个

maven集成



## 创建第一个管理员用户

填好表单创建管理员账号，完成之后进入Jenkins

用户名为登录时的username，全名为该用户的名字

推荐使用**admin**作为管理员名





# 全局配置

以下的所有配置请根据自身情况选择



## 全局安全配置

授权策略：Role-Based Strategy (需要安装插件)

## Manage and Assign Roles

### Manage Roles

- Global roles

  创建角色，例如给开发、测试人员创建基础角色，只需要勾上 Read 权限

- Item roles

  创建比较细的角色，Pattern为正则表达式例如：`dev-free-.*` 表示拥有`dev-free-`开头的项目

  一般只需要看这几个：Build、Cancel、Read、Workspace



### Assign Roles

用户的权限 = Global roles + Item roles

- Global roles

  输入**username**点击add，然后给该用户配置角色

- Item roles

  也是一样的操作

  

## 全局工具配置



### Global Maven settings.xml

Manage Jenkins --> Managed files -> Add a new Config

参考如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings
    xmlns="http://maven.apache.org/SETTINGS/1.0.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">
    <localRepository>/home/maven/repository</localRepository>
    <mirrors>
        <mirror>
            <id>alimaven</id>
            <name>aliyun maven</name>
            <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
            <mirrorOf>central</mirrorOf>
        </mirror>
    </mirrors>
</settings>
```



### Maven settings.xml

Manage Jenkins --> Managed files -> Add a new Config

根据自身情况进行配置，这个主要用于maven打包时选择setting.xml



### JDK

docker安装的jenkins自带JDK，可以通过`docker exec -it <jenkins容器名或ID> java -version`查看jdk版本

如果项目打包的所需要的java版本**大于**jenkins容器的java版本，则需要手动安装jdk

**安装**

- 点击新增jdk

- 取消自动安装

- 输入jdk别名，如`jdk17`

- 输入jdk的路径，如`/var/jenkins_home/java/jdk-17`

  这里的`/var/jenkins_home`挂载的宿主机目录是`/home/jenkins_home`，所以应该将jdk解压到`/home/jenkins_home/java/jdk-17`





### Git

docker安装的jenkins自带git，无需安装



### maven

- 点击新增Maven
- 选择自动安装
- 选择版本（请与本地开发环境版本保持一致）

### node js

- 点击新增NodeJS
- 选择自动安装
- 选择版本（请与本地开发环境版本保持一致）



## 系统配置

### 全局属性-环境变量

如果有多个项目打包

### SSH remote hosts

需要安装插件 ssh





# 任务配置

视图：添加视图，如dev-free

### 后端

任务类型：Maven



## dyyun-ops-web（前端）

任务类型：**自由风格的软件项目**
