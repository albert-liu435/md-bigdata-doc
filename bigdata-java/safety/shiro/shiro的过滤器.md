### shiro的过滤器

shiro与web进行集成的时候，主要通过过滤器来进行实现，下面是shiro的过滤器的继承体系

![4](.\pic\4.png)

**AbstractFilter：**shiro中filter的顶级抽线类，定义了init(FilterConfig filterConfig)方法，用来进行初始化。

**NameableFilter**:用于给filter设置一个名称，每一个filter都会有自己的名称。

**OncePerRequestFilter：**用于防止多次执行Filter的；也就是说一次请求只会走一次过滤器链；另外提供enabled属性，表示是否开启该过滤器实例，默认enabled=true表示开启。

**AbstractShiroFilter：**是Shiro的入口，是Shiro执行过程中的基类，并创建Subject实例，并绑定到当前线程ThreadLocal变量中，同时根据URL配置的filter，选择并执行相应的filter chain

**AdviceFilter**：AdviceFilter提供了AOP风格的支持，类似于SpringMVC中的Interceptor,主要增加了如下三个方法：

```java
//类似于AOP中的前置增强；在拦截器链执行之前执行；如果返回true则继续拦截器链；否则中断后续的拦截器链的执行直接返回；进行预处理（如基于表单的身份验证、授权）
boolean preHandle(ServletRequest request, ServletResponse response) throws Exception  
//类似于AOP中的后置返回增强；在拦截器链执行完成后执行；进行后处理（如记录执行时间之类的）；
void postHandle(ServletRequest request, ServletResponse response) throws Exception  
//类似于AOP中的后置最终增强；即不管有没有异常都会执行；可以进行清理资源（如接触Subject与线程的绑定之类的）；
void afterCompletion(ServletRequest request, ServletResponse response, Exception exception) throws Exception;   
```

**LogoutFilter:**用户退出登录的过滤器，主要用来处理用户退出登录时的一系列操作，如退出后进行重定向。

**PathMatchingFilter**：PathMatchingFilter提供了基于Ant风格的请求路径匹配功能及拦截器参数解析的功能，主要有如下两个方法

```java
//如果路径满足匹配规则，则返回ture
boolean pathsMatch(String path, ServletRequest request) 
//在preHandle中，当pathsMatch匹配一个路径后，会调用opPreHandler方法并将路径绑定参数配置传给mappedValue；然后可以在这个方法中进行一些验证（如角色授权），如果验证失败可以返回false中断流程；默认返回true；也就是说子类可以只实现onPreHandle即可， 无须实现preHandle。如果没有path与请求路径匹配，默认是通过的（即preHandle返回true）。    
boolean onPreHandle(ServletRequest request, ServletResponse response, Object mappedValue) throws Exception   
```

**AnonymousFilter**:匿名过滤器，对于所有的请求全部放行

**AccessControlFilter**：AccessControlFilter提供了访问控制的基础功能；比如是否允许访问/当访问拒绝时如何处理等：

```java
//表示是否允许访问；mappedValue就是[urls]配置中拦截器参数部分，如果允许访问返回true，否则false；
abstract boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) throws Exception;  
//表示当访问拒绝时是否已经处理了；如果返回true表示需要继续处理；如果返回false表示该拦截器实例已经处理了，将直接返回即可。
boolean onAccessDenied(ServletRequest request, ServletResponse response, Object mappedValue) throws Exception;  
abstract boolean onAccessDenied(ServletRequest request, ServletResponse response) throws Exception;   
```

**InvalidRequestFilter:**用于判断请求是否合法的过滤器

**UserFilter**:用户拦截器，表示该请求必须为用户

**AuthorizationFilter**：授权拦截器，表示当用户授予特定的权限的情况下才放行

**HostFilter：**主机拦截器，根据请求的host进行拦截

**HttpMethodPermissionFilter:**est风格拦截器，自动根据请求方法构建权限字符串（GET=read, POST=create,PUT=update,DELETE=delete,HEAD=read,TRACE=read,OPTIONS=read, MKCOL=create）构建权限字符串；示例“/users=rest[user]”，会自动拼出“user:read,user:create,user:update,user:delete”权限字符串进行权限匹配（所有都得匹配，isPermittedAll）；

