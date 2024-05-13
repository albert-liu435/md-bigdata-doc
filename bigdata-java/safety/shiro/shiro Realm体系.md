### shiro Realm体系

![1](.\pic\1.png)

上图是整个Realm的继承和实现体系，Realm主要就是对用户的身份主体进行验证如（账号和密码）,其中顶层接口为Realm

Realm：主要对用户进行身份验证

```java
public interface Realm {

    /**
     * 返回一个唯一的Realm名字
     */
    String getName();

    /**
     * 判断此Realm是否支持此Token
     */
    boolean supports(AuthenticationToken token);

    /**
     * 根据Token获取认证信息
     * 进行身份认证，认证成功则返回AuthenticationInfo
     */
    AuthenticationInfo getAuthenticationInfo(AuthenticationToken token) throws AuthenticationException;
```

CachingRealm：在原有的基础上增加了对用户信息的缓存功能

AuthenticatingRealm：主要进行具体的认证工作

AuthorizingRealm：在认证的过程中添加了授权的功能，在实际的开发中主要采用这个进行认证与授权

JdbcRealm:认证的时候通过查询数据库的方式进行认证

其他为具体的实现类，可以参考shiro源码中文注释

### 下面以JdbcRealm具体的开发demo

1、创建数据sql语句

```sql
create table users (
                       id int auto_increment,
                       username varchar(20),
                       password varchar(20),
                       password_salt varchar(20),
                       constraint pk_users primary key(id)
) charset=utf8 ENGINE=InnoDB;

insert into users(username,password)values('zhangsan','123');
```

2、执行登录的demo

```java
//设置数据库链接
ComboPooledDataSource dataSource = new ComboPooledDataSource();
dataSource.setDriverClass("com.mysql.cj.jdbc.Driver");
dataSource.setJdbcUrl("jdbc:mysql://localhost:3306/shiro");
dataSource.setUser("root");
dataSource.setPassword("root");

//创建DefaultSecurityManager
DefaultSecurityManager securityManager = new DefaultSecurityManager();
SecurityUtils.setSecurityManager(securityManager);
//创建JdbcRealm
JdbcRealm jdbcRealm = new JdbcRealm();
jdbcRealm.setDataSource(dataSource);
securityManager.setRealm(jdbcRealm);

// get the currently executing user:
Subject subject = SecurityUtils.getSubject();

UsernamePasswordToken token = new UsernamePasswordToken("zhangsan", "123");

subject.login(token);
//不跑出异常说明登录成功
System.out.println("登录成功");

subject.login(token);
System.out.println();
//不跑出异常说明登录成功
System.out.println("登录成功");
```

