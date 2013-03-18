---
title: gitlab on centos
date: '2013-03-18'
description:
categories:
- gitlab
- git
tags:
- gitlab
- git
- centos
---

注:阅读前请慎重考虑,这是一场噩梦般的部署过程

### 1.添加EPEL源
不添加这个，什么依赖都装不了。所以，你懂得。这个是centos 5的，其他版本的可以去网上搜，就地址不一样。

	rpm -Uvh http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-5-4.noarch.rpm
	
注:epel源会经常更新,需要找到最新的

### 2.安装依赖

就是安装依赖，建议python自己编译安装一个，版本新一点。

	yum -y groupinstall 'Development Tools' 'Additional Development'
	yum -y install readline readline-devel ncurses-devel gdbm-devel glibc-devel tcl-devel openssl-devel curl-devel expat-devel db4-devel byacc sqlite-devel gcc-c++ libyaml libyaml-devel libffi libffi-devel libxml2 libxml2-devel libxslt libxslt-devel libicu libicu-devel system-config-firewall-tui python-devel redis
	
### 3.编译安装ruby

注意:千万不要用最新版，要用p327版本

	wget http://ftp.ruby-lang.org/pub/ruby/1.9/ruby-1.9.3-p327.tar.gz
	tar xfvz ruby-1.9.3-p327.tar.gz
	cd ruby-1.9.3-p327
	./configure --disable-install-doc --enable-shared --disable-pthread
	
编译前，如果可以的话，最好安装下qt,为毛要qt!如果不行请参阅官方的文档

	yum install qt-devel qtwebkit-devel
	export PATH=$PATH:/usr/lib32/qt4/bin   # 32位和64位
	make && make install
	
### 4.更新gem,安装rails
	gem update --system 
	gem update 
	gem install rails
	
### 5.Gitolite安装
这步非常重要,如果出错,后面前功尽弃

	adduser --shell /bin/bash --create-home --home-dir /home/gitlab gitlab
	adduser --system --shell /bin/sh --comment 'gitolite' --create-home --home-dir /home/git git
	sudo -u gitlab -H ssh-keygen -q -N '' -t rsa -f /home/gitlab/.ssh/id_rsa
	sudo usermod -a -G git gitlab
	
	cd /home/git
	sudo -u git -H git clone -b gl-v320 https://github.com/gitlabhq/gitolite.git /home/git/gitolite
	# Add Gitolite scripts to $PATH
	sudo -u git -H mkdir /home/git/bin
	sudo -u git -H sh -c 'printf "%b\n%b\n" "PATH=\$PATH:/home/git/bin" "export PATH" >> /home/git/.profile'
	sudo -u git -H sh -c 'gitolite/install -ln /home/git/bin'
	 
	# Copy the gitlab user's (public) SSH key ...
	sudo cp /home/gitlab/.ssh/id_rsa.pub /home/git/gitlab.pub
	sudo chmod 0444 /home/git/gitlab.pub
	 
	# ... and use it as the admin key for the Gitolite setup
	sudo -u git -H sh -c "PATH=/home/git/bin:$PATH; gitolite setup -pk /home/git/gitlab.pub"
	 
	# Make sure the Gitolite config dir is owned by git
	sudo chmod -R 750 /home/git/.gitolite/
	sudo chown -R git:git /home/git/.gitolite/
	 
	# Make sure the repositories dir is owned by git and it stays that way
	sudo chmod -R ug+rwXs,o-rwx /home/git/repositories/
	sudo chown -R git:git /home/git/repositories/
	
注:如果自定义目录的话后面会需要手动修改很多文件,如果不是必须的请就用home目录吧

测试gitolite安装

	sudo -u gitlab -H git clone git@localhost:gitolite-admin.git /tmp/gitolite-admin
	 
	sudo rm -rf /tmp/gitolite-admin
	
### 6. 安装Gitlab
	# We'll install GitLab into home directory of the user "gitlab"
	cd /home/gitlab
	# Clone GitLab repository
	sudo -u gitlab -H git clone https://github.com/gitlabhq/gitlabhq.git gitlab
	 
	# Go to gitlab dir 
	cd /home/gitlab/gitlab
	 
	# Checkout to stable release
	sudo -u gitlab -H git checkout 4-2-stable
	
注:我这里通过了4.2的安装,如果选用5.0master分支可能会安装失败

