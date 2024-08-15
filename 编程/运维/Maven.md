## Maven & Nexus3

Maven 是开源的 Java 项目管理工具，所有项目的配置信息都在 pom.xml 文件中定义，Maven 根据 pom.xml 来管理项目的生命周期。

Nexus3 是 Maven 的 Jar 包私有仓库。

安装 Maven

~~~shell
sudo apt install maven
mvn -version
~~~

安装 Nexus3

~~~shell
mkdir -p /usr/local/work/nexus-data && chown -R 200 /usr/local/work/nexus-data

docker run -d \
-p 8081:8081 \
--name nexus \
-v /usr/local/work/nexus-data:/nexus-data \
sonatype/nexus3:3.19.1

# 获取登录密码
# 默认用户名是 admin
echo `docker exec nexus cat /nexus-data/admin.password`
~~~

访问 localhost:8081 即可进入 Nexus3 的 Web 管理页面，注意首次加载时要等待个 1 min。

Nexus3 的仓库类型有：

![img](./../../../../AppData/Roaming/Typora/draftsRecover/assets/2174081-20210902162958227-1673162218.png)

- proxy：远程仓库，如果在本地仓库找不到 Jar 包的话，就从远程仓库中加载
- Hosted：本地仓库
- Group：仓库组（Nexus3 特有的概念），将多个仓库聚合在一起，这样用户在 pom.xml 文件中就不需要配置多个仓库地址了，仅需配置仓库组的地址即可

创建 Hosted 仓库时，要把 Deployment Policy 设置为 allow redeploy，否则只有 depolyment 用户才能上传 Jar 包，其他用户上传会返回 403 错误。

Maven Jar 包的不同版本：

1. Snapshot 版本表示不稳定，尚在开发中的版本
2. Release 版本表示稳定的