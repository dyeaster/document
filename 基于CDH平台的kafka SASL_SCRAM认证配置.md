# kafka SASL/SCRAM认证配置

在CDH集群中配置kafka的鉴权遇到了很多问题，由于很多kafka的配置直接在CDH管理页面上做，但是很多kafka原生的参数不能直接在页面上配置，比如sasl.enabled.mechanisms和sasl.mechanism.inter.broker.protocol Broker这两个参数在页面上就无法直接配置，

另外security.inter.broker.protocol参数如果直接配置为SASL_SCRAM，则重启kafka会报错，如果配置了SASL需要同时开启kerberos认证

等等诸如此类的问题，经过多次试验之后，相关参数需要在kafka的高级参数中配置，同时在页面上去掉原有配置的参数才行。

## 软件版本信息：

> cdh版本：6.3.2
>
> kafka版本：2.2.1+cdh6.3.2



### 配置步骤：

1. 正常启动zookeeper和kafka
2. 创建用户，配置 SASL/SCRAM 的第一步，是创建能否连接 Kafka 集群的用户。在本次测试中，我会创建 3 个用户，分别是 admin 用户、writer 用户和 reader 用户。admin 用户用于实现 Broker 间通信，writer 用户用于生产消息，reader 用户用于消费消息。

我们使用下面这 3 条命令，分别来创建它们。

> However, in cases where you want Kafka brokers to authenticate to each other using SCRAM, and you want to create SCRAM credentials before the brokers are up and running, you must create SCRAM credentials for users in ZooKeeper using the `--zookeeper` option (you cannot use the `--bootstrap-server` option):
>
> 在执行下面的命令中，控制台会有如下提示
>
> `Warning: --zookeeper is deprecated and will be removed in a future version of Kafka.
> Use --bootstrap-server instead to specify a broker to connect to.`
>
> 直接忽略此提示信息，因为创建SCRAM认证信息是在zookeepr中，必须使用`--zookeeper`，不能使用`--bootstrap-server`参数

```shell
$ kafka-configs --zookeeper cdh6030:2181,cdh6031:2181,cdh6032:2181 --alter --add-config 'SCRAM-SHA-256=[password=admin-secret],SCRAM-SHA-512=[password=admin-secret]' --entity-type users --entity-name admin 
Completed Updating config for entity: user-principal 'admin'.
```



```shell
$ kafka-configs --zookeeper cdh6030:2181,cdh6031:2181,cdh6032:2181 --alter --add-config 'SCRAM-SHA-256=[password=writer],SCRAM-SHA-512=[password=writer]' --entity-type users --entity-name writer 
Completed Updating config for entity: user-principal 'writer'.
```



```shell
$ kafka-configs --zookeeper cdh6030:2181,cdh6031:2181,cdh6032:2181 --alter --add-config 'SCRAM-SHA-256=[password=reader],SCRAM-SHA-512=[password=reader]' --entity-type users --entity-name reader 
Completed Updating config for entity: user-principal 'reader'.
```

kafka-configs 脚本是用来设置主题级别参数的。其实，它的功能还有很多，可以使用下列命令来查看刚才创建的用户数据。

```shell
$ kafka-configs --zookeeper cdh6030:2181,cdh6031:2181,cdh6032:2181 --describe --entity-type users --entity-name writer 
Configs for user-principal 'writer' are SCRAM-SHA-512=salt=MWt6OGplZHF6YnF5bmEyam9jamRwdWlqZWQ=,storedkey=hR7+vgeCEz61OmnMezsqKQkJwMCAoTTxw2jftYiXCHxDfaaQU7+9/dYBq8bFuTio832mTHk89B4Yh9frj/ampw==,serverkey=C0k6J+9/InYRohogXb3HOlG7s84EXAs/iw0jGOnnQAt4jxQODRzeGxNm+18HZFyPn7qF9JmAqgtcU7hgA74zfA==,iterations=4096,SCRAM-SHA-256=salt=MWV0cDFtbXY5Nm5icWloajdnbjljZ3JqeGs=,storedkey=sKjmeZe4sXTAnUTL1CQC7DkMtC+mqKtRY0heEHvRyPk=,serverkey=kW7CC3PBj+JRGtCOtIbAMefL8aiL8ZrUgF5tfomsWVA=,iterations=4096 
```

