---
layout: post
title: 证书及管理工具
category: 'other'
---

##  什么是证书？为什么要使用证书？
　　
对数据进行签名(加密)是我们在网络中最常见的安全操作。签名有双重作用，作用一就是保证数据的完整性，证明数据并非伪造，而且在传输的过程中没有被篡改，作用二就是防止数据的发布者否认其发布了该数据。

签名同时使用了非对称性加密算法和消息摘要算法，对一块数据签名时，会先对这块数据进行消息摘要运算生成一个摘要，然后对该摘要使用发布者的私钥进行加密。 比如微信公众平台开发中最常见的调用api接口方法是将参数进行字典序排序，然后将参数名和参数值接成一个字符串，组合的字符串也就相当于我们这里说的摘要，然后将摘要用平台密钥加密。

接收者(客户端)接受到数据后，先使用发布者的公钥进行解密得到原数据的摘要，再对接收到的数据计算摘要，如果两个摘要相同，则说明数据没有被篡改。同时，因为发布者的私钥是不公开的，只要接收者通过发布者的公钥能成功对数据进行解密，就说明该数据一定来源于该发布者。

那么怎么确定某公钥一定是属于某发布者的呢？这就需要证书了。证书由权威认证机构颁发，其内容包含证书所有者的标识和它的公钥，并由权威认证机构使用它的私钥进行签名。信息的发布者通过在网络上发布证书来公开它的公钥，该证书由权威认证机构进行签名，认证机构也是通过发布它的证书来公开该机构的公钥，认证机构的证书由更权威的认证机构进行签名，这样就形成了证书链。证书链最顶端的证书称为根证书，根证书就只有自签名了。总之，要对网络上传播的内容进行签名和认证，就一定会用到证书。关于证书遵循的标准，最流行的是 X.509。

##  证书管理

Keytool是一个Java数据证书的管理工具，这个工具一般在```${JAVA_HOME}\jre\bin\security\```目录下 。所有的数字证书是以一条一条(采用别名区别)的形式存入证书库的，证书库中的每个证书包含该条证书的私钥，公钥和对应的数字证书的信息。

证书库中的一条证书可以导出数字证书文件，数字证书文件只包括主体信息和对应的公钥。

Keytool将密钥（key）和证书（certificates）存在一个称为keystore的文件中。

在keystore里，包含两种数据： 
-   密钥实体（Key entity）— 密钥（secret key）又或者是私钥和配对公钥（采用非对称加密）
-   可信任的证书实体（trusted certificate entries）— 只包含公钥

###  1. 创建证书

``` 
keytool -genkeypair -alias "tomcat" -keyalg "RSA" -keystore "${HOME}/mykeystore.keystore"  \
-dname "CN=localhost, OU=localhost, O=localhost, L=SH, ST=SH, C=CN" -keypass "123456" \
-storepass -validity 180
```

参数说明：

-   -genkeypair  表示要创建一个新的密钥
-   -dname　　表示密钥的Distinguished Names，  表明了密钥的发行者身份
-   CN=commonName      注：生成证书时，CN要和服务器的域名相同，如果在本地测试，则使用localhost
-   OU=organizationUnit
-   O=organizationName
-   L=localityName
-   S=stateName
-   C=country
-   -keyalg　　　　使用加密的算法，这里是RSA
-   -alias　　　　　和keystore关联的别名，这个alias通常不区分大小写
-   -keypass　　    私有密钥的密码，这里设置为 123456
-   -keystore 　　  密钥保存在D:盘目录下的mykeystore文件中
-   -storepass 　　存取密码，这里设置为changeit，这个密码提供系统从mykeystore文件中将信息取出
-   -validity　　　　该密钥的有效期为 180天 (默认为90天)

下面是各选项的缺省值。 

-   -alias "mykey"
-   -keyalg "DSA"
-   -keysize 1024
-   -validity 90
-   -keystore 用户宿主目录中名为 .keystore 的文件
-   -file 读时为标准输入，写时为标准输出 

cacerts证书文件(The cacerts Certificates File),该证书文件存在于```${HOME}/jre/lib/security```目录下，是Java系统的CA证书仓库。

### 2.导出证书，由客户端安装

```

keytool -export -alias tomcat -keystore ${HOME}/mykeystore -file ${HOME}/mycerts.cer -storepass 123456

```

### 3.客户端配置：为客户端的JVM导入密钥(将服务器下发的证书导入到JVM中)

``` 
keytool -import -trustcacerts -alias tomcat -keystore "${JAVA_HOME}/jre/lib/security/cacerts " \
-file d:\mycerts.cer -storepass 123456

```

生成的证书可以交付客户端用户使用，用以进行SSL通讯，或者伴随电子签名的jar包进行发布者的身份认证。常出现的异常：“未找到可信任的证书”--主要原因为在客户端未将服务器下发的证书导入到JVM中，可以用：

``` 
keytool -list -alias tomcat -keystore "${JAVA_HOME}/jre/lib/security/cacerts" \
-storepass 123456

