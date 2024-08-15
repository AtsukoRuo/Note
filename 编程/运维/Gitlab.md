## Gitlab

安装流程：

~~~shell
sudo apt-get install -y curl openssh-server ca-certificates tzdata perl

curl -sS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash

# EXTERNAL_URL 是你 gitlab 服务器的地址
sudo EXTERNAL_URL="http://localhost" apt install gitlab-ce=16.7.0-ce.0
~~~

1. 修改配置文件 `vim /etc/gitlab/gitlab.rb`，然后应用再配置：`gitlab-ctl reconfigure`

2. 启动服务`gitlab-ctl start`，

3. gitlab web 界面的用户名是 `root`，密码存储在 `/etc/gitlab/initial_root_password`

   

在配置文件中添加如下内容：

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

