title: Ubuntu Nginx Passenger MySQL imagemagick Setup
date: 2015-10-19 06:07:36
tags:
thumbnailImage: http://res.cloudinary.com/dudqzxsbb/image/upload/v1444614834/apple-touch-icon-144x144-precomposed_bbmwvw.png
---


```
$ apt-get update
$ adduser <username>
$ adduser <username> sudo
$ sftp <username>@ip
$ put id_rsa.pub .
$ ssh <username>@ip
$ mv id_rsa.pub ~/.ssh/authorized_keys
$ gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3
$ sudo apt-get install curl
$ curl -sSL https://get.rvm.io | bash -s stable
$ source /home/deploy/.rvm/scripts/rvm
$ rvm install 2.1.0
$ gem install passenger 
$ rvmsudo passenger-install-nginx-module
$ /opt/nginx/sbin/nginx
$ sudo apt-get install libmysqlclient-dev mysql-server
$ sudo apt-get install graphicsmagick-libmagick-dev-compat
$ sudo apt-get install nodejs
$ sudo apt-get install git
$ gem install bundler
```
