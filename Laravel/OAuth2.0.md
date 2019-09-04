#### OAuth 2.0定义了四种授权方式
- 授权码模式（authorization code） 
- 简化模式（implicit）
- 密码模式（resource owner password credentials）
- 客户端模式（client credentials）
授权码模式是功能最完整、流程最严密的授权模式，本篇也是主要去理解这种模式

#### 授权码模式大概分为 5 个步骤
![image](https://raw.githubusercontent.com/SexyPhoenix/Blog/master/static/Laravel/12.png)
- 客户端（Client）向服务提供商（HTTP service）申请创建客户端（Client_id、Client_Secret）。
- 用户（Resource Owner）通过浏览器（User Agent）打开后,跳转到授权页，客户端要求用户授权。
- 用户同意给予客户端授权，返回授权码（Code）。
- 客户端通过授权码，向认证服务器（Authorization server）申请令牌（Access Token）。
- 客户端通过令牌，向资源服务器（Resource server）获取资源。

###### 1. 获取Code
```
response_type：表示授权类型，必选项，此处的值固定为"code"
client_id：表示客户端的ID，必选项
redirect_uri：表示重定向URL，可选项
scope：表示申请的权限范围，可选项
state：表示客户端的当前状态，可以指定任意值，认证服务器会原封不动地返回这个值。
```
###### 2. 返回Code（用户授权通过后返回到重定向URL）
```
code：表示授权码，必选项。
state：如果客户端的请求中包含这个参数，认证服务器的回应也必须一模一样包含这个参数。
```
######  3. 客户端向认证服务器申请Access Token
```
grant_type：表示使用的授权模式，必选项，此处的值固定为"authorization_code"。
code：表示获得的授权码，必选项。
redirect_uri：表示重定向URI，必选项，且必须与上面中的该参数值保持一致。
client_id：表示客户端ID，必选项。
client_secret ： 表示客户端密钥，必选项。
```
###### 4. 认证服务器返回Access Token
```
access_token：表示访问令牌，必选项。
token_type：表示令牌类型，该值大小写不敏感，必选项，可以是bearer类型或mac类型。
expires_in：表示过期时间，单位为秒。如果省略该参数，必须其他方式设置过期时间。
refresh_token：表示更新令牌，用来获取下一次的访问令牌，可选项。
scope：表示权限范围，如果与客户端申请的范围一致，此项可省略。
```
###### 5. 向资源服务器获取信息
```
headers.Accept ： media类型，固定值 “application/json”
headers.Authorization 授权，值为返回的token_type + 空格 + access_token
```
#### 疑问
###### 1. 获取code时，只传递了clent_id，redirect_url等值，服务提供商是怎么知道是哪个用户授权？
 授权时，你已经登录了服务提供商的网站或者会要求你登录。

###### 2. 客户端是怎么知道你已经授权？
 授权请求发出后，浏览器得到的是一个http的重定向响应，这个地址是你的redirect_url，同时返回code值

###### 3. 为什么要设置获取code后再去获取access_token
 是为了安全性，直接通过重定向传回access_token，但是HTTP 302是不安全的， 攻击者有可能会获取到access_token，而code不能获取资源，即使被截取也没什么用，client通过HTTPS以及密钥来获取access_token，以保证安全。

###### 为什么不直接用HTTPS重定向回client 
不是所有client都支持HTTPS，为了通用性 和安全性，才衍生出来这么一个code。







