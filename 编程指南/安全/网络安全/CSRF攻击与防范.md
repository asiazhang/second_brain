# CSRF攻击与防范

## 什么是CSRF攻击
CSRF全称是Cross-site request forgery(跨站请求伪造)。CSRF是一种利用当前用户的已认证状态，利用此在Web网站执行一些用户不想执行的操作。

## CSRF攻击有啥影响
在成功的CSRF里面，攻击者让受害者无意中执行了某些操作。比如，攻击者可以修改受害者的账户邮箱地址，或者修改账户密码，又或者发起一个资金转账。根据环境的不同，攻击者设置可以获得受害者的全部账户控制权限。如果用户拥有网站的高级管理员权限，你想想🐶…………

## CSRF是如何生效的
要CSRF生效，必须满足三个条件：
1. **有效的攻击操作**。攻击者来攻击系统的理由就是这个操作。这个可能是一个权限操作（比如修改其他用户权限），或者跟用户数据相关的操作（比如修改用户自身密码）。
2. **必须使用基于[[Cookie]]的Session机制**。由于此种攻击必须要发起一个或者多个HTTP请求，如果应用程序仅依赖基于Cookie的Session认证才会跟攻击者可乘之机。系统不能使用其他方式来验证用户请求是否有效。
3. **没有不可预期的参数**。请求必须不包含攻击者无法猜测的参数。以修改用户账号密码为例，如果请求中需要附带旧密码，这种情况攻击者就无法攻击成功。

从CSRF的生效条件来看，就是利用了Web系统中[[session]]可以跨域，客户端对API的鉴权太过于简单导致的。系统API在设计的时候，最开始就需要考虑：
1. 如何证明用户是有效的
2. 即使用户有效，用户的参数也必须是有效，并且没有被篡改的
3. 重要敏感API引入二次验证机制。

计算机中有一条安全原则：**永远不要相信用户输入。任何输入数据在证明其无害之前，都是有害的。**

### 举例说明CSRF攻击

假设一个应用程序允许用户修改他们账号的Email地址，当用户做这个操作时，发送的HTTP请求如下：

```http
POST /email/change HTTP/1.1  
Host: vulnerable-website.com  
Content-Type: application/x-www-form-urlencoded  
Content-Length: 30  
Cookie: session=yvthwsztyeQkAPzeQ5gHgTvlyxHfsAfE  
  
email=wiener@normal-user.com
```

这满足CSRF的三个攻击条件：

1. 攻击者对这个操作感兴趣，觉得有利可图。在这个操作后，攻击者可以发起密码重置流程，然后就能完全获取用户账号权限。
2. 系统使用基于[[cookie]]的session认证机制，并且没有其他token或者机制来跟踪用户session。
3. 攻击者可以非常简单的识别出完成攻击的参数请求。

三个条件都具备后，攻击者就可以构造这样一个html网页：

```html
<html>  
  <body>  
    <form action="https://vulnerable-website.com/email/change" method="POST">
      <input type="hidden" name="email" value="pwned@evil-user.net" />  
    </form>  
    <script>  
      document.forms[0].submit();  
    </script>  
  </body>  
</html>
```

当受害人访问这个网页后，发生如下流程：
1. 攻击者的网页给受影响的网站发送HTTP请求
2. 如果用户之前已经登陆了受影响网站，他们的浏览器就会自动将session cookie附带到HTTP请求中
3. 受影响网站会认为第二步发送的请求是合法用户请求，然后会修改用户的邮件地址为攻击者设置的邮件地址。

> 虽然CSRF通常见于基于Cookie的Session机制，但是在其他使用场景下也可能出现，比如使用[[HTTP Basic认证]]和使用证书进行认证。

## 如何发布一个CSRF漏洞

攻击者将攻击网页发布到网络上，然后诱使用户访问此网页。这部分涉及到[[社会工程学]]的知识，常见的方式包括：

1. 给用户发送钓鱼邮件，然后攻击网页跟受影响网站基本一致
2. 给用户发送钓鱼社交媒体消息
3. 通过黑客手段注入到常见的站点（比如用户评论网站）

> 有些网站漏洞过大，通过GET请求都能完成数据修改。在这种情况下，攻击者无需通过外部网站进行攻击，只需要构造一个受影响网站的恶意网址就行。比如在前面例子中，修改构建地址可以通过GET操作完成，攻击可能是这样的：
> `<img src="https://vulnerable-website.com/email/change?email=pwned@evil-user.net">`


## 如何防范CSRF攻击

防范CSRF攻击最好的方式就是请求中包含一个[[CSRF token]]。这个token满足：

1. 高不可预测性
2. 跟用户的Session相关
3. 在相关操作之前进行严格校验

这个方式让CSRF中三个条件中的第三个不满足，引入不可预测参数来防止攻击者进行参数构造。