这段命令包含了 writer 用户加密算法 SCRAM-SHA-256 以及 SCRAM-SHA-512 对应的盐值 (Salt)、ServerKey 和 StoreKey。

3. 为每个 Broker 创建一个对应的 JAAS 文件, JAAS 的文件内容如下：

```properties
KafkaServer { org.apache.kafka.common.security.scram.ScramLoginModule required username="admin" password="admin-secret"; };
```

关于这个文件内容，你需要注意以下两点： **不要忘记最后一行和倒数第二行结尾处的分号； JAAS 文件中不需要任何空格键。**

这里，我们使用 admin 用户实现 Broker 之间的通信。接下来配置 Broker 的 server.properties 文件, 下面这些内容，是需要每个节点单独配置的：

```properties
# 当topic没有配置acl权限时，是否允许所有用户访问
allow.everyone.if.no.acl.found=true
# 用于认证授权的程序类
authorizer.class.name=kafka.security.auth.SimpleAclAuthorizer
# 开启SCRAM 认证机制，并启用 SHA-256 算法
sasl.enabled.mechanisms=SCRAM-SHA-256
# 为 Broker 间通信也开启 SCRAM 认证，同样使用 SHA-256 算法；
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-256
# Broker 间通信不配置 SSL，
security.inter.broker.protocol=SASL_PLAINTEXT
# 设置 listeners 使用 SASL_PLAINTEXT，依然是不使用 SSL，需要配置为ip地址，否则无法监听外部请求
listeners=SASL_PLAINTEXT://192.168.220.144:9092
super.users=User:admin
```

4. 在bin/kafka-server-start.sh中配置启动参数

>  在最后的exec 之前添加jaas配置文件路径， base_dir为kafka中的bin目录，

```bash
export KAFKA_OPTS="-Djava.security.auth.login.config=$base_dir/../config/kafka_server_jaas.conf"
```

5. 修改以上配置之后，重启kafka

```shell
$ kafka-server-start config/server.properties
```

​	启动kafka之后，创建好对应的主题，然后给主题进行授权操作

 * 查看授权信息

   ```shell
   $ kafka-acls --authorizer-properties zookeeper.connect=cdh6030:2181,cdh6031:2181,cdh6032:2181 --list --topic test001
   ```

* 生产者授权

  ```shell
  $ kafka-acls --authorizer-properties zookeeper.connect=cdh6030:2181,cdh6031:2181,cdh6032:2181 --add --allow-principal User:writer --operation Write --topic test001
  ```

