# CSRF Token

一般来说CSRF Token都是由Web框架自动化生成，不过我们还是理解下远离比较好。

## CSRF token是啥

一个CSRF token是一个唯一的，秘密的，不可预测的值，它由服务端生成发送给客户端，客户端在之后的HTTP请求中包含这个token。当最后一个请求发送后，服务端对token进行校验，检查token是否缺失或者无效。

CSRF token可以防止CSRF攻击，攻击者无法预测出这个参数是啥，因此他也无法成功构建出一个完全合法的HTTP请求。

## 如何生成CSRF token

CSRF 令牌应该是强不可预测的。使用一个强密度的伪随机数生成器，用它被创建的时间+静态秘密作为种子。CSRF token的生命周期跟session保持一致。

类似于函数:

```
generate_csrf_token(random_num, num_create_time, secret): String
   value result = hash(random_num, num_create_time, secret)
   for(i=0;i++; i<256;)
      result = hash(value)
	  
   return result
```