**RolesAuthorizationFilter:**角色授权拦截器，验证用户是否拥有所有角色；主要属性： loginUrl：登录页面地址（/login.jsp）；unauthorizedUrl：未授权后重定向的地址；示例“/admin/**=roles[admin]”

**PortFilter:**端口拦截器，主要属性：port（80）：可以通过的端口；示例“/test= port[80]”，如果用户访问该页面是非80，将自动将请求端口改为80并重定向到该80端口，其他路径/参数等都一样

**FormAuthenticationFilter:**基于表单的拦截器；如“/**=authc”，如果没有登录会跳到相应的登录页面登录；主要属性：usernameParam：表单提交的用户名参数名（ username）；  passwordParam：表单提交的密码参数名（password）； rememberMeParam：表单提交的密码参数名（rememberMe）；  loginUrl：登录页面地址（/login.jsp）；successUrl：登录成功后的默认重定向地址； failureKeyAttribute：登录失败后错误信息存储key（shiroLoginFailure）；

**BasicHttpAuthenticationFilter:**Basic HTTP身份验证拦截器，主要属性： applicationName：弹出登录框显示的信息（application）；

### **示例代码**

下面以BasicHttpAuthenticationFilter为例整合springboot进行建单的身份认证请求

1、引入相应的依赖

```xml
  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <maven.compiler.source>1.8</maven.compiler.source>
    <maven.compiler.target>1.8</maven.compiler.target>

    <shiro.version>1.7.1</shiro.version>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

<!--    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>4.11</version>
      <scope>test</scope>
    </dependency>-->
    <dependency>
      <groupId>org.apache.shiro</groupId>
      <artifactId>shiro-core</artifactId>
      <version>${shiro.version}</version>
    </dependency>
    <dependency>
      <groupId>org.apache.shiro</groupId>
      <artifactId>shiro-spring</artifactId>
      <version>${shiro.version}</version>
    </dependency>

    <dependency>
      <groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
      <version>8.0.23</version>
    </dependency>
    <dependency>
      <groupId>com.mchange</groupId>
      <artifactId>c3p0</artifactId>
      <version>0.9.5</version>
    </dependency>
  </dependencies>
```

2、主要配置代码

```java
@Configuration
public class ShiroConfig {


    /**
     * 配置 C3P0的ComboPooledDataSource
     *
     * @return
     * @throws PropertyVetoException
     */
    @Bean
    public ComboPooledDataSource buildDataSource() throws PropertyVetoException {
        //设置数据库链接
        ComboPooledDataSource dataSource = new ComboPooledDataSource();
        dataSource.setDriverClass("com.mysql.cj.jdbc.Driver");
        dataSource.setJdbcUrl("jdbc:mysql://localhost:3306/shiro");
        dataSource.setUser("root");
        dataSource.setPassword("root");

        return dataSource;

    }


    @Bean
    public JdbcRealm buildRealm(ComboPooledDataSource dataSource) {
        //创建JdbcRealm
        JdbcRealm jdbcRealm = new JdbcRealm();
        jdbcRealm.setDataSource(dataSource);
        //securityManager.setRealm(jdbcRealm);
//        //认证的过程中，查看是否有相应的权限
//        jdbcRealm.setPermissionsLookupEnabled(true);

        return jdbcRealm;

    }


    @Bean("securityManager")
    public SecurityManager securityManager(JdbcRealm jdbcRealm) {
        DefaultWebSecurityManager securityManager = new DefaultWebSecurityManager();
        securityManager.setRealm(jdbcRealm);
        securityManager.setRememberMeManager(null);
        return securityManager;
    }


    @Bean("shiroFilter")
    public ShiroFilterFactoryBean shiroFilter(SecurityManager securityManager) {
        ShiroFilterFactoryBean shiroFilter = new ShiroFilterFactoryBean();
        shiroFilter.setSecurityManager(securityManager);

        Map<String, String> filterMap = new LinkedHashMap<>(8);
        //对所有的请求添加BasicHttpAuthenticationFilter过滤器
        filterMap.put("/**", "authcBasic");
        shiroFilter.setFilterChainDefinitionMap(filterMap);

        return shiroFilter;
    }


}
```

```java
@RestController
public class HelloController {


    @RequestMapping("/hello")
    public String hello(@PathVariable("id") Long id) {


        return "hello" + id;
    }


}
```

3、配置数据库数据

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
```

4、启动应用并采用postman进行调用即可，如图：

![5](.\pic\5.png)

相关示例代码参考[github](https://github.com/albert-liu435/rookie-springboot-shiro/tree/main/rookie-shiro-httpbasic)

