#How to use Docker links to *safely* connect from a Virgo application to a MongoDB
#利用Docker*链接(links)*实现Virgo应用到MongoDB的\*安全\*连接

***


Some months ago, the Docker Team announced Docker 0.6.5[1]. Since that release it is possible to give your container names – this makes it much easier to find the correct container to interact with. It didn’t take long that I got used to this new feature.  
  
  
几个月前，Docker 0.6.5版本[^1]发布。自该版本开始，你可以命名自己的container——也更方便找到想要的container。很快，这个功能我就上手了。

[^1]:http://blog.docker.io/2013/10/docker-0-6-5-links-container-naming-advanced-port-redirects-host-integration/

***

Together with the freshly introduced explicit linking this is a huge security improvement.  


container命名，配合新引入的链接(links)功能，大大改进了Docker的安全机制。
***
Links allow containers to discover and securely communicate with each other by using the flag -link name:alias[2].
Setting the new Docker daemon flag -icc=false disallows inter-container communication, if needed.  


链接(links),通过使用标记-link name:alias[^2]，在container之间实现安全的通信。在必要之时，设置新的守护进程(daemon)标记－icc=false，则会禁止container间的交互。
[^2]:http://docs.docker.io/en/latest/use/working_with_links_names/
***
Let’s give it a try and create an explicit link from an application running inside a Virgo container to a MongoDB running in another container.  

	$ docker run -name mongodb -t eclipsesource/mongodb
	$ docker run -link mongodb:db -name virgo -t eclipsesource/virgo-spring-325  

我们不妨实践下，显式建立一个链接(links)，从Virgo container中执行的应用，到另一个container里运行的MongoDB。

	$ docker run -name mongodb -t eclipsesource/mongodb
	$ docker run -link mongodb:db -name virgo -t eclipsesource/virgo-spring-325

***
Docker will provide some environment variables within the linked container:  

	DB_PORT_27017_TCP_ADDR  
	DB_PORT_27017_TCP_PORT


Docker提供了访问被链接container[^3]环境变量的方式：

	DB_PORT_27017_TCP_ADDR  
	DB_PORT_27017_TCP_PORT

[^3]:译者注，这里是运行着MongoDB的container

***
Those variable names are built from the exported port of the container mongodb and the second part of the link option given when starting the Virgo container.
For more information about linking please consult the Docker documentation section “Link Containers”  

这些环境变量的名字来源于两部分：MongoDB container所开放的端口；启动Virgo container的命令中，-link选项的第二部分[^4]。  
[^4]:译者注，即"-link mongodb:db"中的db。

更多关于链接的内容，可以参考Docker文档的"Link Containers"章节[^5]。
[^5]:http://docs.docker.io/en/latest/use/working_with_links_names/

***
Within the Virgo application you can access the environment variables when creating the MongoDB factory:

```
<context:property-placeholder location="classpath:mongodb.properties" />
<mongo:db-factory dbname="tabris-trial"
	host="${DB_PORT_27017_TCP_ADDR}:${DB_PORT_27017_TCP_PORT}" />
```   
```
<bean id="mongoTemplate" class="org.springframework.data.mongodb.core.MongoTemplate">
	<constructor-arg ref="mongoDbFactory" />
</bean>
```
The snippet above uses the defaults declared in mongodb.properties when no environment variables are present.
Happy linking…  
  
  

在Virgo应用里，你可以在创建MongoDB factory时，访问上述的环境变量：

```
<context:property-placeholder location="classpath:mongodb.properties" />
<mongo:db-factory dbname="tabris-trial"
	host="${DB_PORT_27017_TCP_ADDR}:${DB_PORT_27017_TCP_PORT}" />
```   
```
<bean id="mongoTemplate" class="org.springframework.data.mongodb.core.MongoTemplate">
	<constructor-arg ref="mongoDbFactory" />
</bean>
```
如若环境变量不可用，须像上面这段一样，采用mongodb.properties声明的默认配置。

先写到这，自己发掘去吧。。。

***

