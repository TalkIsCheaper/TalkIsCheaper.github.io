#            CAS单点登录(三)----多种认证方式

常用单向加密算法： MD5，SHA，HMAC。

加密类型：==NONE | DEFAULT | STANDARD | BCRYPT | SCRYPT | PBKDF2==

数据库前面的配置不变

```properties
#默认加密策略，通过encodingAlgorithm来指定算法，默认NONE不加密
# NONE|DEFAULT|STANDARD|BCRYPT|SCRYPT|PBKDF2
cas.authn.jdbc.query[0].passwordEncoder.type=DEFAULT
# 字符类型
cas.authn.jdbc.query[0].passwordEncoder.characterEncoding=UTF-8
# 加密算法
cas.authn.jdbc.query[0].passwordEncoder.encodingAlgorithm=MD5
# 加密盐
#cas.authn.jdbc.query[0].passwordEncoder.secret=
# 加密字符长度
#cas.authn.jdbc.query[0].passwordEncoder.strength=16
```

这时候之前的密码已经登录不进去了，因为我们加了md5加密模式，修改密码为md5模式后，再次填写密码，就可以进入了。

接着如图新增用户，密码都是用md5加密，而test的expired，test3的disabled都为1。

![1584617798138](C:\Users\刘修恒\AppData\Roaming\Typora\typora-user-images\1584617798138.png)

因此当我们登录test2或者test3时，会有以下提示。

![1584617825834](C:\Users\刘修恒\AppData\Roaming\Typora\typora-user-images\1584617825834.png)

![1584617854544](C:\Users\刘修恒\AppData\Roaming\Typora\typora-user-images\1584617854544.png)

除此之外，如果我们需要自定义加密类型，就需要实现 org.springframework.security.crypto.password.PasswordEncoder接口 ，并且把类名配置在passwordEncoder.type.

==新建 MyPasswordEncoder== 类

```java
public class MyPasswordEncoder implements PasswordEncoder {
    @Override
    public String encode(CharSequence charSequence) {
        //charSequence 为用户输入的密码
        return charSequence.toString();
    }

    @Override
    public boolean matches(CharSequence charSequence, String str) {
        // 当encode方法返回不为null时，matches方法才会调用，charSequence为encode返回的字符串
        // str字符串为数据库中密码字段返回的值
        String encodeStr = charSequence.toString() + "aa";
        if (encodeStr.equals(str)) {
            return true;
        }
        return false;
    }
}
```

更改配置

```properties
cas.authn.jdbc.query[0].passwordEncoder.type=encoding.MyPasswordEncoder
#这种情况下，即使启用了加密算法，也会不起作用，只是依靠自己编写的加密方式进行验证
```

### 二、Shiro认证

1、添加依赖包

```properties
<dependency>
  <groupId>org.apereo.cas</groupId>
  <artifactId>cas-server-support-shiro-authentication</artifactId>
  <version>${cas.version}</version>
</dependency>
```

2、在配置文件中添加

```properties
##
# Shiro配置
#
#允许登录的用户，必须要有以下权限，否则拒绝，多个逗号隔开
cas.authn.shiro.requiredPermissions=staff
#允许登录的用户，必须要有以下角色，否则拒绝，多个逗号隔开
cas.authn.shiro.requiredRoles=admin
#shir配置文件位置
#这种写法最好是在自己的tomcat中运行，不然找不到
cas.authn.shiro.location=classpath:shiro.ini  
#shiro name 唯一
cas.authn.shiro.name=cas-shiro
```

3、在resources下新建==shiro.ini==文件

```ini
[main]
cacheManager = org.apache.shiro.cache.MemoryConstrainedCacheManager
securityManager.cacheManager = $cacheManager

[users]
anumbrella = 123, admin
test = test, developer

[roles]
admin = system,admin,staff,superuser:*
developer = commit:*
```

4、去除之前的jdbc的认证方式，就可以在tomcat中启动项目了。

同样可以添加密码的加密方式，==注意==：和jdbc的加密方式配置不一样，不可以混淆

```properties
#默认加密策略，通过encodingAlgorithm来指定算法，默认NONE不加密
# NONE|DEFAULT|STANDARD|BCRYPT|SCRYPT|PBKDF2
cas.authn.shiro.passwordEncoder.type=DEFAULT
# 字符类型
cas.authn.shiro.passwordEncoder.characterEncoding=UTF-8
# 加密算法
cas.authn.shiro.passwordEncoder.encodingAlgorithm=MD5
```

