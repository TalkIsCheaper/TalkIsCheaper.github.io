# 		CAS单点登录(二) ----搭建基础服务

 [下载打包成WAR的代码架子](https://github.com/apereo/cas-overlay-template) 

### 一、运行war包

> 1. 更改pom.xml中的镜像地址，添加国内镜像地址
> 2. 在maven中执行clean 和 package
> 3. 将target中生成的war包放到tomcat中的webapp中，运行tomcat，在浏览器中访问 http://localhost:8080/cas/login ，出现登录页面
> 4. 默认账号：casuser  密码：Mellon

==但是在上述过程中，会有两个方框提示==

1. 我们的登陆不是安全的，并没有使用HTTPS协议，我们使用的仅仅是http
2. 我们的用户验证方式是静态文件写死的

### 二、配置证书

#### 1、生成证书

​		需要解决两个问题，首先是HTTPS，但是HTTPS是需要证书。

==制作证书==

		> 1. 使用jdk中的bin目录下的自带工具==keytool== ，在bin目录下进入cmd，输入
		>
		>    ```
		>    keytool -genkey -alias caskeystore -keypass 123456 -keyalg RSA -keystore thekeystore  
		>    -genkey表示：要生成一个证书
		>    -alias表示：证书别名
		>    -keyalg表示：RSA表示加密类型
		>    -keystore表示：生成的证书名称为
		>    ```
		>
		>    2.首先输入密钥库口令，然后在输入名字与姓氏时为具体路由地址，其余的按照情况填写即可
		>
		>    ![20180714180957729](D:\1\csdn\图片\20180714180957729.png)
		>
		>    3.导出数字证书
		>
		>    ```
		>    	keytool -export -alias caskeystore -keystore thekeystore -rfc -file cas.crt
		>    ```
		>
		>    输入先前的密钥库口令，然后在当前目录下生成具体的cas.crt数字证书
		>
		>    4.将数字证书导入jdk下的jre里
		>
		>    在这里可能会出现==导入证书的时候提示非法选项==这个错误，我怕这里的做法是在jre文件路径下运行cmd，去写清楚具体的bin路径，去bin路径下拿去数
		>
		>    ```
		>    keytool -import -alias caskeystore -keystore cacerts -file C:/"program files"/java/jdk1.8.0_131/bin/cas.crt -trustcacerts -storepass changeit
		>    ```
		>
		>    导入jdk时需要**默认密码changeit**。
		>
		>    4.配置DNS
		>
		>    部署在本地的CAS服务端，需要做一个本地映射
		>
		>    window以管理员身份修改 C:\Windows\System32\drivers\etc\hosts文件 
		>
		>    添加映射地址  
		>
		>    ```
		>    127.0.0.1       sso.anumbrella.net
		>    ```
		>
		>    5.配置Tomcat 
		>
		>    编辑Tomcat中conf下的server.xml文件
		>
		>    将8443端口配置文件打开，配置如下（添加前面刚刚生成keystore的地址和密钥）
		>
		>    ```xml
		>    <Connector port="8443" protocol="org.apache.coyote.http11.Http11NioProtocol"
		>                   maxThreads="150" SSLEnabled="true">
		>            <SSLHostConfig>
		>                <Certificate certificateKeystoreFile="C:/Program Files/Java/jdk1.8.0_131/bin/thekeystore"
		>                             type="RSA" certificateKeystoreType="JKS" certificateKeystorePassword="123456"/>
		>            </SSLHostConfig>
		>        </Connector>
		>    ```
		>
		>    重启tomcat，接着访问 https://sso.anumbrella.net:8443/cas/login  ，第一个提示已经没有了

### 三、更改配置

 cas-overlay-template是采用配置覆盖的策略来进行自定义的，因此需要==通过覆盖或者继承==某些类某些方法来实现自定义的需求。去Tomcat下webapps目录下面打开web-inf目录下的classess下复制application.properties文件。

![20180714181705460](D:\1\csdn\图片\20180714181705460.png)

这是CAS认证服务器的配置，在项目中新建==src/main/resource==文件，将复制的文件粘贴进去。

现在就可以通过修改改文件，对CAS服务器的默认配置进行修改。

```properties
#SSL配置
server.ssl.enabled=true
server.ssl.key-store=classpath:thekeystore
server.ssl.key-store-password=123456
#server.ssl.key-password=changeit 最好不要带上这个，启动的时候会报错。
server.ssl.keyAlias=caskeystore
```

同时需要将证书放在resources下，这样直接打包就可以使用了。

##### 打包方法：

> 1. 在maven中运行clean 
> 2. 在项目工程的文件上右键选择 run cmd shell
> 3. 输入  build.cmd run (运行)   也可以输入 build.cmd package (打包) 不过需要使用 java -jar cas.war运行war包

但是还需要继续修改CAS的配置，来确认是通过数据库来确认用户信息 

```properties
cas.authn.accept.users=casuser::Mellon	#默认用户
```

默认的认证方式为静态文件认证，现在改为使用mysql数据库的jdbc认证方式 

添加相关驱动在pom.xml中

```xml
<!--新增支持jdbc验证-->
                <dependency>
                    <groupId>org.apereo.cas</groupId>
                    <artifactId>cas-server-support-jdbc</artifactId>
                    <version>${cas.version}</version>
                </dependency>
                
				<dependency>
                    <groupId>mysql</groupId>
                    <artifactId>mysql-connector-java</artifactId>
                    <version>5.1.20</version>
                </dependency>
                
                <!--若不想找驱动可以直接写下面的依赖即可，其中包括HSQLDB、Oracle、MYSQL、PostgreSQL、MariaDB、Microsoft SQL Server-->
                <!--<dependency>
                    <groupId>org.apereo.cas</groupId>
                    <artifactId>cas-server-support-jdbc-drivers</artifactId>
                    <version>${cas.version}</version>
                </dependency>-->
                <!--在这里如果使用上面的默认的CAS综合驱动库，且没有写版本，就有可能造成数据库连接失败，所以这里建议使用mysql-connector-java 5.1.20 （视数据库和数据库版本而言），且没有引入过多的jar包  -->
```

现在更改application.properties

```properties
##
# CAS Authentication Credentials
#查询账号用户信息的sql语句，且字段里必须包含password字段
cas.authn.jdbc.query[0].sql=select * from user where username=?

#指定上面的SQL查询字段名（必须）
cas.authn.jdbc.query[0].fieldPassword=password

#指定过期字段，1为过期，若过期不可用
cas.authn.jdbc.query[0].fieldExpired=expired

#为不可用字段段，1为不可用，需要修改密码
cas.authn.jdbc.query[0].fieldDisabled=disabled

#数据库的连接
cas.authn.jdbc.query[0].url=jdbc:mysql://127.0.0.1:3306/cas?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&useSSL=false

#数据库dialect配置
cas.authn.jdbc.query[0].dialect=org.hibernate.dialect.MySQLDialect

#数据库用户名
cas.authn.jdbc.query[0].user=root

#数据库用户密码
cas.authn.jdbc.query[0].password=123456

#数据库事务自动提交
cas.authn.jdbc.query[0].autocommit=false

#数据库驱动
cas.authn.jdbc.query[0].driverClass=com.mysql.jdbc.Driver

#超时配置
cas.authn.jdbc.query[0].idleTimeout=5000

#默认加密策略，通过encodingAlgorithm来指定算法，默认NONE不加密
cas.authn.jdbc.query[0].passwordEncoder.type=NONE
#cas.authn.jdbc.query[0].passwordEncoder.characterEncoding=UTF-8
#cas.authn.jdbc.query[0].passwordEncoder.encodingAlgorithm=MD5
```

![1584616347640](C:\Users\刘修恒\AppData\Roaming\Typora\typora-user-images\1584616347640.png)

创建数据库，重新运行。再次访问 https://sso.anumbrella.net:8443/cas/login ，就没有了提示出现。