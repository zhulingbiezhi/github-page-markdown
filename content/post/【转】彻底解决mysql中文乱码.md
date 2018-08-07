---
title: "【转】彻底解决mysql中文乱码"
date: 2018-08-07T11:35:45+08:00
weight: 170
keywords: ["mysql"]
description: "彻底解决mysql中文乱码"
tags: ["mysql"]
categories: ["mysql"]
author: "去去"
---

彻底解决mysql中文乱码
=============

mysql是我们项目中非常常用的数据型数据库。但是因为我们需要在数据库保存中文字符，所以经常遇到数据库乱码情况。下面就来介绍一下如何彻底解决数据库中文乱码情况。

### **1、中文乱码**

#### **1.1、中文乱码**

     create table user(name varchar(11));    # 创建user表
     insert into table user("carl");         # 添加数据
     select * from user;

![这里写图片描述](https://img-blog.csdn.net/20170312150004135?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjQxMDczMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

    insert into user value("哈哈");

**无法插入中文字符：**

![这里写图片描述](https://img-blog.csdn.net/20170312150059411?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjQxMDczMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

#### **1.2、查看表字符编码**

    mysql> show create table user \G;
    *************************** 1. row ***************************
           Table: user
    Create Table: CREATE TABLE `user` (
      `name` varchar(11) DEFAULT NULL
    ) ENGINE=InnoDB DEFAULT CHARSET=latin1
    1 row in set (0.00 sec)

**我们可以看到表的默认字符集是latin1.**  
**所以我们在创建表的时候就需要指定表的字符集:**

     create table user(name varchar(11)) default charset=utf8; 

**这样在Linux里面可以访问并且可以插入与访问这个表了。**

![这里写图片描述](https://img-blog.csdn.net/20170312151820544?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjQxMDczMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

#### **1.3、数据库与操作系统编码**

虽然在服务器端可以显示中文正常，但是在客户端可能会显示乱码。因为我们的服务器是UTF8。

![这里写图片描述](https://img-blog.csdn.net/20170312152019411?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjQxMDczMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

**而且数据库的编码也存在问题。**

![这里写图片描述](https://img-blog.csdn.net/20170312152204299?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjQxMDczMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

**这里我们可以看character\_sert\_database与character\_set\_server的字符集都是latin1.那么在mysql数据库中，server,database,table的字符集都默认是latin1.下面我们就来看看如何解决mysql乱码情况。**

### **2、mysql设置变量的范围**

#### **2.1、session范围**

查看数据库编码：

    show variables like '%char%';

![这里写图片描述](https://img-blog.csdn.net/20170312153042389?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjQxMDczMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

修改字符编码：

    set character_set_server=utf8;
    set character_set_database=utf8;
    show variables like '%char%';

![这里写图片描述](https://img-blog.csdn.net/20170312153312892?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjQxMDczMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)  
我们可以看到字符集已经修改成都是utf8了。但是这里有一个问题，那就是我们重新打开一个命令窗口然后查看数据编码就会出现下面的画面：

![这里写图片描述](https://img-blog.csdn.net/20170312153601693?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjQxMDczMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

#### **2.2、global范围**

mysql设置变量的范围默认是session范围。如果设置多个会话的字符集那么需要设置global范围:**Set \[global|session\] variables …**

    set global character_set_database=utf8;
    set global character_set_server=utf8;
    show variables like '%char%';

当我们跨会话查看mysql字符集都会看到都是utf8。如果你以为万事大吉了的话，那么你就大错特错了。

#### **2.3、设置数据全局范围**

当我们数据库重启的时候，你们发现设置global范围的值又变成latin1了。

    service mysqld restart
    mysql -uroot -pyourpassword
    show variables like '%char%';

![这里写图片描述](https://img-blog.csdn.net/20170312154546628?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjQxMDczMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

不要怕，下面就教你终极大招：

修改mysql配置文件/etc/my.cnf。

    [mysqld]
    character-set-server=utf8 
    [client]
    default-character-set=utf8 
    [mysql]
    default-character-set=utf8

请注意这几个参数配置的位置，不然可能会启动不起来mysql服务：

![这里写图片描述](https://img-blog.csdn.net/20170312160537963?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjQxMDczMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

OK。这下如果你重启mysql服务也会发现它的字符集是utf8.

![这里写图片描述](https://img-blog.csdn.net/20170312161517748?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjQxMDczMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

而且我们创建表的时候不需要指定字符编码,它默认就是utf8;

    drop database test;
    create database test;
    use test;
    create table user(name varchar(11));
    show create table user \G;

![这里写图片描述](https://img-blog.csdn.net/20170312161546359?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjQxMDczMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### **3、总结**

我看网上很多答案都是直接在session级别设置mysql的字符编码，这是治标不治本的方法。我们还是要从源头上解决这个问题。那就是修改mysql默认的配置文件，把它的字符集修改成能够使用中文字符的UTF8就OK了。

