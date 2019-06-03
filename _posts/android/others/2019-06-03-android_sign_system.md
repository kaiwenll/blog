---
layout:      post
title:      "Android签名系统详解"
subtitle:   "深入介绍电子签名的设计原理"
navcolor:   "invert"
date:       2019-06-03
author:     "Cheson"
#header-img: "/img/website/home-bg.jpg"
catalog: true
tags:
    - Android
    - 签名
---


### 为什么需要电子签名

#### 水浒传给我们的启示
在《水浒传》第三十九回宋江在江州来到浔阳楼自饮自吃，题反诗两首。无为军通判黄文炳于浔阳楼上发现宋江反诗，蔡九知府下令捉拿。宋江装疯。蔡九知府闻知是近疯，把宋江下到死囚中，派戴宗去东京报告蔡太师。戴宗被朱贵领上梁山泊，吴用教戴宗赚萧让（书法家）金大坚（雕刻家）上山，假造蔡京回书。黄文炳通过书信中使用的印章识破了假信，将宋江和戴宗一同下入死囚牢，之后引出了梁山好汉大闹江州城。  
可见从古以来签名就是一项重要的验证发布者身份的方式。  
#### 电子签名的意义
在安卓上，电子签名的意义包含了两大部分的应用意义。第一个为文件来源的认证，包括了apk来源的识别，此项可以判断apk是什么渠道发布的；禁止签名不一致的apk进行更新；对需要系统权限的应用进行授权管理。第二个为防止数据被篡改：准确点来说签名机制并无法保证数据不被篡改，它能做的是在数据被篡改之后能被识别出来，从而使用方不会掉入数据被篡改的陷阱。  
### 如何设计电子签名

介绍了电子签名的意义，或者说是功能，那么我们根据这些功能来讨论下如何设计一套电子签名系统。   

#### 签名和加密的区别

首先来看下签名和加密的概念，签名是否等同于加密呢？思考这样一个现实场景，我们在签署一份文件的时候会把文件上的内容都改成别人看不懂的密文吗？显然不会，我们需要做的只是附属上一个能标示自己身份的东西，这里也就是签名。而加密是什么呢？看过谍战片的对电报应该都不陌生吧，电报就是用密码本对所要发送的内容进行加密，即使电报被获取也无法被解读。这个就是典型的加密。

签名和加密的区别就是，签名是在原有内容的基础上附属上一份别人无法篡改的信息，而加密就是对原有内容通过一种变换，生成别人无法识别的密文。而签名时也用到了加密技术。我们在签名时附属上签名信息时会用到加密，保证这份附属信息无法被轻易识别和篡改。  

#### 电子签名的特征

于是我们能得到电子签名的几个特征：1、唯一性：你的签名必须要唯一，这个是由它的功能需要决定的，如果出现两份相同的签名，那么电子签名的身份识别功能将会丧失；2、不破坏明文：签名不对明文进行加密，它所要做的是在明文上附加一份专属的信息。那么这里又出现了一个问题，如何确保一份签名是对应的这份明文呢？这个问题将在后面电子签名的设计时解决。3、难以破解：签名保证了唯一之后还不够，还需要无法被破解，否则其安全性也是存在风险的。4、有加密有解密：上面提到了附属的签名信息也会是经过加密的内容，有加密那就必须要有对应的解密，否则接收者无法识别签名的来源和认证。  

#### 电子签名的设计模型

根据电子签名的功能和特征，我们能大概抽象出一个电子签名的模型：首先，针对明文A，通过一种方法B，提取出对应于A的信息C，然后把C进行加密方法D处理后生成密文E，这里的E就是加密的签名内容了。它是通过对A的处理和解密生成的，要保证不同的A是不会得到相同的E。这里也就回答了上一节的问题：如何确保一份签名是对应的这份明文呢？得到加密的签名内容之后，通过工具F，将明文A和签名E写入到一起，得到最终签名后的内容G。G被接收到之后，首先将其中的A通过方法B再提取一次信息C，另外将E通过与加密方法D对应的解密方法D'解密得到信息C‘，然后将C和C'进行对比是否一致来确保内容A没有被篡改。

