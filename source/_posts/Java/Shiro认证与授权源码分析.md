---
title: Shiro认证与授权源码分析
date: 2018-1-2
categories: JAVA
---

这里使用官方提供的demo进行调试，进入源码分析。

官方demo地址：https://github.com/apache/shiro/tree/master/samples/quickstart

```java
public class Quickstart {

    private static final transient Logger log = LoggerFactory.getLogger(Quickstart.class);


    public static void main(String[] args) {
        Factory<SecurityManager> factory = new IniSecurityManagerFactory("classpath:shiro.ini");
        SecurityManager securityManager = factory.getInstance();

        SecurityUtils.setSecurityManager(securityManager);

        // 获取当前用户:
        Subject currentUser = SecurityUtils.getSubject();

        // 获取当前用户的会话 (不依赖于Web容器!!!)
        Session session = currentUser.getSession();
        session.setAttribute("someKey", "aValue");
        String value = (String) session.getAttribute("someKey");
        if (value.equals("aValue")) {
            log.info("Retrieved the correct value! [" + value + "]");
        }

        if (!currentUser.isAuthenticated()) {
            UsernamePasswordToken token = new UsernamePasswordToken("lonestarr", "vespa");
            token.setRememberMe(true);
            try {
                // 登录的时候进行身份认证
                currentUser.login(token);
            } catch (UnknownAccountException uae) {
                log.info("There is no user with username of " + token.getPrincipal());
            } catch (IncorrectCredentialsException ice) {
                log.info("Password for account " + token.getPrincipal() + " was incorrect!");
            } catch (LockedAccountException lae) {
                log.info("The account for username " + token.getPrincipal() + " is locked.  " +
                        "Please contact your administrator to unlock it.");
            }
            // 捕获异常
            catch (AuthenticationException ae) {
                //unexpected condition?  error?
            }
        }

        //打印身份标识 (这里的身份标识是用户名):
        log.info("User [" + currentUser.getPrincipal() + "] logged in successfully.");

        //测试用户角色:
        if (currentUser.hasRole("schwartz")) {
            log.info("May the Schwartz be with you!");
        } else {
            log.info("Hello, mere mortal.");
        }

        //测试用户权限
        if (currentUser.isPermitted("lightsaber:wield")) {
            log.info("You may use a lightsaber ring.  Use it wisely.");
        } else {
            log.info("Sorry, lightsaber rings are for schwartz masters only.");
        }

        //测试实例级别的用户权限:
        if (currentUser.isPermitted("winnebago:drive:eagle5")) {
            log.info("You are permitted to 'drive' the winnebago with license plate (id) 'eagle5'.  " +
                    "Here are the keys - have fun!");
        } else {
            log.info("Sorry, you aren't allowed to drive the 'eagle5' winnebago!");
        }
        currentUser.logout();

        System.exit(0);
    }
}
```

## 身份认证过程

官方文档中提供了一张身份认证的图，直接看这张图可能还不能完全掌握认证的过程。调试源码源码过后，再回头看这张图，这张图才会深深的烙印在脑海中。

![img](http://shiro.apache.org/assets/images/ShiroAuthenticationSequence.png)

### 1. 从Demo中的Subject.login方法开始

![登录](http://img-blog.csdn.net/20180104170408572?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

Subject接口实现类如下，这里demo不是Web环境，所以使用的实现类是DelegatingSubject：

![Subject实现类](http://img-blog.csdn.net/20180104170435371?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![DelegateSubject代理SecurityManager.login](http://img-blog.csdn.net/20180104170450197?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### 2. SecurityManager.login()

SecurityManager接口的实现类

![SecurityManager](http://img-blog.csdn.net/20180104170549107?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![DefaultSecurityManager](http://img-blog.csdn.net/20180104170605956?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![认证器](http://img-blog.csdn.net/20180104170630450?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### 3. Authenticator认证器

认证器的实现类，SecurityMananger也继承自Authenticator，通过查看AuthenticatingSecurityManager源码其实就是Authenticator的代理。而真正实现认证功能的Authenticator实现类只有一个ModularRealmAuthenticator，从类的名字可以看出这个认证器的实现原理——模块化认证器：一个Realm就是一个认证模块。

![Authenticator](http://img-blog.csdn.net/20180104170649555?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

认证器源码实现：

![AbstractAuthenticator](http://img-blog.csdn.net/20180104170709093?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

模块化认证器：

![模块化认证器](http://img-blog.csdn.net/20180104170731346?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

#### 单模块认证

官方提供的这个例子就是单模块认证(IniRealm,用户、角色、权限等信息保存在ini配置文件中)。

单模块认证很简单，它直接调用realm.getAuthenticationInfo方法。

![doSingleRealmAuthentication](http://img-blog.csdn.net/20180104170754624?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

#### 多模块认证

![doMultiRealmAuthentication](http://img-blog.csdn.net/20180104170825091?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### 4. 多模块认证策略AuthenticationStrategy

Shiro提供了三种认证策略

![认证策略](http://img-blog.csdn.net/20180104170851663?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
| `AuthenticationStrategy` 类               | 描述                                       |
| ---------------------------------------- | ---------------------------------------- |
| [`AtLeastOneSuccessfulStrategy`](http://shiro.apache.org/static/current/apidocs/org/apache/shiro/authc/pam/AtLeastOneSuccessfulStrategy.html) | 如果一个（或多个）Realm认证成功，才被认为是成功的。如果没有任何一个验证成功，则认证失败。 |
| [`FirstSuccessfulStrategy`](http://shiro.apache.org/static/current/apidocs/org/apache/shiro/authc/pam/FirstSuccessfulStrategy.html) | 只有从第一个成功验证的领域返回的信息将被使用，所有后面的Realm的认证信息将被忽略。如果没有任何一个验证成功，则认证失败。 |
| [`AllSuccessfulStrategy`](http://shiro.apache.org/static/current/apidocs/org/apache/shiro/authc/pam/AllSuccessfulStrategy.html) | 所有配置的Realm都必须认证成功，才能被认为是成功的。如果有任何一个认证不成功，则认证失败。 |

ModularRealmAuthenticator默认使用**AtLeastOneSuccessfulStrategy**。

如果想修改认证策略可以在ini文件中配置：

```ini
[main]
...
authcStrategy = org.apache.shiro.authc.pam.FirstSuccessfulStrategy

securityManager.authenticator.authenticationStrategy = $authcStrategy

...
```

### 5. 认证模块Realm的实现

Shiro提供了如下的认证模块实现类，在官方的这个Demo中，由于使用的是Ini配置文件的方式，所以使用的Realm是IniRealm。

通常我们的用户权限信息存储在数据库中，需要我们继承AuthenticatingRealm，并重载它的`doGetAuthenticationInfo`方法来从数据库中获取用户身份认证信息。但是大多数情况我们会继承AuthorizingRealm，因为它不仅仅包括认证，还包括授权过程，通过重载它的`doGetAuthorizationInfo`方法实现授权。

![Realm模块实现类](http://img-blog.csdn.net/20180104170923976?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## 授权过程

授权主要在调用Subject.hasRole或Subject.isPermitted等检查角色或权限的方法时触发。

![Subject检查权限的方法](http://img-blog.csdn.net/20180104170953862?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

官方也提供了一张图来描述授权的过程：

![img](http://shiro.apache.org/assets/images/ShiroAuthorizationSequence.png)

> 授权过程与身份认证的过程代码很类似(其实更简单)，我就不在文章中具体贴图了，感兴趣的读者可以自行调试。

