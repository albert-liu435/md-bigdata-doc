### shiro AuthenticationToken体系

![03e91e13e89d4b0f31b1dd0bea908f3](.\pic\03e91e13e89d4b0f31b1dd0bea908f3.png)

如上图，为AuthenticationToken的继承体系，AuthenticationToken是在shiro进行身份验证时不可或缺的一部分，在验证是用户的身份(如：用户名)及凭证(如密码)过程中，都是从该AuthenticationToken获取的。

### AuthenticationToken

```java
//AuthenticationToken用于收集用户提交的身份（如用户名）及凭据（如密码）：
public interface AuthenticationToken extends Serializable {

    /**
     * 返回身份信息，相当于用户的用户名
     */
    Object getPrincipal();

    /**
     * 返回用户凭证信息，相当于用户的用户密码
     */
    Object getCredentials();
```

**RememberMeAuthenticationToken：**RememberMeAuthenticationToken在AuthenticationToken的基础上添加了记住我的功能

**HostAuthenticationToken：**在AuthenticationToken基础上添加了获取用户主机的功能

**UsernamePasswordToken：**shiro认证中最常用的一种，里面封装了用户的用户名，密码，用户主机，是否记住我的功能

**BearerToken：**里面封装了一个字符串token,该字符串token为用户的一些不敏感信息经过加密之后生成的，可以通过一定的规则解密来获取用户的信息，主要在web应用中结合自定义Realm使用，用于前后端分离，将字符串token放到header中传给系统进行验证。

[BearerToken的简单使用参考简单示例](https://github.com/albert-liu435/rookie-springboot-shiro/blob/main/rookie-shiro-demo/src/main/java/com/rookie/bigdata/CustomBearerToken.java)

[AuthenticationToken的中文注释参考](https://github.com/albert-liu435/shiro-root-1.7.1)

