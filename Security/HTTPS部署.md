#### 前言
***
考虑到HTTP的安全性问题，现在很多网站已经将HTTP升级到了HTTP + SSL（HTTPS）。
但也并不是所有的HTTPS站点就是安全的，也可能存在中间人的攻击（不是权威的CA机构颁发的证书以及证书校验不严格）。下图就是关于“中间人攻击”的原理图。
![image](https://raw.githubusercontent.com/SexyPhoenix/Blog/master/static/Security/1.png)
不过权威CA机构颁发证书大多数是收费的，想用免费的可以考虑 Let's Encrypt。
什么才是权威呢？
就是CA机构向浏览器厂商申请，申请通过后，由浏览器厂商将CA机构的根证书（简称CA证书）内嵌在浏览器中。也就是那些为企业签发证书的CA证书都是受浏览器信任的。

而证书一般有三种，根证书、服务器证书、客户端证书。
根证书是生成服务器证书和客户端证书的基础，也就是CA证书。
服务器证书是放在服务器上的，并引入到站点的配置文件中，由CA证书签名。相当于有一封信件（服务器证书），由CA盖章（签名），表示此信件受CA信任。
客户端证书是对于个人的，这里不做演示。

这样就可以防御中间人攻击了，当客户端发起HTTPS请求时，服务器将服务器证书传给客户端，客户端用内嵌的CA证书和获取到的服务器证书做信息比较，如果发现是伪造的证书，客户端发出警告。

接下来就模拟下整个证书生成的环节，可以有一个清楚的认识。因为是本地环境，就自建CA根证书了（Let's Encrypt 有域名验证之类的步骤）。

#### 根证书（CA证书）
***
```
openssl version -a  //openssl所有安装信息
```
![image](https://raw.githubusercontent.com/SexyPhoenix/Blog/master/static/Security/2.png)
```
cd /usr/lib/ssl
```
![image](https://raw.githubusercontent.com/SexyPhoenix/Blog/master/static/Security/3.png)
```
cd  /etc/ssl  //到Linux专门的配置目录中设置CA
mkdir req //放服务器证书
mkdir newcert //放签名后的服务器证书
```
```
cp openssl.cnf cacert.cnf
```
```
vim  cacert.cnf
```
![image](https://raw.githubusercontent.com/SexyPhoenix/Blog/master/static/Security/4.png)
修改v3_ca 下面设置项。

![image](https://raw.githubusercontent.com/SexyPhoenix/Blog/master/static/Security/5.png)
修改v3_req的设置项， DNS参数值为要升级为HTTPS的域名。
开启 v3_req（去掉 #）。

生成根证书的私钥
```
openssl genrsa -aes256 -out private/cakey.pem 2048 //用-aes256加密生成cakey.pem私钥，密码记住后面要用
```
生成根证书CA （自签）
```
openssl req -new -x509 -subj "/C=CN/CN=FocusChina Corporation Root CA/ST=JiangSu/L=NanJing/O=FocusChina/OU=FC" -extensions v3_ca -days 3650 -key private/cakey.pem -sha256 -out cacert.pem -config cacert.cnf
```
查看CA证书
```
openssl x509 -in cacert.pem -text -noout
![6.png](https://upload-images.jianshu.io/upload_images/13834020-709773247db1696b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
Issuer与Subject一致
```
以上CA根证书建立完成，下面就可以给相应的服务器证书签名。

#### 服务器证书
***
```
cd /etc/nginx/ssl/  //这里将服务器证书放在 /etc/nginx/ssl 目录下。
```
生成服务器证书私钥 (www.app.goods)
```
openssl genrsa -out www.app.goods.key 2048
```
生成服务器证书
```
openssl req -subj "/C=CN/CN=app.goods/ST=JiangSu/L=NanJing/O=FocusChina/OU=FC" -extensions v3_req -sha256 -new -key www.app.goods.key -out www.app.goods.csr
```
CN的值为服务器名，其他的和根证书保持一致。
```
cp www.app.goods.csr /etc/ssl/req
```
服务器证书生成后，就可以将相关信息（公司信息、服务器证书，域名等）交给CA机构，CA机构会根据提供的信息去验证公司信息、域名是否属实。接下来，给服务器证书签名。

#### 签名
***
在前面已经建立了自己的CA证书，下面就用生成的CA证书给服务器签名。

签名
```
openssl x509 -req -extensions v3_req -days 3650 -sha256 -in ./req/www.app.goods.csr -CA cacert.pem -CAkey private/cakey.pem -CAcreateserial -out ./newcert/www.app.goods.crt -extfile cacert.cnf //用CA证书、CA私钥、服务器证书生成www.app.goods.crt，有效期10年
```
![image](https://raw.githubusercontent.com/SexyPhoenix/Blog/master/static/Security/7.png)

查看证书
```
openssl x509 -in ./newcert/www.app.goods.crt -text -noout
```
![image](https://raw.githubusercontent.com/SexyPhoenix/Blog/master/static/Security/8.png)
```
cp ./newcert/www.app.goods.crt /etc/nginx/ssl  //将签名后的证书交给服务器
```
#### 配置服务器
***
```
cd  /etc/nginx/conf.d
vim www.app.goods.conf
```
![image](https://raw.githubusercontent.com/SexyPhoenix/Blog/master/static/Security/9.png)
ssl 监听端口为443， 开启ssl，并加载服务器证书私钥以及证书。
```
service nginx restart //重启服务
```
https://www.app.good //chrome 打开网站 
![image](https://raw.githubusercontent.com/SexyPhoenix/Blog/master/static/Security/10.png)
页面出现“您的连接不是私密连接”，是因为自建的根证书或者服务器证书不被浏览器信任。

导出根证书
```
cd /etc/ssl
sz cacert.pem //发送到桌面。
```
Google 设置  高级 >  管理证书 受信任的根证书颁发机构  > 导入cacert.pem
运行 > certmgr.msc  //chrome用的是window系统的证书管理
![image](https://raw.githubusercontent.com/SexyPhoenix/Blog/master/static/Security/11.png)

刷新  https://www.app.goods/
![image](https://raw.githubusercontent.com/SexyPhoenix/Blog/master/static/Security/12.png)
chrome、IE等已成功

Firefox 用的不是window系统的证书管理，需要导入到浏览器
Firefox 选项   > 隐私与安全 查看证书  > 导入cacert.pem 证书颁发机构 （下载证书 勾选第一个框）
![image](https://raw.githubusercontent.com/SexyPhoenix/Blog/master/static/Security/13.png)
至此，HTTPS部署成功

#### 强制HTTP跳转
***
![image](https://raw.githubusercontent.com/SexyPhoenix/Blog/master/static/Security/14.png)
```
service nginx restart //重启服务
```
访问 http://www.app.goods 会调整到 https://www.app.goods/