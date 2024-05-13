学习shiro之前，需要先了解RBAC权限模型，关于该模型的介绍，可以参考 SHUWOOM的博客   [RBAC权限系统分析、设计与实现](https://shuwoom.com/?p=3041)

### Apache Shiro简介

Apache Shiro 是一个强大而灵活的开源安全框架，可以方便地处理身份验证、授权、企业会话管理和加密。

Apache Shiro主要可以做如下的一些操作：

1、对用户进行身份验证以验证其身份
2、对用户执行访问控制，如 确定是否为用户分配了特定的角色，确定是否允许用户做某事
3、在任何环境中使用会话 API，即使没有 Web 或 EJB 容器。
4、在身份验证、访问控制或会话生命周期期间对事件做出反应。
5、启用单点登录 (SSO) 功能
6、无需登录即可为用户关联启用“记住我”服务
等等

### Shiro架构

​	从最简化的架构来看，shiro的架构如下图，主要包括 appliation ode，Subject,SecurityManager和Realms四个概念，其中Apllication Code可以认为外部系统

![ShiroBasicArchitecture](.\pic\ShiroBasicArchitecture.png)

**Subject:**请求主体。比如登录用户，比如一个被授权的app。在程序中任何地方都可以通过SecurityUtils.getSubject()获取到当前的subject。subject中可以获取到Principal，这个是subject的标识，比如登陆用户的用户名或者id等， shiro不对值做限制。但是在登录和授权过程中，程序需要通过principal来识别唯一的用户。

**SecurityManager:**安全管理器，可以理解成控制中心，所有请求最终基本上都通过它来代理转发，即所有与安全有关的操作都会与SecurityManager交互；且它管理着所有Subject；可以看出它是Shiro的核心，它负责与后边介绍的其他组件进行交互

**Realm:**realm可以访问安全相关数据，提供统一的数据封装来给上层做数据校验。shiro的建议是每种数据源定义一个realm，比如用户数据存在数据库可以使用JdbcRealm；存在属性配置文件可以使用PropertiesRealm。一般我们使用shiro都使用自定义的realm。 当有多个realm存在的时候，shiro在做用户校验的时候会按照定义的策略来决定认证是否通过，shiro提供的可选策略有一个成功或者所有都成功等。

#### 详细架构

![ShiroArchitecture](.\pic\ShiroArchitecture.png)

Subject,SeurityManager,Realm上面已经进行了介绍，下面主要介绍下面几个概念

**Authenticator:**认证器，主要负责主体的认证，如用户名与密码

**Authrizer**：授权器，或者访问控制器，用来决定主体是否有权限进行相应的操作；即控制着用户能访问应用中的哪些功能；

**SessionManager**：用来对web应用环境下的session进行管理

**SessionDAO**：用来对Session进行增删改查的操作，类似mybatis中的mapper

**CacheManager**：缓存控制器，来管理如用户、角色、权限等的缓存的；因为这些数据基本上很少去改变，放到缓存中后可以提高访问的性能

### 参考：

 [RBAC权限系统分析、设计与实现](https://shuwoom.com/?p=3041)

[shiro官方文档](https://shiro.apache.org/architecture.html)

[跟我学shiro](https://www.iteye.com/blog/jinnianshilongnian-2018936)

