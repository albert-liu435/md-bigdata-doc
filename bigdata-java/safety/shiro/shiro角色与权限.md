### shiro角色与权限

shiro在认证过程中，会判断该用户是否含有一定的权限或者角色，其中位于顶层的接口为AuthorizationInfo，其继承体系如下：

![2](.\pic\2.png)

AuthorizationInfo:为用户在认证的过程中，会将相应的角色与权限封装到AuthorizationInfo中，后面会根据该信息进行权限与角色的校验。其中AuthorizationInfo主要封装了如下三个方法：

```java
public interface AuthorizationInfo extends Serializable {

    /**
     * 返回分配给相应主题的所有角色的名称。
     */
    Collection<String> getRoles();

    /**
     * 返回对应主体的所有的权限
     */
    Collection<String> getStringPermissions();

    /**
     * 返回权限
     */
    Collection<Permission> getObjectPermissions();
```

SimpleAuthorizationInfo:为AuthorizationInfo的默认建单实现，封装了角色与权限。

SimpleAccount：除了含有用户角色与权限的信息外，添加了用户主体的信息，如：账号与密码

### shiro用户主体

用户主体的校验顶级接口为AuthenticationInfo，其继承体系关系如下：

![3](.\pic\3.png)

SimpleAuthenticationInfo:里面封装了用户的主体身份和主体的凭证等信息

### shiro权限和角色认证示例代码如下

```sql
drop table if EXISTS users;

create table users (
                       id int auto_increment,
                       username varchar(20),
                       password varchar(20),
                       password_salt varchar(20),
                       constraint pk_users primary key(id)
) charset=utf8 ENGINE=InnoDB;

insert into users(username,password)values('zhangsan','123');

drop table if EXISTS user_roles;
create table user_roles(
                           id bigint auto_increment,
                           username varchar(100),
                           role_name varchar(100),
                           constraint pk_user_roles primary key(id)
) charset=utf8 ENGINE=InnoDB;

INSERT INTO user_roles VALUES(1,'zhangsan','admin');


drop table if EXISTS roles_permissions;

create table roles_permissions(
                                  id bigint auto_increment,
                                  role_name varchar(100),
                                  permission varchar(100),
                                  constraint pk_roles_permissions primary key(id)
) charset=utf8 ENGINE=InnoDB;


INSERT INTO roles_permissions VALUES(1,'admin','update');
```

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
//认证的过程中，查看是否有相应的权限
jdbcRealm.setPermissionsLookupEnabled(true);
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
//判断是否有相应角色
boolean admin = subject.hasRole("admin");
System.out.println(admin);

boolean update = subject.isPermitted("update");
System.out.println(update);
```

认证过程中主要调用

doGetAuthenticationInfo(AuthenticationToken token)：用于验证用户的身份的主体，主要为账号和密码
 doGetAuthorizationInfo(PrincipalCollection principals)：主要验证用户的角色与权限

[示例代码](https://github.com/albert-liu435/rookie-springboot-shiro)

