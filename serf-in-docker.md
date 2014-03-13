#在Docker中使用Serf

> 本文编写思路来自 [Decentralizing Docker: How to Use Serf with Docker](http://www.centurylinklabs.com/decentralizing-docker-how-to-use-serf-with-docker/)

在之前的 [Docker Link使用示例](http://my.oschina.net/marker/blog/200407) 中，我们对 [Docker](http://docker.io) 的link特性进行了简单的演示，这次的主题是使用 [Serf](http://www.serfdom.io/) 实现更加低耦合的容器关系结构，最终达到的效果是服务化各个服务。

##构建Serf镜像

在这我并没有使用 [Supervisord](http://supervisord.org) 来启动Serf服务，大家可以参照后面的示例来启动Serf。

**Dockerfile_serf**：

```
FROM ubuntu:12.04
MAINTAINER Marker.King <majk@vip.qq.com>
RUN echo "deb http://mirrors.aliyun.com/ubuntu precise main universe" > /etc/apt/sources.list
RUN apt-get update
RUN apt-get install -y wget unzip
RUN wget --no-check-certificate https://dl.bintray.com/mitchellh/serf/0.4.1_linux_amd64.zip
RUN unzip 0.4.1_linux_amd64.zip
RUN rm 0.4.1_linux_amd64.zip
RUN mv serf /usr/bin/
EXPOSE 7946 7373
CMD ["-tag", "role=serf-agent"]
ENTRYPOINT ["serf", "agent"]
```

**构建Serf镜像**：

```
$ docker build -t serf - < Dockerfile_serf
```

##构建mysql镜像

这一步比较复杂，我们将编写多个shell脚本，用于启动supervisor、启动mysql、创建数据库用户、创建数据库等，先让我们看一下目录结构

```
mysql +
      | - create_db.sh
      | - create_mysql_admin_user.sh
      | - Dockerfile
      | - join-cluster.sh
      | - my.cnf
      | - run.sh
      | - start-serf.sh
      | - start.sh
      | - supervisord-mysqld.conf
      | - supervisord-serf.conf
```

**create_db.sh**：

```
#!/bin/bash

read -r line
if [ -n "$line" ]; then
	echo "=> Creating database $line"
	mysql -uroot -e "CREATE DATABASE $line"
	echo "=> Done!"
else
	echo "Usage: $0 <db_name>"
	exit 1
fi
```

**create_mysql_admin_user.sh**：

```
#!/bin/bash

if [ -f /.mysql_admin_created ]; then
	echo "MySQL 'admin' user already created!"
	exit 0
fi

/usr/bin/mysqld_safe > /dev/null 2>&1 &

PASS=$(pwgen -s 12 1)
echo "=> Creating MySQL admin user with random password"
RET=1
while [[ RET -ne 0 ]]; do
	sleep 5
	mysql -uroot -e "CREATE USER 'admin'@'%' IDENTIFIED BY '$PASS'"
	RET=$?
done

mysql -uroot -e "GRANT ALL PRIVILEGES ON *.* TO 'admin'@'%' WITH GRANT OPTION"

mysqladmin -uroot shutdown

echo "=> Done!"
touch /.mysql_admin_created

echo "========================================================================"
echo "You can now connect to this MySQL Server using:"
echo ""
echo "    mysql -uadmin -p$PASS -h<host> -P<port>"
echo ""
echo "Please remember to change the above password as soon as possible!"
echo "MySQL user 'root' has no password but only allows local connections"
echo "========================================================================"
```

**join-cluster.sh**：

```
#!/bin/bash
exec serf join $SERF_AGENT_PORT_7946_TCP_ADDR:$SERF_AGENT_PORT_7946_TCP_PORT
```

**my.cnf**：

```
[mysqld]
bind-addres=0.0.0.0
```

**run.sh**：

```
#!/bin/bash
if [ ! -f /.mysql_admin_created ]; then
	/create_mysql_admin_user.sh
fi
exec supervisord -n
```

**start-serf.sh**：

```
#!/bin/bash
exec serf agent -tag role=db \
	-event-handler="user:create_db=/create_db.sh"
```

**start.sh**：

```
#!/bin/bash
exec mysqld_safe
```

**supervisord-mysqld.conf**：

```
[program:mysqld]
command=/start.sh
numprocs=1
autostart=true
autorestart=true
```

**supervisord-serf.conf**：

```
[program:serf]
command=/start-serf.sh
numprocs=1
autostart=true
autorestart=true

[program:serf-join]
command=/join-cluster.sh
autorestart=false
```

**Dockerfile**：

```
FROM ubuntu:12.04
MAINTAINER Marker King <majk@vip.qq.com>

RUN echo "deb http://mirrors.aliyun.com/ubuntu precise main universe" > /etc/apt/sources.list
RUN apt-get update
RUN apt-get -y upgrade
RUN ! DEBIAN_FRONTEND=noninteractive apt-get -qy install unzip supervisor mysql-server pwgen; ls

ADD https://dl.bintray.com/mitchellh/serf/0.4.1_linux_amd64.zip serf.zip
RUN unzip serf.zip
RUN rm serf.zip
RUN mv serf /usr/bin/

ADD /run.sh /run.sh
ADD /start.sh /start.sh
ADD /start-serf.sh /start-serf.sh
ADD /join-cluster.sh /join-cluster.sh
ADD /supervisord-mysqld.conf /etc/supervisor/conf.d/supervisord-mysqld.conf
ADD /supervisord-serf.conf /etc/supervisor/conf.d/supervisord-serf.conf
ADD /my.cnf /etc/mysql/conf.d/my.cnf
ADD /create_mysql_admin_user.sh /create_mysql_admin_user.sh
ADD /create_db.sh /create_db.sh
RUN chmod 755 /*.sh

EXPOSE 3306 7946 7373
CMD ["/run.sh"]
```

大家可能看到了，这里使用了`ADD https://dl.bintray.com/mitchellh/serf/0.4.1_linux_amd64.zip serf.zip`来添加一个远程文件到镜像中，这跟Serf服务镜像构建时使用的方法不一样，但是达到了同样的效果。在此我需要解释一下，`RUN wget <url>`是可以被缓存的，而`ADD <url> <name>`是不能够被缓存的，也就是说每次构建都会再次下载这个文件，`RUN wget <url>`则会使用缓存，从而加快构建速度。大家可以试一下分别构建两次镜像，可以发现Serf镜像第二次全部使用了缓存，而mysql会在Step 6再次下载文件。

进入mysql目录构建镜像：

```
$ docker build -t mysql .
```

##测试Serf连接

通过Serf镜像启动一个容器：

```
$ SERF_ID=$(docker run -d -p 7946 -p 7373 -name serf_agent serf)
```

让我们测试一下Serf是否正常工作，在我们的宿主机上(安装docker的机器)也安装Serf，来连接容器的Serf：

```
$ wget --no-check-certificate https://dl.bintray.com/mitchellh/serf/0.4.1_linux_amd64.zip
$ unzip 0.4.1_linux_amd64.zip
$ rm 0.4.1_linux_amd64.zip
$ sudo mv serf /usr/bin/
$ nohup serf agent &
$ serf members
packer-virtualbox    10.0.2.15:7946    alive
```

现在，在我们的机器上就拥有了一个可用的Serf，你可以连接Docker的Serf代理容器来看看效果：

```
$ serf join $(docker port $SERF_ID 7946)
Successfully joined cluster by contacting 1 nodes.
$ serf members
precise64        10.0.2.15:7946      alive    
9be517551dda     172.17.0.2:7946     alive    role=serf-agent
```

##测试Serf事件派发到mysql服务

让我们启动一个mysql容器，并与serf_agent连接，使用-link参数，注意格式为name:alias，这里alias必须使用serf_agent，因为在join-cluster.sh中固定调用了`SERF_AGENT_PORT_7946_TCP_ADDR`和`SERF_AGENT_PORT_7946_TCP_PORT`两个环境变量，可以按照不同的名称要求做连接，命令如下：

```
$ MYSQL_ID=$(docker run -d -p 3306 -p 7946 -p 7373 -link serf_agent:serf_agent mysql)
```

在宿主机中查看Serf集群情况：

```
$ serf members
precise64        10.0.2.15:7946     alive    
9be517551dda     172.17.0.2:7946    alive    role=serf-agent
410d5a7f709a     172.17.0.3:7946    alive    role=db
```

因为在mysql容器启动时，执行了join-cluster.sh脚本连接到了serf_agent，因此在宿主机中我们可以看到之后加入进来的mysql。

此时我们可以通过`docker logs $MYSQL_ID`查看mysql输出的用户名和密码信息(可以通过修改create_mysql_admin_user.sh来改变用户信息或输出格式等)。

```
$ docker logs $MYSQL_ID
=> Creating MySQL admin user with random password
=> Done!
========================================================================
You can now connect to this MySQL Server using:

    mysql -uadmin -paWQD8mdb3N1j -h<host> -P<port>

Please remember to change the above password as soon as possible!
MySQL user 'root' has no password but only allows local connections
========================================================================
```

执行自定义命令，通知mysql创建数据库：

```
$ serf event create_db wordpress
Event 'create_db' dispatched! Coalescing enabled: true
```
连接数据库，并查看数据库是否创建成功：

```
$ mysql -uadmin -paWQD8mdb3N1j -h127.0.0.1 -P49157
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| test               |
| wordpress          |
+--------------------+
5 rows in set (0.00 sec)
```

成功了！我们看到了数据库已经被创建。

#总结

今天的示例就写到这里，大家可以继续发散思考，我的mysql容器已经具备了接受命令的功能，那么如果我再创建一个wordpress容器，当容器启动时会发送create_db命令给serf_agent，这样mysql就达到了服务化的目的。当然能够服务化的还很多，例如：负载均衡、memcached、redis等等。如果大家有什么更好的想法，可以共享出来，我的邮箱地址是：majk@vip.qq.com