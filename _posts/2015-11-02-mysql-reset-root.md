---
layout: post
title: "mysql重置root密码"
comments: true
categories:
- mysql
---

centos7下重置mysql root密码:

1. sudo systemctl stop mariadb.service
1. sudo mysqld_safe --skip-grant-tables &
1. mysql -uroot -p ，然后直接回车
1. use mysql;
1. update user set password=PASSWORD('12345678') where user="root";
1. quit
