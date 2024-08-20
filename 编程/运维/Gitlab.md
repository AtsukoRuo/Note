## Gitlab

安装流程：

~~~shell
$ sudo apt-get install -y curl openssh-server ca-certificates postfix tzdata perl
# 在安装 postfix 时，/etc/postfix/main.cf 里的配置可能不符合要求。我们要修改该配置文件，然后重新安装


# 2. 添加 GitLab 存储库
# GitLab 存储库的镜像源如下（直接修改镜像源，无需执行下述命令）
# deb https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/apt/packages.gitlab.com/gitlab/gitlab-ce/ubuntu/ bionic main
# deb-src https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/apt/packages.gitlab.com/gitlab/gitlab-ce/ubuntu/ bionic main
$ curl -sS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash


$ apt install gitlab-ce
~~~

可以在 `vim /etc/gitlab/gitlab.rb` 中修改它的配置，然后执行`gitlab-ctl reconfigure`命令来应用配置。我们要修改以下配置：

~~~shell
external_url 'http://localhost:82'
nginx['listen_port'] = 82
~~~

通过`gitlab-ctl start`来启动服务。gitlab web 界面的用户名是 `root`，密码存储在 `/etc/gitlab/initial_root_password`





要开启邮箱通知功能，那么在配置文件中添加如下内容：

~~~shell
 gitlab_rails['smtp_enable'] = true
 gitlab_rails['smtp_address'] = "smtp.qq.com"
 gitlab_rails['smtp_port'] = 465
 gitlab_rails['smtp_user_name'] = "38687855@qq.com"
 gitlab_rails['smtp_password'] = "inflfllezgxqcafa"	 # qq的授权码
 gitlab_rails['smtp_domain'] = "mail.qq.com"
 gitlab_rails['smtp_authentication'] = "login"
 gitlab_rails['smtp_enable_starttls_auto'] = false
 gitlab_rails['smtp_tls'] = true
 gitlab_rails['smtp_pool'] = true
 
 gitlab_rails['gitlab_email_from'] = '38687855@qq.com'
 gitlab_rails['gitlab_email_display_name'] = 'AtsukoRuo'
~~~

测试是否成功配置邮箱：

~~~shell
$ gitlab-rails console

> Notify.test_email('38687855@qq.com','GitLab Test','HelloWorld').deliver_now
~~~





GitLab 中的 Group 就是对应公司中的项目组，其下可以有多个项目（Project）。组有三种访问权限：

- private：只有组成员可见
- internal：登录的用户可见
- public：所有的人可见。

我们可以邀请其他用户入组，并赋予它们权限：

- Guest：可以创建 issue、发表评论，不能读写版本库
- Reporter：可以克隆代码，不能提交
- Developer：可以克隆代码、开发、提交、push
- Maintainer：可以创建项目、添加tag、保护分支、添加项目成员、编辑项目
- Owner：可以设置项目访问权限、删除项目、迁移项目、管理组成员

## 同步到GitHub上