```
来查看证书是否真的导入到JVM中。

keytool生成根证书时出现如下错误：

```
keytool错误:java.io.IOException:keystore was tampered with,or password was incorrect
```

原因是在你的home目录下是否还有```.keystore```存在。如果存在那么把他删除掉，然后再执行或者删除```${JAVA_HOME}/jre/lib/security/cacerts``` 再执行。

数字证书中keytool命令使用说明
```
-alias           证书别名 
-keystore     指定密钥库的名称（cacerts是jre中默认的证书库名字，也可以使用其它名字 ）
-storepass   指定密钥库的密码 
-keypass     指定别名条目的密码 
-list        　　显示密钥库中的证书信息 
-v           　　显示密钥库中的证书详细信息 
-export      　将别名指定的证书导出到文件 
-file        　　指定导出到文件的文件名 
-delete     　  删除密钥库中某条目 
-import      　 将已签名数字证书导入密钥库 
-keypasswd   修改密钥库中指定条目口令 
-dname          指定证书拥有者信息 
-keyalg           指定密钥的算法 
-validity          指定创建的证书有效期多少天 
-keysize         指定密钥长度 
```

示例说明： 

-   浏览证书库里面的证书信息，可以使用如下命令： 
```
keytool -list -v -alias tomcat -keystore cacerts -storepass 666666 
keytool -list  -rfc -keystore e:/yushan.keystore -storepass 123456
keytool -list -v -keystore privateKey.store
```
缺省情况下，-list 命令打印证书的 MD5 指纹。而如果指定了 -v 选项，将以可读格式打印证书，如果指定了 -rfc 选项，将以可打印的编码格式输出证书
 
要删除证书库里面的某个证书，可以使用如下命令： 
```
keytool -delete -alias tomcat -keystore cacerts -storepass 666666 
```

要导出证书库里面的某个证书，可以使用如下命令： 

```
keytool -export -keystore cacerts -storepass 666666 -alias tomcat -file ${HOME}/server.cer 
keytool -export -alias tomcat -keystore ${HOME}/server.jks -file ${HOME}/server.crt -rfc -storepass 123456
```

>   备注： keystore.jks 也可以为 .keystore格式 ; server.crt也可以为 .cer格式；"-rfc" 表示以base64输出文件，否则以二进制输出。
 
要修改某个证书的密码（注意：有些数字认证没有私有密码，只有公匙，这种情况此命令无效）,这个是交互式的，在输入命令后，会要求你输入密码:
```
keytool -keypasswd -alias tomcat -keystore cacerts 
```

这个不是交互式的，输入命令后直接更改:

```
Keytool -keypasswd -alias tomcat -keypass 888888 -new 123456 -storepass 666666 -keystore cacerts
```

修改别名：
```
keytool -changealias -keystore mykeystore.keystore -alias 当前别名 -destalias 新别名
```

### 主流数字证书都有哪些格式

一般来说，主流的Web服务软件，通常都基于OpenSSL和Java两种基础密码库。

Tomcat、Weblogic、JBoss等Web服务软件，一般使用Java提供的密码库。通过Keytool工具，生成Java Keystore（JKS）格式的证书文件。
Apache、Nginx等Web服务软件，一般使用OpenSSL工具提供的密码库，生成PEM、KEY、CRT等格式的证书文件。
IBM的Web服务产品，如Websphere、IBM Http Server（IHS）等，一般使用IBM产品自带的iKeyman工具，生成KDB格式的证书文件。
微软Windows Server中的Internet Information Services（IIS）服务，使用Windows自带的证书库生成PFX格式的证书文件。
如何判断证书文件是文本格式还是二进制格式
您可以使用以下方法简单区分带有后缀扩展名的证书文件：

-   *.DER或*.CER文件： 这样的证书文件是二进制格式，只含有证书信息，不包含私钥。
-   *.CRT文件： 这样的证书文件可以是二进制格式，也可以是文本格式，一般均为文本格式，功能与 *.DER及*.CER证书文件相同。
-   *.PEM文件： 这样的证书文件一般是文本格式，可以存放证书或私钥，或者两者都包含。 *.PEM 文件如果只包含私钥，一般用*.KEY文件代替。
-   *.PFX或*.P12文件： 这样的证书文件是二进制格式，同时包含证书和私钥，且一般有密码保护。

您也可以使用记事本直接打开证书文件。如果显示的是规则的数字字母，例如：

```
—–BEGIN CERTIFICATE—–
MIIE5zCCA8+gAwIBAgIQN+whYc2BgzAogau0dc3PtzANBgkqh......
—–END CERTIFICATE—–
```
那么，该证书文件是文本格式的。

如果存在——BEGIN CERTIFICATE——，则说明这是一个证书文件。
如果存在—–BEGIN RSA PRIVATE KEY—–，则说明这是一个私钥文件。

