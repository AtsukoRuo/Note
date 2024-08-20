## Harbor

Harbor 是一个用于存储和分发 Docker 镜像的企业级 Registry 服务器。Harbor 下载地址：https://github.com/goharbor/harbor/releases，选择 offline 版本的安装包

进入 Harbor 文件夹中，修改 Harbor.yml 配置文件，随便填写域名（禁止为 localhost）。在 /etc/hosts 中将这个域名映射到本地网卡上的地址即可（禁止填写为 127.0.0.1）。下面将配置 HTTPS

1. 配置主机名与 hosts 文件

   ~~~shell
   $ cat /etc/hosts
   127.0.0.1 localhost
   192.168.10.112 harbor.snow.com		
   
   # 修改主机名
   $ hostnamectl set-hostname harbor
   $ bash
   $ hostname
   harbor
   ~~~

2. 生成证书颁发机构证书及私钥

   ~~~shell
   cd /usr/local/src/harbor/certs
   
   openssl genrsa -out ca.key 4096
   
   openssl req -x509 -new -nodes -sha512 -days 3650 \
   -subj "/C=CN/ST=Shanghai/L=Shanghai/O=SmartX/OU=Lab/CN=harbor.snow.com" \
   -key ca.key \
   -out ca.crt
   ~~~

3. 生成服务器私钥及证书签名请求

   ~~~shell
   openssl genrsa -out harbor.snow.com.key 4096
   
   openssl req -sha512 -new \
   -subj "/C=CN/ST=Shanghai/L=Shanghai/O=SmartX/OU=Lab/CN=harbor.snow.com" \
   -key harbor.snow.com.key \
   -out harbor.snow.com.csr
   ~~~

4. 创建 x509 v3 扩展文件`v3.ext`

   ~~~txt
   authorityKeyIdentifier=keyid,issuer
   basicConstraints=CA:FALSE
   keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
   extendedKeyUsage = serverAuth
   subjectAltName = @alt_names
   
   [alt_names]
   DNS.1=harbor.snow.com
   DNS.2=snow.com
   DNS.3=harbor
   ~~~

5. 使用该v3.ext文件为 Harbor 服务器生成证书

   ~~~shell
   openssl x509 -req -sha512 -days 3650 \
   -extfile v3.ext \
   -CA ca.crt -CAkey ca.key -CAcreateserial \
   -in harbor.snow.com.csr \
   -out harbor.snow.com.crt
   ~~~

6. 将 harbor.snow.com.crt 转换为 harbor.snow.com.cert , 供 Docker 使用。Docker 守护进程将 .crt 文件解释为 CA 证书，.cert将文件解释为客户端证书。

   ~~~shell
   openssl x509 -inform PEM -in harbor.snow.com.crt -out harbor.snow.com.cert
   ~~~

7. Harbor 的配置文件修改如下：

   ~~~shell
   https:
     # https port for harbor, default is 443
     port: 443
     # The path of cert and key files for nginx
     certificate: /usr/local/src/harbor/certs/harbor.snow.com.cert
     private_key: /usr/local/src/harbor/certs/harbor.snow.com.key
   ~~~

   

尴尬的是，Docker 并不认我们自己生成的证书。因此，我们还要在 `/etc/docker/daemon.json`将 `harbor.snow.com` 域名设置为可信任的。

~~~shell
{
   "insecure-registries": ["harbor.snow.com:20014"],
}
~~~

~~~shell
$ systemctl daemon-reload
$ systemctl restart docker.service
~~~



此外，我们还要安装 Compose 

~~~shell
$ curl -L https://github.com/docker/compose/releases/download/v2.29.1/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
# 其中通过 `uname -s` 以及 `uname -m` 来获取应该下载哪个版本（linux x86_64）

$ chmod +x /usr/local/bin/docker-compose
~~~

通过 install.sh 脚本来启动 Harbor 服务。通过 localhost:80 来访问 Harbor 管理页面，默认用户名为 admin，密码为 Harbor12345。

在启动过程中，可能遇到 `Ports are not available` 错误。此时，我们仅需重启 Windows 上的 NAT 服务即可

~~~shell
$ net stop winnat
$ net start winnat
~~~





下面介绍如何将镜像推送到 harbor：

1. ~~~shell
   docker login http://harbor.snow.com
   ~~~

   docker login : 登陆到 一个Docker 镜像仓库，如果未指定镜像仓库地址，默认为官方仓库 Docker Hub。这里我们登录到 Harbor 私有镜像仓库中

2. 先给要推送的镜像打 Tag

   ~~~shell
   # hub.demo.com 是 harbor 的域名，test 是 harbor 中项目名，busybox:v1 是镜像的名称和版本号
   #busybox 是
   $ docker tag busybox:latest harbor.snow.com/test/busybox:v1
   ~~~

3. 然后再推送镜像

   ~~~shell
   $ docker push harbor.snow.com/test/busybox:v1
   ~~~


可以给用户分配以下权限：

1. 访客：对于指定项目拥有只读权限
2. 开发人员：对于指定项目拥有读写权限
3. 维护人员 ：对于指定项目拥有读写权限，创建 Webhooks
4. 项目管理员：除了读写权限，同时拥有用户管理/镜像扫描等管理权限