![sign_model.png](https://chendongqi.github.io/blog/img/2019-06-03-sign_system/sign_model.png)

#### 电子签名的要素

以上电子签名的设计模型也就是目前我们使用的最多的模型。这个模型中涉及到三个要素：1、提取明文A信息的方法B，我们实际使用到的是Hash算法，因为Hash算法可以计算出文件的散列值作为文件摘要，这个散列值和明文A是一一对应的。Hash算法有很多种，这里常用的是Sha1/Sha256。2、加解密用到的解密算法D和D’目前用到的是RSA算法。RSA算法用于对摘要信息进行加密解密的算法，是目前公认最安全的既可用于加密又能用于数字签名的算法。3、使用的F就是签名工具，在android源码中用到的是signapk，另外也可以使用jarsigner来签名。两者的区别是使用的秘钥对格式不同。这里我们主要看signapk的使用，其实在signapk中所做的事不止是把签名写入到明文中去，而且还参与了签名生成的过程，后面会专门介绍。  

### 电子签名实例

从一个签过名的包中来分析下电子签名具体是如何执行的。  

#### MINIFEST.MF

我们先来看下MANIFEST.MF，它是这个环节中的第一步。它的文件内容部分如下  

```xml
Manifest-Version: 1.0
Created-By: 1.0 (Android SignApk)

Name: system/lib/lib_omx_rtps_pipe_arm11_elinux.so
SHA1-Digest: B0ue12ES7ARHUqKOBXRd/PhJpLU=

Name: system/vendor/lib/libijkplayer.so
SHA1-Digest: xaDhy0KXn7gCusAajweldA0StUA=

Name: system/framework/webview/paks/nl.pak
SHA1-Digest: 9HkaDmNqPTx3g/oUEqLfRBYzjyQ=

Name: system/bin/zteconshell.sh
SHA1-Digest: T7CBWtbYANfc8Io52SMljQK7J4M=

Name: system/lib/libgnsdk_link.3.07.3.so
SHA1-Digest: iUYsCkfe9aBoUR8Z3PbC0XzrJCQ=
```

这是一个manifest文件，首先是版本号，然后是创建者，这里是Android SignApk，后面都是一样，包括了这个升级包中处理META-INF目录下的所有文件，第一行是文件名，第二行是SHA1-Digest，生成的规则是对此文件做sha1计算hash值，然后将hash值再做base64编码。这个过程是在signapk里通过代码执行的，当然这个规则涉及到的hash值和base64编码我们都可以手动计算，那么我们以“system/lib/lib_omx_rtps_pipe_arm11_elinux.so”这一项为例来做一次手动计算，看看结果是否相符。  

首先对升级包中的system/lib/lib_omx_rtps_pipe_arm11_elinux.so文件计算hash值，通过linux的sha1sum命令，计算结果为  

```xml
sha1sum system/lib/lib_omx_rtps_pipe_arm11_elinux.so
074b9ed76112ec044752a28e05745dfcf849a4b5  system/lib/lib_omx_rtps_pipe_arm11_elinux.so
```

然后是对hash值做base64编码，网上有很多在线转的工具，测试了结果都不行，还有一些是exe的转换工具，但是是直接将hash数据做转换，结果跟在线工具一样，都是因为输入的编码不正确。sha1sum生成的hash值是十六进制的，我这边用了window下的WinHex编辑器来创建文件，然后把hash值粘贴进去，注意选择编码格式为“ASCII Hex”  

![WinHex-FormatChoose.png](https://chendongqi.github.io/blog/img/2018-12-07-system_upgrade/WinHex-FormatChoose.png)  

![WinHex-ManifestData.png](https://chendongqi.github.io/blog/img/2018-12-07-system_upgrade/WinHex-ManifestData.png)  

将此文件保存之后，我们就开始祭出base64编码工具了，这里用到的也是在网上找到的，能够对文件中的十六进制hash值做编码的工具。  

![Base64-ManifestData.png](https://chendongqi.github.io/blog/img/2018-12-07-system_upgrade/Base64-ManifestData.png)    

这里的文件“333”就是刚在WinHex里保存的文件名，而“222.txt”可以随便新建一个文本文档，用来工具输出编码结果的，完成后就可以打开来查看结果。  

看到“222.txt”中记录的结果为  

![ManifestDataResult.png](https://chendongqi.github.io/blog/img/2018-12-07-system_upgrade/ManifestDataResult.png)    

对比MANIFEST.MF文件中的结果，是一样的。  

我们这里再同步叙述一条支线，签名系统就是为了防止apk或者升级包被篡改，那么我们站到一个攻击者的角度来思考下篡改MANIFEST.MF文件的可能性，通过上面手动操作的这一套，就应该知道了篡改是可行的。只要修改了包里的某个文件，然后将它的sha1值重新做一遍base64编码就可以了。

#### CERT.SF

CERT.SF文件是在MANIFEST.MF的基础上生成的，网上有这样一种说法

> 这是对摘要的签名文件。对前一步生成的MANIFEST.MF，使用SHA1-RSA算法，用开发者的私钥进行签名。在安装时只能使用公钥才能解密它。解密之后，将它与未加密的摘要信息（即，MANIFEST.MF文件）进行对比，如果相符，则表明内容没有被异常修改。

初看的时候被这种自圆其说的方式给说服了，直到后来看到了一种质疑，说是用不同的密钥对apk进行签名，查看生成的CERT.SF是一样的。然后我自己测试了一次，确如其说结果是一样的，再后来终于找到了正解，还是感慨国内论坛人云亦云的现象太严重了，绝知此事要躬行。

> https://stackoverflow.com/questions/4245303/android-sf-file
>
> The digests in the .SF file are computed by hashing the 3 lines of the 
> corresponding entry in the .MF file. The .RSA (or .DSA) file contains a 
> signature of the .SF file created from the signing private key, along 
> with the public certificate chain of the signing key. The .RSA (or .DSA)
> file is in a binary (i.e. non-human readable) format that can be 
> programmatically parsed with effort.

上面这段我们也可以用手动的方式来操作下，先来看下CERT.SF的部分内容  

```xml
Signature-Version: 1.0
Created-By: 1.0 (Android SignApk)
SHA1-Digest-Manifest: oCY/jG+8Kr43FXyMzOMLU52VeBM=

Name: system/lib/lib_omx_rtps_pipe_arm11_elinux.so
SHA1-Digest-Manifest: pFDu2GJrNQuqRw2KxCCOTAz0fJw=

Name: system/vendor/lib/libijkplayer.so
SHA1-Digest-Manifest: LxzRx6uB+RlVc+x6dy/KiHNwdLQ=

Name: system/framework/webview/paks/nl.pak
SHA1-Digest-Manifest: 8Qxi3iVzG4DBW3NaoWUEPdeQ3sU=

Name: system/bin/zteconshell.sh
SHA1-Digest-Manifest: 43UDRuC4A/eDXFeetfFS4994A3E=
```

它是一个签名文件，第一项里包含了签名版本、创建者和对MANIFEST.MF文件的加密信息，此信息的来源是通过对前面MANIFEST.MF文件用sha1sum做hash值计算，然后通过上面的WinHex和base64编码工具对此hash值做计算生成base64编码，结果就是这里的“oCY/jG+8Kr43FXyMzOMLU52VeBM=”  

后面的每一项跟MANIFEST.MF里的一一对应，我们挑第一项来模拟下SHA1-Digest-Manifest的生成。  

把MANIFEST.MF中的第一项复制到一个文本文件中

```xml
Name: system/lib/lib_omx_rtps_pipe_arm11_elinux.so
SHA1-Digest: B0ue12ES7ARHUqKOBXRd/PhJpLU=


```

注意后面要加两个“\r\n”，我开始是在linux下测试的，总是发现结果不对，后来查询了资料说是linux下将回车键按"\n"处理，所以文本文件计算出来的hash值始终不对。后来移步到windows下用文本工具就成功了。接下来就跟之前的一样了，计算hash值，然后计算base64编码，生成的结果就是“pFDu2GJrNQuqRw2KxCCOTAz0fJw=”。  

那么我们再来思考下CERT.SF是否能被攻破呢？在修改了MANIFEST.MF之后，手动修改CERT.SF应该也不是难事对吧。  

#### CERT.RSA

CERT.RSA文件是生成的包含签名信息的文件，来分析此文件之前先了解下是如何签名跟验证的。  

![ManifestDataResult.png](https://chendongqi.github.io/blog/img/2018-12-07-system_upgrade/ManifestDataResult.png)    

签名过程：对所有数据进行摘要计算（这个可能会有多种形式，而这里所谈论的摘要指的也就是CERT.SF），然后用私钥对CERT.SF进行加密，同时将签名证书信息也一起写入，生成一个加密后的数据文件，这里指的就是CERT.RSA。  

验证过程：对CERT.RSA通过公钥解密，解密结果和摘要信息（CERT.SF）对比，如果结果一致则验证通过。  

用到的密钥对可以是自己用keytool生成的，Android签名中证书的格式采用X.509标准的版本三，一对秘钥包括了私钥（.pk8）和公钥（x59.pem），该证书的格式标准如下  

![X509CA.jpeg](https://chendongqi.github.io/blog/img/2018-12-07-system_upgrade/X509CA.jpeg)  

可以用命令```keytool -printcert -file <*.x59.pem>```来看到公钥的信息，这里包含的是证书的基本信息。  

```xml
Owner: EMAILADDRESS=yuchunbo@hiksmart.com, CN=Simba, OU=Smart, O=CETHIK, L=Wu Chang, ST=Zhejiang, C=CN
Issuer: EMAILADDRESS=yuchunbo@hiksmart.com, CN=Simba, OU=Smart, O=CETHIK, L=Wu Chang, ST=Zhejiang, C=CN
Serial number: a283878a0a1f8d7f
Valid from: Mon Nov 09 12:23:27 CST 2015 until: Fri Mar 27 12:23:27 CST 2043
Certificate fingerprints:
	 MD5:  66:0C:41:77:08:3B:35:62:06:68:FE:3A:E2:C3:D8:FD
	 SHA1: 7E:DE:67:0A:E7:F9:B5:C0:66:22:7B:BE:91:AC:8A:83:62:E5:28:8C
	 SHA256: 93:45:02:00:A1:27:4B:59:9F:0B:D0:E8:43:7A:6B:90:7A:F4:E5:7C:82:06:10:90:62:04:2B:C9:29:BD:14:19
Signature algorithm name: SHA1withRSA
Subject Public Key Algorithm: 2048-bit RSA key
Version: 3

Extensions: 

#1: ObjectId: 2.5.29.35 Criticality=false
AuthorityKeyIdentifier [
KeyIdentifier [
0000: 94 1D 52 E5 8B B9 DE 60   84 58 19 63 64 DE 5E 98  ..R....`.X.cd.^.
0010: 22 40 1E 17                                        "@..
]
]

#2: ObjectId: 2.5.29.19 Criticality=false
BasicConstraints:[
  CA:true
  PathLen:2147483647
]

#3: ObjectId: 2.5.29.14 Criticality=false
SubjectKeyIdentifier [
KeyIdentifier [
0000: 94 1D 52 E5 8B B9 DE 60   84 58 19 63 64 DE 5E 98  ..R....`.X.cd.^.
0010: 22 40 1E 17                                        "@..
]
]
```

私钥的内容我们可以通过命令```openssl pkcs8 -in build/target/product/security/releasekey.pk8 -inform DER -nocrypt```来看到  

```xml
-----BEGIN PRIVATE KEY-----
MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQDEmbdtHsOFrdub
u0xuLas1hCjULRsszraNL/VT3lC5XfElRMltXM4I1hcrWZTn9/rcoyLQF+PKIMXw
hgziBgFP7DJ7BUaYAqopcm13V7gruxzzOoUrQVY/VaITQFVy9UfTfMKoOQdhVOBA
oeijN/ikaYIl3a6McijwEmp+gpqj7ju5HakHvx3kwYSirQUciJzFBXslh6sgcTOv
co2q8LFNYnCq+uEESKCfeo6aM5EqSYXIyiuh4KSc4qg237V7q3J54x3HZ6JqX4eV
6mP9zdIB8gsTEVas3J54a8fASrEH39CXtmwVa8fugic7MvlliZRnU4GG17o/HvYg
6zCcYFO/AgMBAAECggEActoSLC9Crf+pQcsjlWIcmQECESHgtEZ2oviXa01+/yuA
SvNqcPc8bjEUDAEjWnimFus+1S5/pn+K4z6MnCZB8fzcaL3mRbuYyOnORV/7eaCw
Au/3CBP9XLacHn8A7E2ajlReK4RVaWj6MQflLiTunq38mD5vUCEJBWbcy9dkm8r0
IvAk1sqe+PpPEZ8gEZJXFh4dGQ55rURuNnWgETz10PnTvZRuwgfOWuWiHNPJA3tO
KzsNjf+etGCe5Zsc8ZT0T9nZKVXhXELizIHozSO7QAADrbRkyO+Y9bd/BDMSAaev
w3cvs6ABWFyF6JqZhu+gqCrNrod1BSLI8QEhHBWXAQKBgQDzNirpF93ezlVwrlDf
s5fuH5EVo/a351AFiBEnNca9G1mwwTTiWmmwgAn9YSEn8+zJDrl1SU0pYCe6D5n8
W4QHsppwplINA/Okpv95ZgAicv0wQSmkr6RG0P8TnlAFkmLbdXyTHluKWIja/MPY
iTgHcjb3RbiX1g4kQqY04YqjwQKBgQDO8B+ysQRP5nHNIaPGMHMhu0jpoyKoY66Q
QKE9VG7l8LIni3cy15APrbBSukhpkKDLgByb6fJODL0nsjD9w8YTU6jZL3roC2O4
yRgOE/fxJD6b9IcBRNtcQkgEOdKdu70cpK/bU6Iv+Dn0Y43irdpYmnkAkfDMw/oV
ypINJZLXfwKBgGLyq7yPeDXYjkw8ryyD3ZEEiLtcLNkfI6BMfmYMa+GuCexufnyE
ujETtny+koW1qKUX933vJ5RoyWDaThSsiuey00B3ejRPYkWfp5qVVAKv87A5Ip8c
0mH5T32E0BukNdIBV4BnPmjnoi4t3ePv17q3zgMF+5bSgIhiEUq8Y/JBAoGAFitZ
af5W1Ox+Mpiw//F+1BVJWWZVty5+rAuQeo6KFu4zV9M0IOlBELztz98PFOgeoc6G
whlNERmCRjdr0jPgC4AB7cqNY0CdHVXF0vRGsrnMT07iC7vBuF+NcY50RtuvBduK
z3dlP7hbFRh5QdiYNLfP0MTRxE4Wg7Eg9nGZCqcCgYEAh459bgKmlYfa8LJ775E+
9+hn3/aCuS66Ujrsr4kF4OWum4SAeEInwqFV5bCEBFiWaHC71+dcYdzTkIWoLB2k
mUbXmQ957bdykwYmFSgAjKQyLkxmzIDwOXPfjJy7YCucc/CF5Rvc59Ni58FM1oex
VAT4M3K1ml/zEBxQZvAetp4=
-----END PRIVATE KEY-----
```



然后来看一下签名之后的CERT.SF文件的内容，可以用命令```openssl pkcs7 -inform DER -in CERT.RSA -noout -print_certs -text```来查看到其中包含的内容，是符合标准格式的  

```xml
Certificate:
    Data:
        Version: 3 (0x2)//版本号
        Serial Number: 11710352483038498175 (0xa283878a0a1f8d7f)//序列号
    Signature Algorithm: sha1WithRSAEncryption//签名算法
        Issuer: C=CN, ST=Zhejiang, L=Wu Chang, O=CETHIK, OU=Smart, CN=Simba/emailAddress=yuchunbo@hiksmart.com//发布者
        Validity
            Not Before: Nov  9 04:23:27 2015 GMT//生效日期
            Not After : Mar 27 04:23:27 2043 GMT//失效日期
        Subject: C=CN, ST=Zhejiang, L=Wu Chang, O=CETHIK, OU=Smart, CN=Simba/emailAddress=yuchunbo@hiksmart.com
        Subject Public Key Info://公钥信息
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:c4:99:b7:6d:1e:c3:85:ad:db:9b:bb:4c:6e:2d:
                    ab:35:84:28:d4:2d:1b:2c:ce:b6:8d:2f:f5:53:de:
                    50:b9:5d:f1:25:44:c9:6d:5c:ce:08:d6:17:2b:59:
                    94:e7:f7:fa:dc:a3:22:d0:17:e3:ca:20:c5:f0:86:
                    0c:e2:06:01:4f:ec:32:7b:05:46:98:02:aa:29:72:
                    6d:77:57:b8:2b:bb:1c:f3:3a:85:2b:41:56:3f:55:
                    a2:13:40:55:72:f5:47:d3:7c:c2:a8:39:07:61:54:
                    e0:40:a1:e8:a3:37:f8:a4:69:82:25:dd:ae:8c:72:
                    28:f0:12:6a:7e:82:9a:a3:ee:3b:b9:1d:a9:07:bf:
                    1d:e4:c1:84:a2:ad:05:1c:88:9c:c5:05:7b:25:87:
                    ab:20:71:33:af:72:8d:aa:f0:b1:4d:62:70:aa:fa:
                    e1:04:48:a0:9f:7a:8e:9a:33:91:2a:49:85:c8:ca:
                    2b:a1:e0:a4:9c:e2:a8:36:df:b5:7b:ab:72:79:e3:
                    1d:c7:67:a2:6a:5f:87:95:ea:63:fd:cd:d2:01:f2:
                    0b:13:11:56:ac:dc:9e:78:6b:c7:c0:4a:b1:07:df:
                    d0:97:b6:6c:15:6b:c7:ee:82:27:3b:32:f9:65:89:
                    94:67:53:81:86:d7:ba:3f:1e:f6:20:eb:30:9c:60:
                    53:bf
                Exponent: 65537 (0x10001)
        X509v3 extensions://额外信息
            X509v3 Subject Key Identifier: //使用者唯一标识
                94:1D:52:E5:8B:B9:DE:60:84:58:19:63:64:DE:5E:98:22:40:1E:17
            X509v3 Authority Key Identifier: //签发者使用标识
                keyid:94:1D:52:E5:8B:B9:DE:60:84:58:19:63:64:DE:5E:98:22:40:1E:17

            X509v3 Basic Constraints: //CA认证
                CA:TRUE
    Signature Algorithm: sha1WithRSAEncryption//签名算法，后面为加密过的摘要信息
         a1:70:ea:aa:93:1f:e2:7d:4a:b0:5c:6b:6d:2a:dc:c0:40:95:
         53:84:ed:94:ac:91:b0:5d:8a:7a:d8:b3:91:42:aa:af:af:61:
         35:81:8e:fd:12:a6:61:2a:3f:ce:2f:42:4f:04:2e:29:9c:9c:
         d7:56:7c:68:94:71:aa:d9:b1:c5:73:b6:9a:32:00:f1:e9:8c:
         57:21:ff:3c:fb:5c:cc:3e:05:6b:c3:0f:a2:cf:d2:d2:3f:bf:
         0a:e2:4e:e3:8e:ef:2b:de:18:04:25:d5:8a:28:49:75:10:28:
         77:99:82:47:f6:d5:53:82:29:d4:7d:22:40:42:c5:17:a9:9f:
         00:02:b4:2a:ed:1f:92:95:c4:a2:18:fc:7a:47:93:42:c1:02:
         d5:26:eb:7d:6f:39:a0:01:f0:ce:18:29:4a:32:84:cb:31:21:
         ad:e7:ff:e4:a1:f9:eb:ae:e8:25:53:00:f3:88:39:17:62:8b:
         84:67:84:cd:46:be:a3:05:54:56:67:db:59:e7:6d:ea:ae:e4:
         46:38:7b:f2:c7:ea:24:37:35:6b:35:f0:22:42:3f:dc:12:85:
         fb:d5:b6:ba:1f:9c:41:96:12:ee:b3:96:c0:2d:16:f5:c8:db:
         ba:09:67:af:44:68:fb:5d:de:67:2b:56:e0:e8:8f:85:77:32:
         6a:33:18:66
```

我们用十六进制编辑器来打开CERT.RSA（我这边用notpad++安装HEX-Editor来打开），可以看到中间包含了公钥和加密的信息。  

![PublicKey.png](https://chendongqi.github.io/blog/img/2018-12-07-system_upgrade/PublicKey.png)   

![EncryptedData.png](https://chendongqi.github.io/blog/img/2018-12-07-system_upgrade/EncryptedData.png)   

其实也就是说我们用命令查看到证书和加密信息，都是可以通过读取二进制文件的内容通过特定的offset来解析得到的，这个给用程序来获取CERT.SF提供了一个思路。  

那么我们再来谈论一件事情，Android系统在安装apk的时候，是如何进行对签名校验的呢？涉及到了PackageParser.java和JarVerifier.java简单来说分成如下几个步骤（对于系统签名来说）：  

1、读取DSA/RSA/EC后缀的签名证书文件（在/system/etc/下），然后调用verifyCertificate进行难，此处并无限制文件名，因此将CERT.RSA改成CERT123.RSA依然有效，但SF文件得跟RSA文件等签名证书文件同名；  

2、读取SF（与后面证书同名）与RSA/DSA/EC文件的内容，再调用verifySignature进行校验；

3、在verifySignature进行校验时会去读取各项证书信息，包括证书序列号、拥有者、加密算法等等，然后判断CERT.RSA证书对CERT.SF的文件签名是否正确，防止CERT.SF被篡改，若成功则返回证书链，否则抛出异常；

4、在返回证书链时，对于自签名证书，则直接作为合法证书返回；

5、继续回到verifyCertificate函数，它会校验SF文件中的MANIFEST.MF中各项Hash值是否正确，防止MF文件被篡改；

而对于非系统应用，也就说第三方应用来说，去枚举除META-INF目录以外的所有文件，然后进行哈希运算，并将其与MANIFEST.MF中的各文件哈希值进行比对，只有相匹配后才允许安装应用。也就是说只校验文件的完整性，不校验签名也就不校验apk的颁发者。  

那么我们继续站在攻击者的角度来思考下有了CERT.RSA之后，此apk或者升级包是否还能被攻破？按照攻击的思路，我们先修改MANIFEST.MF，然后修改CERT.SF，接下来就要用私钥对CERT.SF加密生成CERT.RSA了，因为我们无法拿到开发者的私钥，假设用自己的密钥对进行了签名。那么在系统端进行校验时，使用开发者的公钥对经过我们自己私钥加密的CERT.RSA进行解密，解出来的信息跟CERT.SF中的肯定不同，所以校验失败，这么来说确实是保护了文件不被篡改。那么在拿不到开发者秘钥对的前提下，我能想到的一个方式就是在校验阶段用自己的公钥去解密CERT.RSA了，再CERT.RSA里本来就包含了公钥信息，那么如何让校验程序能使用到我们的公钥呢？替换系统中的公钥？system分区是只读的，先root？做坏事还真的比较麻烦的一件事，不过挺有意思。  

前面提到了第三方程序的签名校验是不严格的，这里延伸出一个问题，那么这个签名信息还能用来做什么？能想到的一点时，我们如何去判断一个第三方apk是不是官方发布的，或者说在现在有那么多应用市场的情况下，我们如何来判断apk的来源。那么就可以利用到RSA里包含的公钥信息了。我们可以将公钥信息和官方发布的公钥或是应用市场的公钥做比对来确定来源。  

升级包的签名校验过程会在后面章节中专门介绍。  

#### signapk分析

前面通过手动的方式确认了MANIFEST.MF和CERT.SF是如何生成的，那么我们看到的文件是签名工具（Android这里用的是Signapk），那么我们来从代码看下SignApk的签名过程来印证下。代码位于源码下build/tools/signapk/SignApk.java中。  

```java
	public static void main(String[] args) {
    	...
        //参数带-w，全局签名置为true
        if (args[0].equals("-w")) {
            signWholeFile = true;
            argstart = 1;
        }
    	...
		if (signWholeFile) {
            // 全局签名，后面代码跳转到CMSSigner
            SignApk.signWholeFile(inputJar, firstPublicKeyFile,
                                  publicKey[0], privateKey[0], outputFile);
        } else {
            JarOutputStream outputJar = new JarOutputStream(outputFile);

            // For signing .apks, use the maximum compression to make
            // them as small as possible (since they live forever on
            // the system partition).  For OTA packages, use the
            // default compression level, which is much much faster
            // and produces output that is only a tiny bit larger
            // (~0.1% on full OTA packages I tested).
            outputJar.setLevel(9);//压缩等级，可以自行测试下时间和空间的关系
			// 正常的签名
            // 1.生成MANIFEST.MF文件
            Manifest manifest = addDigestsToManifest(inputJar, hashes);
            // 2.生成新的压缩包，并将文件拷贝进去
            copyFiles(manifest, inputJar, outputJar, timestamp);
            // 3.对新包进行签名
            signFile(manifest, inputJar, publicKey, privateKey, outputJar);
            outputJar.close();
        }
	}
```

```java
	/**
     * Add the hash(es) of every file to the manifest, creating it if
     * necessary.
     */
	// 生成全局文件摘要到MANIFEST.MF文件
    private static Manifest addDigestsToManifest(JarFile jar, int hashes)
        throws IOException, GeneralSecurityException {
        Manifest input = jar.getManifest();
        Manifest output = new Manifest();
        Attributes main = output.getMainAttributes();
        if (input != null) {
            main.putAll(input.getMainAttributes());
        } else {
            main.putValue("Manifest-Version", "1.0");
            main.putValue("Created-By", "1.0 (Android SignApk)");
        }

        MessageDigest md_sha1 = null;
        MessageDigest md_sha256 = null;
        if ((hashes & USE_SHA1) != 0) {
            md_sha1 = MessageDigest.getInstance("SHA1");
        }
        if ((hashes & USE_SHA256) != 0) {
            md_sha256 = MessageDigest.getInstance("SHA256");
        }

        byte[] buffer = new byte[4096];
        int num;

        // We sort the input entries by name, and add them to the
        // output manifest in sorted order.  We expect that the output
        // map will be deterministic.

        TreeMap<String, JarEntry> byName = new TreeMap<String, JarEntry>();

        for (Enumeration<JarEntry> e = jar.entries(); e.hasMoreElements(); ) {
            JarEntry entry = e.nextElement();
            byName.put(entry.getName(), entry);
        }

        for (JarEntry entry: byName.values()) {
            String name = entry.getName();
            if (!entry.isDirectory() &&
                (stripPattern == null || !stripPattern.matcher(name).matches())) {
                InputStream data = jar.getInputStream(entry);
                while ((num = data.read(buffer)) > 0) {
                    if (md_sha1 != null) md_sha1.update(buffer, 0, num);
                    if (md_sha256 != null) md_sha256.update(buffer, 0, num);
                }

                Attributes attr = null;
                if (input != null) attr = input.getAttributes(name);
                attr = attr != null ? new Attributes(attr) : new Attributes();
                if (md_sha1 != null) {
                    attr.putValue("SHA1-Digest",
                                  new String(Base64.encode(md_sha1.digest()), "ASCII"));
                }
                if (md_sha256 != null) {
                    attr.putValue("SHA-256-Digest",
                                  new String(Base64.encode(md_sha256.digest()), "ASCII"));
                }
                output.getEntries().put(name, attr);
            }
        }

        return output;
    }
```

```java
	// 签名文件
	private static void signFile(Manifest manifest, JarFile inputJar,
                                 X509Certificate[] publicKey, PrivateKey[] privateKey,
                                 JarOutputStream outputJar)
        throws Exception {
        // Assume the certificate is valid for at least an hour.
        long timestamp = publicKey[0].getNotBefore().getTime() + 3600L * 1000;

        // MANIFEST.MF
        JarEntry je = new JarEntry(JarFile.MANIFEST_NAME);
        je.setTime(timestamp);
        outputJar.putNextEntry(je);
        manifest.write(outputJar);

        int numKeys = publicKey.length;
        for (int k = 0; k < numKeys; ++k) {
            // CERT.SF / CERT#.SF
            je = new JarEntry(numKeys == 1 ? CERT_SF_NAME :
                              (String.format(CERT_SF_MULTI_NAME, k)));
            je.setTime(timestamp);
            outputJar.putNextEntry(je);
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            // 生成CERT.SF
            writeSignatureFile(manifest, baos, getAlgorithm(publicKey[k]));
            byte[] signedData = baos.toByteArray();
            outputJar.write(signedData);

            // CERT.RSA / CERT#.RSA
            je = new JarEntry(numKeys == 1 ? CERT_RSA_NAME :
                              (String.format(CERT_RSA_MULTI_NAME, k)));
            je.setTime(timestamp);
            outputJar.putNextEntry(je);
            // 生成CERT.RSA
            writeSignatureBlock(new CMSProcessableByteArray(signedData),
                                publicKey[k], privateKey[k], outputJar);
        }
    }
```

```java
	/** Write a .SF file with a digest of the specified manifest. */
	// 生成CERT.SF文件
    private static void writeSignatureFile(Manifest manifest, OutputStream out,
                                           int hash)
        throws IOException, GeneralSecurityException {
        Manifest sf = new Manifest();
        Attributes main = sf.getMainAttributes();
        main.putValue("Signature-Version", "1.0");
        main.putValue("Created-By", "1.0 (Android SignApk)");

        MessageDigest md = MessageDigest.getInstance(
            hash == USE_SHA256 ? "SHA256" : "SHA1");
        PrintStream print = new PrintStream(
            new DigestOutputStream(new ByteArrayOutputStream(), md),
            true, "UTF-8");

        // Digest of the entire manifest
        manifest.write(print);
        print.flush();
        main.putValue(hash == USE_SHA256 ? "SHA-256-Digest-Manifest" : "SHA1-Digest-Manifest",
                      new String(Base64.encode(md.digest()), "ASCII"));

        Map<String, Attributes> entries = manifest.getEntries();
        for (Map.Entry<String, Attributes> entry : entries.entrySet()) {
            // Digest of the manifest stanza for this entry.
            print.print("Name: " + entry.getKey() + "\r\n");
            for (Map.Entry<Object, Object> att : entry.getValue().entrySet()) {
                print.print(att.getKey() + ": " + att.getValue() + "\r\n");
            }
            print.print("\r\n");
            print.flush();

            Attributes sfAttr = new Attributes();
            sfAttr.putValue(hash == USE_SHA256 ? "SHA-256-Digest" : "SHA1-Digest-Manifest",
                            new String(Base64.encode(md.digest()), "ASCII"));
            sf.getEntries().put(entry.getKey(), sfAttr);
        }

        CountOutputStream cout = new CountOutputStream(out);
        sf.write(cout);

        // A bug in the java.util.jar implementation of Android platforms
        // up to version 1.6 will cause a spurious IOException to be thrown
        // if the length of the signature file is a multiple of 1024 bytes.
        // As a workaround, add an extra CRLF in this case.
        if ((cout.size() % 1024) == 0) {
            cout.write('\r');
            cout.write('\n');
        }
    }
```

```java
	private static class CMSSigner implements CMSTypedData {
        private JarFile inputJar;
        private File publicKeyFile;
        private X509Certificate publicKey;
        private PrivateKey privateKey;
        private String outputFile;
        private OutputStream outputStream;
        private final ASN1ObjectIdentifier type;
        private WholeFileSignerOutputStream signer;

        public CMSSigner(JarFile inputJar, File publicKeyFile,
                         X509Certificate publicKey, PrivateKey privateKey,
                         OutputStream outputStream) {
            this.inputJar = inputJar;
            this.publicKeyFile = publicKeyFile;
            this.publicKey = publicKey;
            this.privateKey = privateKey;
            this.outputStream = outputStream;
            this.type = new ASN1ObjectIdentifier(CMSObjectIdentifiers.data.getId());
        }

        public Object getContent() {
            throw new UnsupportedOperationException();
        }

        public ASN1ObjectIdentifier getContentType() {
            return type;
        }

       	// 写新包
        public void write(OutputStream out) throws IOException {
            try {
                signer = new WholeFileSignerOutputStream(out, outputStream);
                JarOutputStream outputJar = new JarOutputStream(signer);

                int hash = getAlgorithm(publicKey);

                // Assume the certificate is valid for at least an hour.
                long timestamp = publicKey.getNotBefore().getTime() + 3600L * 1000;

                Manifest manifest = addDigestsToManifest(inputJar, hash);
                copyFiles(manifest, inputJar, outputJar, timestamp);
                // -w的方式会添加otacert
                addOtacert(outputJar, publicKeyFile, timestamp, manifest, hash);

                signFile(manifest, inputJar,
                         new X509Certificate[]{ publicKey },
                         new PrivateKey[]{ privateKey },
                         outputJar);

                signer.notifyClosing();
                outputJar.close();
                signer.finish();
            }
            catch (Exception e) {
                throw new IOException(e);
            }
        }

        public void writeSignatureBlock(ByteArrayOutputStream temp)
            throws IOException,
                   CertificateEncodingException,
                   OperatorCreationException,
                   CMSException {
            SignApk.writeSignatureBlock(this, publicKey, privateKey, temp);
        }

        public WholeFileSignerOutputStream getSigner() {
            return signer;
        }
    }
```

从上面的代码中可以看出Signapk签名的步骤就是从新的压缩包里生成摘要文件MANIFEST.MF，然后根据生成CERT.SF，再根据秘钥对生成CERT.RSA，将这些写入到新的压缩包里，如果带-w参数，则拷贝otacert公钥。  

### 签名算法

#### 对称加密和不对称加密

在对称加密算法中，数据发信方将明文和加密秘钥一起经过加密算法处理后，变成密文发送出去。
收信方收到密文后，需要使用加密用过的密钥及相同算法的逆算法对密文进行解密，恢复成可读明文。在对称加密算法中，使用的密钥只有一个。  
1976年，两位美国计算机学家Whitfield Diffie 和 Martin Hellman，提出了一种崭新构思，可以在不直接传递密钥的情况下，完成解密。
这被称为“Diffie-Hellman密钥交换算法”。这个算法启发了其他科学家。人们认识到，加密和解密可以使用不同的规则，只要这两种规则之间存在某种对应关系即可，这样就避免了直接传递密钥。
这种新的加密模式被称为“非对称加密算法”  
#### RSA算法原理
##### 数论基础
###### 互质关系
如果两个正整数，除了1以外，没有其他公因子，我们就称这两个数是互质关系（coprime）。比如，15和32没有公因子，所以它们是互质关系。这说明，不是质数也可以构成互质关系。
关于互质关系，不难得到以下结论：
> 1. 任意两个质数构成互质关系，比如13和61。  
> 
> 2. 一个数是质数，另一个数只要不是前者的倍数，两者就构成互质关系，比如3和10。  
> 
> 3. 如果两个数之中，较大的那个数是质数，则两者构成互质关系，比如97和57。  
> 
> 4. 1和任意一个自然数是都是互质关系，比如1和99。  
> 
> 5. p是大于1的整数，则p和p-1构成互质关系，比如57和56。  
> 
> 6. p是大于1的奇数，则p和p-2构成互质关系，比如17和15。  

###### 欧拉函数
请思考以下问题：
> 任意给定正整数n，请问在小于等于n的正整数之中，有多少个与n构成互质关系？（比如，在1到8之中，有多少个数与8构成互质关系？）  

计算这个值的方法就叫做欧拉函数，以φ(n)表示。在1到8之中，与8形成互质关系的是1、3、5、7，所以 φ(n) = 4。
> 如果n=1，则 φ(1) = 1 。因为1与任何数（包括自身）都构成互质关系。
> 
> 如果n是质数，则 φ(n)=n-1 。因为质数与小于它的每一个数，都构成互质关系。比如5与1、2、3、4都构成互质关系。  
> 
> 如果n是质数的某一个次方，即 n = p^k (p为质数，k为大于等于1的整数)，则φ(n)=p^k-p^(k-1)=(p-1)*p^(k-1)=p^k*(1-1/p)  
> 
> 如果n可以分解成两个互质的整数之积,则φ(n) = φ(p1p2) = φ(p1)φ(p2)。特殊的，当m=2，n为奇数时，φ(2*n)=φ(n)  

###### 欧拉定理
欧拉函数的用处，在于欧拉定理。"欧拉定理"指的是：
> 如果两个正整数a和n互质，则n的欧拉函数 φ(n) 可以让下面的等式成立：
> 
> a^ φ(n) ≡1(mod n)  

###### 模反元素
还剩下最后一个概念：如果两个正整数a和n互质，那么一定可以找到整数b，使得 ab-1 被n整除，或者说ab被n除的余数是1。这时，b就叫做a的"模反元素"。  
比如，3和11互质，那么3的模反元素就是4，因为 (3 × 4)-1 可以被11整除。显然，模反元素不止一个， 4加减11的整数倍都是3的模反元素 {...,-18,-7,4,15,26,...}，即如果b是a的模反元素，则 b+kn 都是a的模反元素。  
##### 生成秘钥的流程

| 序号   | 步骤                                                 | 实例                                                         |
| ------ | ---------------------------------------------------- | ------------------------------------------------------------ |
| 第一步 | 选择两个不相等的质数p和q                             | p = 61； q = 53                                              |
| 第二步 | 计算p和q的乘积n                                      | n = 61*53=3233                                               |
| 第三步 | 计算n的欧拉函数φ(n)                                  | φ(n) = (p-1)(q-1) = 3120                                     |
| 第四步 | 随机选择一个整数e，条件是1< e < φ(n)，且e与φ(n) 互质 | 随机选择e = 17                                               |
| 第五步 | 计算e对于φ(n)的模反元素d                             | •ed ≡ 1 (mod φ(n))<br />•ed - 1 = kφ(n)<br />•ex + φ(n)y = 1   <br />•17x + 3120y = 1（y<n      <br />•一组整数解为(x,y)=(2753,-15)，即d=2753   <br /> |
| 第六步 | 将n和e封装成公钥，n和d封装成私钥                     | •公钥私钥（3233,17），私钥（3233,2753）                      |



##### 秘钥的安全性
回顾上面的密钥生成步骤，一共出现六个数字：p、q、n、φ(n)、e和d。这六个数字之中，公钥用到了两个（n和e），其余四个数字都是不公开的。其中最关键的是d，因为n和d组成了私钥，一旦d泄漏，就等于私钥泄漏。那么，有无可能在已知n和e的情况下，推导出d？
> （1）ed≡1 (mod φ(n))。只有知道e和φ(n)，才能算出d。  
> 
> （2）φ(n)=(p-1)(q-1)。只有知道p和q，才能算出φ(n)。  
> 
> （3）n=pq。只有将n因数分解，才能算出p和q。  

如果n可以被因数分解，d就可以算出，也就意味着私钥被破解。可是，大整数的因数分解，是一件非常困难的事情。目前，除了暴力破解，还没有发现别的有效方法。维基百科这样写道：
> "对极大整数做因数分解的难度决定了RSA算法的可靠性。换言之，对一极大整数做因数分解愈困难，RSA算法愈可靠。
> 
> 假如有人找到一种快速因数分解的算法，那么RSA的可靠性就会极度下降。但找到这样的算法的可能性是非常小的。今天只有短的RSA密钥才可能被暴力破解。到2008年为止，世界上还没有任何可靠的攻击RSA算法的方式。只要密钥长度足够长，用RSA加密的信息实际上是不能被解破的。"
> 

举例来说，你可以对3233进行因数分解（61×53），但是你没法对下面这个整数进行因数分解
> 1230186684530117755130494958384962720772853569595334792197322452151726400507263657518745202199786469389956474942774063845925192557326303453731548268507917026122142913461670429214311602221240479274737794080665351419597459856902143413   

它等于这样两个质数的乘积：  
> 33478071698956898786044169848212690817704794983713768568912431388982883793878002287614711652531743087737814467999489*36746043666799590428244633799627952632279158164343087642676032283815739666511279233373417143396810270092798736308917

事实上，这大概是人类已经分解的最大整数（232个十进制位，768个二进制位）。比它更大的因数分解，还没有被报道过，因此目前被破解的最长RSA密钥就是768位。  

##### 加密算法和解密算法  
有了公钥和密钥，就能进行加密和解密了。假设鲍勃要向爱丽丝发送加密信息m，他就要用爱丽丝的公钥 (n,e) 对m进行加密。这里需要注意，m必须是整数（字符串可以取ascii值或unicode值），且m必须小于n。  
所谓"加密"，就是算出下式的c：
> me ≡ c (mod n)  

爱丽丝的公钥是 (3233, 17)，鲍勃的m假设是65，那么可以算出下面的等式：
> 6517 ≡ 2790 (mod 3233)  

于是，c等于2790，鲍勃就把2790发给了爱丽丝。  
爱丽丝拿到鲍勃发来的2790以后，就用自己的私钥(3233, 2753) 进行解密。可以证明，下面的等式一定成立：  
> cd ≡ m (mod n)  

也就是说，c的d次方除以n的余数为m。现在，c等于2790，私钥是(3233, 2753)，那么，爱丽丝算出  
> 27902753 ≡ 65 (mod 3233)  

因此，爱丽丝知道了鲍勃加密前的原文就是65。  
至此，"加密--解密"的整个过程全部完成。  
我们可以看到，如果不知道d，就没有办法从c求出m。而前面已经说过，要知道d就必须分解n，这是极难做到的，所以RSA算法保证了通信安全。  
你可能会问，公钥(n,e) 只能加密小于n的整数m，那么如果要加密大于n的整数，该怎么办？有两种解决方法：一种是把长信息分割成若干段短消息，每段分别加密；另一种是先选择一种"对称性加密算法"（比如DES），用这种算法的密钥加密信息，再用RSA公钥加密DES密钥。

##### 加解密算法实现的证明  
![zhengming.png](https://chendongqi.github.io/blog/img/2019-06-03-sign_system/zhengming.png)

### 签名校验  
以apk安装校验为例分析  
#### 校验的思路  
1. 通过RSA文件解出SF文件的摘要，来和实际的SF文件比较  

2. 校验MF文件中的每项的hash值是否和SF中的匹配  

3. apk中的每项文件的摘要和MF中的记录比较  

4. RSA中的公钥和系统存储的可信秘钥比较  

#### apk安装校验流程
![sign_verify.png](https://chendongqi.github.io/blog/img/2019-06-03-sign_system/sign_verify.png)