---
title: 前后端分离-JWT
date: 2019-04-17 15:26:17
tags: [JWT,前后端分离]
catalog: [前后端分离]
---

> 参考：[jwt.io](https://jwt.io/)
> 参考：[https://blog.csdn.net/bntx2jsqfehy7/article/details/79224042](https://blog.csdn.net/bntx2jsqfehy7/article/details/79224042)
> 参考：[https://blog.csdn.net/kevin_lcq/article/details/74846723](https://blog.csdn.net/kevin_lcq/article/details/74846723)
> 参考：[https://www.cnblogs.com/eret9616/p/9661314.html](https://www.cnblogs.com/eret9616/p/9661314.html)

# Cookie&Session

## HTTP的无状态性

&emsp;&emsp;HTTP 是无状态协议，它不对之前发送过的请求和响应的状态进行管理。也就是说，无法根据之前的状态进行本次的请求处理。假设要求登录认证的 Web 页面本身无法进行状态的管理（不记录已登录的状态），那么每次跳转新页面不是要再次登录，就是要在每次请求报文中附加参数来管理登录状态。

&emsp;&emsp;不可否认，无状态协议当然也有它的优点。由于不必保存状态，自然可减少服务器的 CPU 及内存资源的消耗。从另一侧面来说，也正是因为 HTTP 协议本身是非常简单的，所以才会被应用在各种场景里。
<!-- more -->

## Cookie技术的引入

&emsp;&emsp;如果让服务器管理全部客户端状态则会成为负担，保留无状态协议这个特征的同时又要解决类似的矛盾问题，于是引入了 Cookie 技术。Cookie 技术通过在请求和响应报文中写入Cookie信息来控制客户端的状态。

&emsp;&emsp;Cookie会根据从服务器端发送的响应报文内的一个叫做Set-Cookie 的首部字段信息，通知客户端保存 Cookie。当下次客户端再往该服务器发送请求时，客户端会自动在请求报文中加入Cookie 值后发送出去。

## 基于Session的登录

&emsp;&emsp;目前用户的认证多半是基于表单的认证，基于表单的认证一般会使用 Cookie 来管理Session（Session会话，Session代表着服务器和客户端一次会话的过程，直到Session失效（服务端关闭）或者客户端关闭时结束）。基于表单认证本身是通过服务器端的 Web应用，将客户端发送过来的用户ID和密码与之前登录过的信息做匹配来进行认证的。
&emsp;&emsp;但鉴于 HTTP 是无状态协议， 之前已认证成功的用户状态无法通过协议层面保存下来。 即无法实现状态管理， 因此即使当该用户下一次继续访问，也无法区分他与其他的用户。于是我们会使用Cookie 来管理 Session，以弥补 HTTP 协议中不存在的状态管理功能。

![](JWT/session.png)

### 同源策略

&emsp;&emsp;同源策略（Same origin policy）是一种约定，它是浏览器最核心也最基本的安全功能，如果缺少了同源策略，则浏览器的正常功能可能都会受到影响。可以说 Web 是构建在同源策略基础之上的，浏览器只是针对同源策略的一种实现。

&emsp;&emsp;只有满足以下三种情况浏览器才会认定为同源：

```textile
•    协议相同
•    域名相同
•    端口相同
```

### 同源问题解决

- JSOUP

  ```javascript
  $.ajax({
        dataType:'jsonp',
        data:'id=10',
        jsonp:'jsonp_callback',
        url:'http://www.yiwuku.com/getdata',
        success:function(){
        //dostuff
        },
    });
  ```

- CORS

  `CORS`需要浏览器和服务器同时支持。（**通过`@CrossOrigin`注解允许跨域，`origins`设置了允许跨域请求的域，`allowCredentials`允许设置和接受Cookie。**）

### 集群情况下的问题

&emsp;&emsp;Session 是存储在Web服务器（例如：Tomcat）中的，并针对每个客户端（客户），通过SessionID来区别不同用户的。Session是以Cookie技术或URL重写实现，默认以Cookie技术实现，服务端会给这次会话创造一个JSESSIONID的Cookie值。
&emsp;&emsp;但是一个显著的问题就是，在集群模式下如果通过Nginx负载均衡的时候，如果有一个用户登录的时候请求被分配到服务器A上，登录成功后设置的Session就会存放在服务器A上了，但是在服务器B上却没有该用户的Session数据，当用户再次发起一个请求的时候，此时请求如果被分配到服务器B上，则就不会查询到该用户的登录状态，就会出现登录失败的情况！
&emsp;&emsp;一种可以想到的方式就是将多个Web服务器上存储的Session统一存储到某一存储介质中，保证进集群中的每一台机器都可以看到所有相同Session数据，这里的同步体现在所有的Session存储在同一的存储介质里边。

- spring-session

  1、[使用Redis存储Nginx+Tomcat负载均衡集群的Session](http://mp.weixin.qq.com/s?__biz=MzI1NDQ3MjQxNA==&mid=2247483665&idx=1&sn=126be172d9804cdcf75a17b50dcd48fa&chksm=e9c5f8a0deb271b62898af1266da1bde81aed33b231b7c79216dd550d467401728bb6726f5bd&scene=21#wechat_redirect)

  2、[使用Spring Session和Redis解决分布式Session跨域共享问题](http://mp.weixin.qq.com/s?__biz=MzI1NDQ3MjQxNA==&mid=2247483939&idx=1&sn=f32abd7236fb53273b94799ea72ef79c&chksm=e9c5fb92deb27284c09a38e9c226171fbf5f8c30aec36d6e01a5d1b3a3a19a8e4974131f2053&scene=21#wechat_redirect)

  3、[Spring Session解决分布式Session问题的实现原理](http://mp.weixin.qq.com/s?__biz=MzI1NDQ3MjQxNA==&mid=2247483979&idx=1&sn=0565ac1d37ac9d10eefefbfb8a621378&chksm=e9c5fbfadeb272ec6d74d2c152feadd9c09a3b947e653b8063fae1d6b00c7a544ed4ff213e7e&scene=21#wechat_redirect)

# JWT([JSON Web Token](https://jwt.io/))

&emsp;&emsp;jwt是目前最流行的跨域认证解决方案;原理是`json对象`+`签名`。[Ebook](jwt-handbook-v0_14_1.pdf)

## JWT数据结构

![](JWT/jwt-1.png)

&emsp;&emsp;JWT包含三个由点（.）分隔的部分，它们是：

1. 头部（header）

2. 有效负载（payload）

3. 签名（signature）

   ```textile
   //Header.Payload.Signature
   eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
   ```

### header

&emsp;&emsp;Header 部分是一个 JSON 对象，描述 JWT 的元数据，通常是下面的样子。

```json
{
  "alg": "HS256",//令牌的类型

  "typ": "JWT"//散列算法

}
```

&emsp;&emsp;最后，将上面的 JSON 对象使用 `Base64URL` 算法转成字符串。

### Payload

&emsp;&emsp;Payload 部分也是一个 JSON 对象，用来存放实际需要传递的数据。JWT 规定了7个官方字段，供选用。

```textile
•    iss (issuer)：签发人
•    exp (expiration time)：过期时间
•    sub (subject)：主题
•    aud (audience)：受众
•    nbf (Not Before)：生效时间
•    iat (Issued At)：签发时间
•    jti (JWT ID)：编号
```

&emsp;&emsp;除了官方字段，还可以在这个部分定义私有字段，下面就是一个例子。

```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "admin": true
}
```

&emsp;&emsp;注意，JWT 默认是`不加密`的，任何人都可以读到，所以不要把秘密信息放在这个部分。这个 JSON 对象也要使用` Base64URL` 算法转成字符串。

### Signature

&emsp;&emsp;Signature 部分是对前两部分的签名，防止数据篡改。

&emsp;&emsp;首先，需要指定一个密钥（secret）。这个密钥只有服务器才知道，不能泄露给用户。然后，使用 Header 里面指定的签名算法（默认是 HMAC SHA256），按照下面的公式产生签名。

```java
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret)
```

&emsp;&emsp;算出签名以后，把 Header、Payload、Signature 三个部分拼成一个字符串，每个部分之间用"点"（.）分隔，就可以返回给用户。

### Base64URL

&emsp;&emsp;前面提到，Header 和 Payload 串型化的算法是 `Base64URL`。这个算法跟 `Base64 `算法基本类似，但有一些小的不同。

&emsp;&emsp;JWT 作为一个令牌（token），有些场合可能会放到 URL（比如 api.example.com/?token=xxx）。Base64 有三个字符`+`、`/`和`=`，在 URL 里面有特殊含义，所以要被替换掉：`=`被省略、`+`替换成`-`，`/`替换成`_`。这就是 Base64URL 算法。

## 案例

![](JWT/jwt-2.png)

这里依赖于开源库auth0

```maven
<dependency>
    <groupId>com.auth0</groupId>
    <artifactId>java-jwt</artifactId>
    <version>3.7.0</version>
</dependency>
```

### 用户认证请求

用户提交认证请求到服务器，服务器认证用户有效性并生成签名，签名方法如下：

```java
public static String sign(AuthToken authToken) {
    try {
        logger.debug("begin sign token");
        //随机生成加密算法

        Algorithm algorithm = generateAlgorithm(authToken);
        Date issuedAt = new Date();
        //根据配置文件的会话超时时间计算token失效时间

        Date expiresAt = DateUtils.addMinutes(issuedAt, authToken.getMaxAge());
        logger.debug("begin build jwt builder");
        //构建jwt

        JWTCreator.Builder builder = JWT.create()
                .withIssuer(ISSUER)
                .withIssuedAt(issuedAt)
                .withExpiresAt(expiresAt)
                //添加主题--学号

                .withSubject(authToken.getReaderBarCode())
                //添加私有字段--角色

                .withClaim(USER_ROLE, authToken.getUserRole());
        String token = builder.sign(algorithm);
        logger.debug("sign token success {0}", token);
        logger.debug("begin authToken.setRecommendId (md5hex(token)) 32 ");
        logger.debug("begin authToken.setBeginDate");
        authToken.setBeginDate(issuedAt.getTime());
        logger.debug("begin authToken.setExpiredDate");
        authToken.setExpiredDate(expiresAt.getTime());
        return token;
    } catch (UnsupportedEncodingException | NoSuchAlgorithmException e) {
        logger.error("sing error", e);
        return null;
    }
}
```

最终服务器将token保存在缓存中并返回给浏览器，浏览器将token保存在本地，每次请求要在header中携带token参数。

### 用户令牌验证

本例使用spring`拦截器`和自定义`注解`方式实现令牌校验。其中`Auth`为自定义注解。

```java
@Override
public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
    //auth 注解判断
    if (handler.getClass().isAssignableFrom(HandlerMethod.class)) {
        Auth authToken = ((HandlerMethod) handler).getMethodAnnotation(Auth.class);
        if (authToken != null) {
            //获取请求头中token字符串

            String token = request.getHeader(WebAuthUtils.JWT_TOKEN);
            if (StringUtils.isNotEmpty(token) && token.length() > WebAuthUtils.JWT_TOKEN_AUTH_PREFIX.length()) {
                //去除前缀

                token = token.substring(WebAuthUtils.JWT_TOKEN_AUTH_PREFIX.length());
                AuthVerificationRequest authRequest = new AuthVerificationRequest(token, authToken.acceptRole());
                //验证token和角色

                CertifiedUser certifiedUser = authService.verifyToken(authRequest);
                request.setAttribute(WebAuthUtils.WEB_READER, certifiedUser);
            } else {
                throw new ServiceException(ErrorCode.Common.NEED_AUTH, "请登录");
            }
        }
    }
    return super.preHandle(request, response, handler);
}
```

unsign方法

```java
public static void unsign(String token, AuthToken authToken) throws UnsupportedEncodingException, NoSuchAlgorithmException, InvalidKeySpecException {
    Algorithm algorithm = getAlgorithm(authToken);
    JWTVerifier verifier = JWT.require(algorithm)
            .withIssuer(ISSUER)
            .build(); //Reusable verifier instance

    verifier.verify(token);
}
```