设置权限还有其他选项

cd /home/gitlab/gitlab
 
	# Copy the example GitLab config
	sudo -u gitlab -H cp config/gitlab.yml.example config/gitlab.yml
	 
	# 把其中的gitlab部分和ssh部分的host改成自己的域名就行了
	sudo -u gitlab -H vim config/gitlab.yml
	 
	# Make sure GitLab can write to the log/ and tmp/ directories
	sudo chown -R gitlab log/
	sudo chown -R gitlab tmp/
	sudo chmod -R u+rwX  log/
	sudo chmod -R u+rwX  tmp/
	 
	# Copy the example Unicorn config
	sudo -u gitlab -H cp config/unicorn.rb.example config/unicorn.rb
	
数据库设置

	sudo -u gitlab cp config/database.yml.mysql config/database.yml
	
安装Gems,用了n多的gem,请耐心等待

	cd /home/gitlab/gitlab
	 
	sudo gem install charlock_holmes --version '0.6.9'
	 
	# For mysql db
	sudo -u gitlab -H bundle install --deployment --without development test postgres
	 
	# Or For postgres db
	sudo -u gitlab -H bundle install --deployment --without development test mysql
	
	# 设置git全局设置
	sudo -u gitlab -H git config --global user.name "GitLab"
	sudo -u gitlab -H git config --global user.email "gitlab@localhost"
	 
	# 设置Hook脚本
	sudo cp ./lib/hooks/post-receive /home/git/.gitolite/hooks/common/post-receive
	sudo chown git:git /home/git/.gitolite/hooks/common/post-receive
	 
	# 初始化数据库
	sudo -u gitlab -H bundle exec rake gitlab:app:setup RAILS_ENV=production
	 
	# 安装初始化脚本，这是centos，ubuntu有对应的脚本
	sudo wget https://raw.github.com/gitlabhq/gitlab-recipes/master/init.d/gitlab-centos -P /etc/init.d/
	sudo chmod +x /etc/init.d/gitlab-centos
	chkconfig --add gitlab-centos
	
注:如果sudo -u gitlab无法执行指令,请su gitlab切换至gitlab用户进行才做,否则权限错乱导致安装失败

测试gitlab的状态，正常则启动

	# 查看环境信息
	sudo -u gitlab -H bundle exec rake gitlab:env:info RAILS_ENV=production
	 
	# 检测gitlab状态，非绿色的太多了，要注意修复下
	sudo -u gitlab -H bundle exec rake gitlab:check RAILS_ENV=production
	 
	# 启动
	sudo service gitlab start
	
### 7. Nginx配置

就是建立一个反向代理给unicon

	upstream gitlab {
	  server unix:/home/gitlab/gitlab/tmp/sockets/gitlab.socket;
	}
	 
	server {
	  listen 80;         # e.g., listen 192.168.1.1:80;
	  server_name Domain_NAME;     # e.g., server_name source.example.com;
	  root /home/gitlab/gitlab/public;
	 
	  # individual nginx logs for this gitlab vhost
	  access_log  /var/log/nginx/gitlab_access.log;
	  error_log   /var/log/nginx/gitlab_error.log;
	 
	  location / {
	    # serve static files from defined root folder;.
	    # @gitlab is a named location for the upstream fallback, see below
	    try_files $uri $uri/index.html $uri.html @gitlab;
	  }
	 
	  # if a file, which is not found in the root folder is requested,
	  # then the proxy pass the request to the upsteam (gitlab unicorn)
	  location @gitlab {
	    proxy_read_timeout 300; # https://github.com/gitlabhq/gitlabhq/issues/694
	    proxy_connect_timeout 300; # https://github.com/gitlabhq/gitlabhq/issues/694
	    proxy_redirect     off;
	 
	    proxy_set_header   X-Forwarded-Proto $scheme;
	    proxy_set_header   Host              $http_host;
	    proxy_set_header   X-Real-IP         $remote_addr;
	 
	    proxy_pass http://gitlab;
	  }
	}
	
### 8. 完成
期间可能会出现各种依赖问题和运行失败,只能靠你自己处理了,噩梦般的过程结束了,享受吧,注意看看git的hook是否正常工作,检查redis等等,post-recieve的问题还是没找到很好的解决办法,后面再来更新此文