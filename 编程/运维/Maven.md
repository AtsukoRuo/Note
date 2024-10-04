# Maven & Nexus3

[TOC]

## 概念

Maven 是开源的 Java 项目管理工具，所有项目的配置信息都在 pom.xml 文件中定义，Maven 根据 pom.xml 来管理项目的生命周期。**Maven 仅仅定义了抽象的生命周期，而具体的操作则是由 Maven 插件来完成的**。

通过依赖坐标引入我们项目需要的 Jar 包，它包括：

~~~xml
<dependency>
    <groupId>net.lazyegg.maven</groupId>
    <artifactId>Hello</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <scope>compile</scope>            
</dependency>
~~~

- groupid：组织域名的倒序 

- artifactId：模块名称

- version：模块版本

- scope：依赖的范围

  | compile  | test | provided |      |
  | -------- | ---- | -------- | ---- |
  | 主程序   | √    | ×        | √    |
  | 测试程序 | √    | √        | √    |
  | 参与部署 | √    | ×        | ×    |

通过 exclusions 来排除依赖：

~~~xml
<dependency>
    <groupId>net.lazyegg.maven</groupId>
    <artifactId>Hello</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <scope>compile</scope>
    <!--当引入 net.lazyegg.maven 时，就排除 commons-logging 依赖-->
    <exclusions>
        <exclusion>
            <groupId>commons-logging</groupId>
            <artifactId>commons-logging</artifactId>
            </exclusion>
    </exclusions>
</dependency>
~~~



Maven 的主要环节：

- 清理（clean）：删除以前的编译结果
- 编译（compile）：将 Java 源程序编译为字节码文件
- 测试（test）
- 打包（package）：在 pom.xml 里，项目的打包方式有 pom，war，jar 三种。
  - pom 用在父级工程或聚合工程中，做所有子模块的公共配置：
  - Jar 包，它并**没有**把所依赖的 Jar 包一并打入，需要运行时指定 lib 目录。Jar 包的不同版本：
    1. Snapshot 版本表示不稳定，尚在开发中的版本
    2. Release 版本表示稳定的
  - war 包，可以直接在 Web 容器中部署运行。
- 安装（install）：会将打包后的 Jar 包放到 maven 本地仓库当中，然后其他项目就可以通过坐标来引用它。
- 部署（deploy）：将打包的结果部署到远程仓库，或将 war 包部署到服务器上运行。

给 SpringBoot 项目打包时，推荐安装如下插件：

~~~xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
~~~

这样在以 Jar 包方式打包本项目时，一并打包它所依赖的包，以及会集成一个 Tomcat 服务器进去（以 Jar 包形式提供）。这样便可以直接通过 java 命令来运行。



当依赖继承时，父项目要打包成 pom，定义父子项目的示例：

~~~xml
<!--在子文件中，通过 parent 标签来指定配置文件-->
<parent>
    <groupId>cn.atsukoruo</groupId>
    <artifactId>CloudMusic</artifactId>
    <version>0.0.1-SNAPSHOT</version>
</parent>


<dependencies>
    <!--使用其他微服务中的包-->
    <dependency>
        <groupId>cn.atsukoruo</groupId>
        <artifactId>product-common</artifactId>
    </dependency>
</dependencies>
~~~

~~~xml
<!--  packaging的默认打包类型是jar。所有的父工程打包方式都需要设置成pom-->
<packaging>pom</packaging>
<!--子 Maven 项目名-->
<modules>
    <module>project-common</module>
</modules>

<!--依赖配置：目前这里的配置的依赖所引入的 jar 包在此工程下的所有子工程都会被引入-->
<dependencies>
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>fastjson</artifactId>
        <version>1.2.73</version>
    </dependency>
</dependencies>

<!--依赖管理：这里的配置的依赖只是对依赖版本的管理配置，子工程并不会直接引入。如果子工程要需要引入只需要加入如下标签：<dependency> -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-core</artifactId>
            <version>${spring.version}</version>
        </dependency>
    </dependencies>
</dependencyManagement> 
~~~



将多个工程拆分为模块后，需要手动逐个打包。使用了聚合之后（一般在父项目中使用）就可以批量进行 Maven 工程的安装、清理工作。

~~~xml
<!-- 配置聚合 -->
<modules>
    <!-- 通过名字来指定各个子工程 -->
    <module>starfish-learn-grpc</module>
    <module>starfish-learn-kafka</module>
    <module>starfish-web-demo</module>
</modules>
~~~

在 Maven 构建时请注意：

- 有关 Maven 的 Build 配置请在子项目中编写，即使它们的 Build 配置是相同的，也不要在父项目中编写。**父项目仅仅起到提供默认依赖 dependency 的作用**
-  优先 install 父项目以及公共模块，然后再 package 其他微服务项目
-  公共项目请不要引入 Build 配置

## 安装

安装 Maven

~~~shell
sudo apt install maven
mvn -version
~~~

在 ${Maven}/conf/settings 中，更换下载源：

~~~xml
<mirrors>
    <mirror>
        <id>alimaven</id>
        <name>aliyun maven</name>
        <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
        <mirrorOf>central</mirrorOf>
    </mirror>
    <!--阿里源中有些包可能没有，需要到中心仓库中下载-->
    <mirror>
        <id>repo1</id>
        <mirrorOf>central</mirrorOf>
        <name>central repo</name>
        <url>http://repo1.maven.org/maven2/</url>
    </mirror>
</mirrors>
~~~





Nexus3 是 Maven 的 Jar 包私有仓库。安装 Nexus3

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

- proxy：远程仓库，如果在本地仓库找不到 Jar 包的话，就从远程仓库中加载
- Hosted：本地仓库
- Group：仓库组（Nexus3 特有的概念），将多个仓库聚合在一起，这样用户在 pom.xml 文件中就不需要配置多个仓库地址了，仅需配置仓库组的地址即可

创建 Hosted 仓库时，要把 Deployment Policy 设置为 allow redeploy，否则只有 depolyment 用户才能上传 Jar 包，其他用户上传会返回 403 错误。







