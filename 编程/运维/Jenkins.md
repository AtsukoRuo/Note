# Jenkins

Jenkins 是一个 CI/CD工具，它的作用主要包括自动化构建、测试和部署软件项目，并且支持分布式构建与测试。

1. 在 Docker 中部署 Jenkins

   ~~~shell
   mkdir -p /usr/docker/jenkins_data && chown -R  1000:1000 /usr/docker/jenkins_data
   
   docker run -d -p 8082:8080 -p 50000:50000 \
   -v  /usr/docker/jenkins_data:/var/jenkins_home \
   -v /etc/localtime:/etc/localtime \
   -v /usr/bin/docker:/usr/bin/docker  \ 
   -v /var/run/docker.sock:/var/run/docker.sock \
   # 通过 mvn -v 来查看 maven 的 bin路径
   -v /usr/lib/jvm/java-11-openjdk-amd64:/usr/local/java \
   -v /usr/share/maven:/usr/local/maven \
   --dns=8.8.8.8 \
   --name myjenkins jenkins/jenkins:latest
   ~~~

   注意，容器中 jenkins user 的 uid 为1000，我们要保证本地数据卷的权限与之匹配：

   ~~~shell
   sudo chown -R 1000:1000 /usr/docker/jenkins_data
   ~~~

2. 访问 `http://localhost:8080`，并安装推荐插件（一定记得修改容器的 DNS）。

   如果插件下载失败，就先跳过该步骤，然后进入「插件管理」来手动安装插件

   按照如下命令来获取密码：

   ~~~shell
   docker exec -it myjenkins bash
   cat /var/jenkins_home/secrets/initialAdminPassword
   ~~~

3. 在 Manage Jenkins -> Tools 下配置 JDK 、 Maven 环境变量、Git，之前我们将本地的 JDK 挂载到 `/usr/local/java` 了，这里我们仅需填入这个路径即可。 Maven 同理。对于 Git，我们执行 `which git` 来获取 Git 的可执行路径



安装两个插件 `Role-based Authorization Strategy`、`Authorize Project`，然后在 Manage Jenkins -> Security -> 授权策略中使用 Role-Based Strategy。然后在 Manage Jenkins -> Manage and Assign Roles 中配置角色和权限。