* 消费者授权

  ```shell
  $ kafka-acls --authorizer-properties zookeeper.connect=cdh6030:2181,cdh6031:2181,cdh6032:2181 --add --allow-principal User:reader --operation Read --topic test001

6. 测试生产者

> 在创建好测试主题之后，我们使用 kafka-console-producer 脚本来尝试发送消息。由于启用了认证，客户端需要做一些相应的配置。在producer.properties 配置文件中增加以下配置，内容如下：

```properties
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-256
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="writer" password="writer";
```

>  之后运行 Console Producer 程序：

```shell
$ kafka-console-producer --broker-list cdh6029:9092,cdh6030:9092,cdh6031:9092,cdh6032:9092 --topic test001 --producer.config config/producer.properties 
```

7. 测试消费者

> 使用 Console Consumer 程序来消费一下刚刚生产的消息。同样地，我们需要为 kafka-console-consumer 脚本创建一个名为 consumer.properties的脚本，内容如下：

```properties
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-256
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="reader" password="reader";
```

> 之后运行 Console Consumer 程序：

```shell
$ kafka-console-consumer --bootstrap-server cdh6029:9092,cdh6030:9092,cdh6031:9092,cdh6032:9092 --topic test001 --from-beginning --consumer.config config/consumer.properties 
```

### kafka tools(offset explorer)配置

> 在linux服务器kafka所在的节点1，192.168.220.141这个服务器上，这样配置可以连接上。但是在其他服务器，配置老是报连接不上zookeeper
>
> 配置对应ip地址的host之后问题解决
>
> 192.168.220.141 mint1
> 192.168.220.142 mint2
> 192.168.220.143 mint3

**按照如下配置，即可通过kafka tools连接上kafka集群**

1. 属性配置

   > 如果此处配置成本机的ip地址，也会报连接zookeeper超时
   >
   > 配置hosts文件之后，问题解决
   >
   > 192.168.220.141 mint1
   > 192.168.220.142 mint2
   > 192.168.220.143 mint3

   ![image-20211130105431822](https://github.com/dyeaster/document/blob/main/images/image-20211130105431822.png)

2. 安全配置

   ![image-20211130105459568](https://github.com/dyeaster/document/blob/main/images/image-20211130105459568-16382409048701.png)

3. 高级配置

   ![image-20211130105533934](https://github.com/dyeaster/document/blob/main/images/image-20211130105533934-16382409357092.png)

4. jaas属性配置

   ![image-20211130105551099](https://github.com/dyeaster/document/blob/main/images/image-20211130105551099-16382409530493.png)

   >org.apache.kafka.common.security.scram.ScramLoginModule required username="admin" password="admin-secret";

5. 连接上kafka集群之后，可以看到权限控制信息，也可以通过工具修改权限

   ![image-20211130105657194](https://github.com/dyeaster/document/blob/main/images/image-20211130105657194-16382410194454.png)

   

### 问题：

1. 需要将server.properties里面的listener监听配置为ip地址，否则本机只能使用localhost访问，ip地址无法访问，外部也访问不到，报无法找到broker	

   ```properties
