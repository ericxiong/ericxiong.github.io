---
layout: page
title: "PKIX path building failed."
description: ""
category:
tags: ["ssl", "https", "pki"]

---

团队内同学使用Feign Client调用外部https服务时程序报错，报无法构建有效的证书路径，
看到问题第一反应是证书不被信任，直接用浏览器上访问证书有效，一切正常，没有警告。

报错的具体具体信息为：

```java
sun.security.validator.ValidatorException: PKIX path building failed:
sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target
```

Google搜索大部分文章给出的原因是证书不被信任，需要把相应证书通过keytool导入到keystore。

通过`keytool -cacerts -list`（jdk11命令，其他版本参数可能不一样）查看jdk信任的根证书，
发现访问的目标服务器证书链的根证书存在于jdk受信根证书列表中，根证书的fingerprint也一致。

事情变的很诡异，按照我对PKI公钥体系的了解，如果根证书被信任，那根证书签发的中间证书以及中间证书签发的证书都是可信的，
除非证书处以过期或者吊销等无效状态，但这些证书都是有效的，并且浏览器显示整个证书链的证书都是可信任的。

![cert-chain](/images/posts/2019-08-21/certs.png)

在本地连接目标网站发现确实报证书验证失败，把url改为`https://baidu.com`后证书验证通过，
因为目标网站使用的正式是阿里云签发的免费证书，猜测是签发的证书有问题。
把url改为另一个使用了阿里云签发免费证书的网站，结果证书验证通过，猜测是目标网站服务器配置有问题。

开启调试器进行调试，`sun.security.validator#engineValidate`方法，证书链只有网站证书，没有中间证书，
而网站证书的issue不在可信CA证书列表中，导致无法构造出完整的证书链，报错。
联系目标网站管理员，检查部署的证书，的确缺失了中间证书，重新部署中间证书后，服务访问正常。

现在的问题是服务器只放置网站证书而未放置中间证书的情况下，浏览器为何显示所有证书可信。

经过一番搜索，X.509有一个概念是[AIA](https://tools.ietf.org/html/rfc5280#section-4.2.2.1)(Authority Information Access)，网站证书会带有证书签发者使用的证书的[URL](https://tools.ietf.org/html/rfc2585#section-3)，
浏览器可以通过网络下载该URL所指向的中间证书文件，从而最终构建出完整的证书路径。
[JDK也支持AIA](https://docs.oracle.com/javase/8/docs/technotes/guides/security/certpath/CertPathProgGuide.html#AIA)，但是默认不开启，需要使用系统属性开启。`sun.security.provider.certpath.Builder#USE_AIA`

> Support for the caIssuers access method of the Authority Information Access extension is available. It is disabled by default for compatibility and can be enabled by setting the system property com.sun.security.enableAIAcaIssuers to the value true.
If set to true, Sun's PKIX implementation of CertPathBuilder uses the information in a certificate's AIA extension (in addition to CertStores that are specified) to find the issuing CA certificate, provided it is a URI of type ldap, http, or ftp.

通常该选项不会开启，所以建议网站开启https时部署完整的证书链对应的证书，同时保证证书的顺序为证书链中各证书的顺序。