# 设置 listeners 使用 SASL_PLAINTEXT，依然是不使用 SSL，需要配置为ip地址，否则无法监听外部请求
   listeners=SASL_PLAINTEXT://192.168.220.144:9092
   ```
   
   

1. 需要在server.properties中配置，鉴权java类，否则acl鉴权无法生效，所有用户均可以访问

   ```properties
   # 权限认证的java类
   authorizer.class.name=kafka.security.auth.SimpleAclAuthorizer
   ```

3. 通过kafka-acls.sh添加了消费者read权限之后，还需要添加消费者组的权限，否则会报错

   > 实际也不会报错，命令行的消费者，修改consumer.properties配置文件里面的消费者组之后，仍然可以正常消费，只是会从头开始消费kafka中的数据
   >

### 配置CDH集群的kafka权限

#### kafka配置

1. 创建用户

      ```bash
      $ bin/kafka-configs.sh --zookeeper localhost:2181 --alter --add-config 'SCRAM-SHA-256=[password=admin-secret],SCRAM-SHA-512=[password=admin-secret]' --entity-type users --entity-name admin 
      Completed Updating config for entity: user-principal 'admin'.
      ```

      

      ```shell
      $ bin/kafka-configs.sh --zookeeper localhost:2181 --alter --add-config 'SCRAM-SHA-256=[password=writer],SCRAM-SHA-512=[password=writer]' --entity-type users --entity-name writer 
      Completed Updating config for entity: user-principal 'writer'.
      ```

      

      ```shell
      $ bin/kafka-configs.sh --zookeeper localhost:2181 --alter --add-config 'SCRAM-SHA-256=[password=reader],SCRAM-SHA-512=[password=reader]' --entity-type users --entity-name reader 
      Completed Updating config for entity: user-principal 'reader'.
      ```

2. 修改super.admin配置，删除默认配置，置空，待会在高级配置中设置super.users

   ![image-20211203120348052](https://github.com/dyeaster/document/blob/main/images/image-20211203120348052.png)

3. 在各个节点服务器中创建kafka_server_jaas.conf文件，放置在/etc/kafka/conf目录下，文件内容如下

   ```properties
   KafkaServer { org.apache.kafka.common.security.scram.ScramLoginModule required username="admin" password="admin-secret"; };
   ```

   

4. 修改broker_java_opts，在最末尾加上启动变量java.security.auth.login.config，变量值为上一步jaas文件的绝对路径

   ```bash
   -Djava.security.auth.login.config=/etc/kafka/conf/kafka_server_jaas.conf
   ```

5. kafka高级配置，具体配置如下：

   **kafka.properties 的 Kafka Broker 高级配置代码段（安全阀）**

   ![image-20211203173639099](C:\Users\crrc\Documents\基于CDH平台的kafka SASL_SCRAM认证配置.assets\image-20211203173639099-16385242022006.png)

   default group全局配置

   ```properties
   # 当topic没有配置acl权限时，是否允许所有用户访问
   allow.everyone.if.no.acl.found=true
   # 用于认证授权的程序类(以下2个类都可以使用)
   authorizer.class.name=kafka.security.auth.SimpleAclAuthorizer
   #authorizer.class.name=kafka.security.authorizer.AclAuthorizer
   super.users=User:admin
   ```

   各节点配置，需要修改为各个节点的IP地址

   ```properties
   # 开启SCRAM 认证机制，并启用 SHA-256 算法
   sasl.enabled.mechanisms=GSSAPI,PLAIN,SCRAM-SHA-256,SCRAM-SHA-512
   # 为 Broker 间通信也开启 SCRAM 认证，同样使用 SHA-256 算法；
   sasl.mechanism.inter.broker.protocol=SCRAM-SHA-256
   # Broker 间通信不配置 SSL
   security.inter.broker.protocol=SASL_PLAINTEXT
   # 设置 listeners 使用 SASL_PLAINTEXT，依然是不使用 SSL，需要配置为ip地址，否则无法监听外部请求
   listeners=SASL_PLAINTEXT://10.12.36.29:9092
   advertised.listeners=SASL_PLAINTEXT://10.12.36.29:9092
   ```

6. 重启kafka

      

##### kafka-configs操作命令

创建用户

```bash
$ bin/kafka-configs.sh --zookeeper localhost:2181 --alter --add-config 'SCRAM-SHA-256=[password=admin-secret],SCRAM-SHA-512=[password=admin-secret]' --entity-type users --entity-name admin 
Completed Updating config for entity: user-principal 'admin'.
```

查看用户

   ```bash
   $ bin/kafka-configs.sh --zookeeper localhost:2181 --describe --entity-type users --entity-name writer 
   ```

   删除用户配置

   ```bash
   kafka-configs --zookeeper cdh6030:2181,cdh6031:2181,cdh6032:2181 --alter --entity-type users --entity-name admin --delete-config 'SCRAM-SHA-256'
   kafka-configs --zookeeper cdh6030:2181,cdh6031:2181,cdh6032:2181 --alter --entity-type users --entity-name admin --delete-config 'SCRAM-SHA-512'
   ```

1. 协议配置错误，

   > ```
   > java.lang.IllegalArgumentException: requirement failed: inter.broker.listener.name must be a listener name defined in advertised.listeners. The valid options based on currently configured listeners are SASL_PLAINTEXT
   > 	at scala.Predef$.require(Predef.scala:224)
   > 	at kafka.server.KafkaConfig.validateValues(KafkaConfig.scala:1486)
   > 	at kafka.server.KafkaConfig.<init>(KafkaConfig.scala:1462)
   > 	at kafka.server.KafkaConfig.<init>(KafkaConfig.scala:1117)
   > 	at kafka.server.KafkaConfig$.fromProps(KafkaConfig.scala:1097)
   > 	at kafka.server.KafkaServerStartable$.fromProps(KafkaServerStartable.scala:28)
   > 	at kafka.Kafka$.main(Kafka.scala:59)
   > 	at com.cloudera.kafka.wrap.Kafka$$anonfun$1.apply(Kafka.scala:92)
   > 	at com.cloudera.kafka.wrap.Kafka$$anonfun$1.apply(Kafka.scala:92)
   > 	at com.cloudera.kafka.wrap.Kafka$.runMain(Kafka.scala:103)
   > 	at com.cloudera.kafka.wrap.Kafka$.main(Kafka.scala:95)
   > 	at com.cloudera.kafka.wrap.Kafka.main(Kafka.scala)
   > ```


2. inter.broker.listener.name未配置

   > ```
   > java.lang.IllegalArgumentException: requirement failed: inter.broker.listener.name must be a listener name defined in advertised.listeners. The valid options based on currently configured listeners are SASL_PLAINTEXT
   > 	at scala.Predef$.require(Predef.scala:224)
   > 	at kafka.server.KafkaConfig.validateValues(KafkaConfig.scala:1486)
   > 	at kafka.server.KafkaConfig.<init>(KafkaConfig.scala:1462)
   > 	at kafka.server.KafkaConfig.<init>(KafkaConfig.scala:1117)
   > 	at kafka.server.KafkaConfig$.fromProps(KafkaConfig.scala:1097)
   > 	at kafka.server.KafkaServerStartable$.fromProps(KafkaServerStartable.scala:28)
   > 	at kafka.Kafka$.main(Kafka.scala:59)
   > 	at com.cloudera.kafka.wrap.Kafka$$anonfun$1.apply(Kafka.scala:92)
   > 	at com.cloudera.kafka.wrap.Kafka$$anonfun$1.apply(Kafka.scala:92)
   > 	at com.cloudera.kafka.wrap.Kafka$.runMain(Kafka.scala:103)
   > 	at com.cloudera.kafka.wrap.Kafka$.main(Kafka.scala:95)
   > 	at com.cloudera.kafka.wrap.Kafka.main(Kafka.scala)
   > ```

3. 

#### 其他建议

1. allow.everyone.if.no.acl.found参数

   此参数如果设置为true，未配置acl策略的topic则允许所有用户访问。建议配置为true，同时提供一默认用户，用户名和密码可以提供给所有相关方。如此，可以只针对指定主题来设置acl策略，未设置的就允许所有人通过默认用户来访问这部分topic。

2. 用户设置

   * 设置一个默认账户，提供给相关方来访问不需要权限管控的主题(需要配合上一条的参数设置为true使用)
   * 不使用的用户及时删除，或者收回对应的权限，保证数据安全
   * 当前用户的添加和删除，据我了解，只能通过命令行来解决，api的暂时还没找到。

3. 权限设置

   >权限配置当前有两个途径：
   >
   >1. 通过命令行的方式kafka-acls来配置
   >2. 通过kafka提供的api来配置

   * 授权的时候除了管理员之外尽量不要授予ALL的权限
   * 建议严格区分开消费者和生产者权限，控制权限粒度，比如生产者就只有write的权限，消费者就只有read的权限
   * 权限配置还可以设置黑白名单，比如只允许对应ip的用户来消费或者生产，以保证数据安全性
   * 不使用的权限及时删除

4. 

#### 参考文档

1. [kafka动态权限认证（SASL SCRAM + ACL）CSDN博客](https://blog.csdn.net/weixin_45682234/article/details/109158975)
2. [Kafka认证机制](http://liuxiaojun.github.io/kafka-authentication.html)
3. [Java版 Kafka ACL使用实战_芒果无忧的博客-CSDN博客](https://blog.csdn.net/qq_41203483/article/details/121405232)
4. 

