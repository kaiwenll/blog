---
layout:      post
title:      "系统升级系列二"
subtitle:   "系统升级包介绍"
navcolor:   "invert"
date:       2018-12-17
author:     "Cheson"
#header-img: "/img/website/home-bg.jpg"
catalog: true
tags:
    - Android
    - System Upgrade
    - OTA
    - update.zip
---
### 概述

系列一中介绍了系统升级的概念，升级方式等，而系统升级的原料离不开升级包。巧妇难为无米之炊，那么我们先来看下这个“米”是什么样的“米”，才能更好的理解它是怎么做出的“饭”。  

此篇中会介绍的内容包括：升级包结构的剖析、升级包制作过程、制作相关的工具以及厂商定制的内容和一个非常规但是特别有用的技巧。  

### 升级包结构剖析

全量包和增量包的内容结构稍有差别，这里以某厂商的某项目生成的升级包为例

#### 全量包

全量包，顾名思义，就是包含了完整系统的升级包，将全量包里的内容分几个大类详见如下  

##### 系统镜像  

├── file_contexts  
├── fsl_app.s19  
├── logo.bin  
├── recovery.img  
├── system  
├── u-boot.bin  
├── uImage  
├── upgrade.dat  
└── uramdisk.img  

如上所示，为系统镜像相关部分，uramdisk.img、u-boot.bin、logo.bin、uImage、recovery.img这些是直接被写入到对应的分区中的。升级包中需要升哪些分区这个也是视不同厂商而不同，需要在生成升级包和升级包升级脚本中做对应匹配；system是一个目录，包含了所有编译产生的所有system下的内容，在这个版本中升级时将这里的system目录内容全部解压到系统的system分区做覆盖，完成对system分区的升级；file_contexts是对节点和可执行文件的权限配置；upgrade.dat和fsl_app.s19是控制升级mcu，其中.s19是mcu的软件，upgrade.dat是mcu升级程序，手机中是不会有mcu这部分的。  

##### 签名

├── META-INF  
│   ├── CERT.RSA  
│   ├── CERT.SF  
│   ├── com  
│   │   ├── android  
│   │   │   ├── metadata  
│   │   │   └── otacert  
│   └── MANIFEST.MF  

签名是一套庞大的知识体系，这里只对升级包里涉及到的部分以及自己探究的相关衍生做介绍。META-INF在升级包或是apk解开后都能看到，这个是目录是经过签名工具（可以是signapk）签名产生的，可以做一把试验，把升级包解开，然后把META-INF目录删除之后再打包，然后做签名，签名后会发现这个目录又出现了。这个过程涉及到Signapk的工作原理，后面会通过源码来提及一些签名原理。这篇中我们只简单介绍下升级包里文件的用途，签名系统的详细介绍会有专门系列来介绍，来看META-INF下的三个紧密相关的文件CERT.RSA、CERT.SF和MANIFEST.MF。  

| File        | Description                                                  |
| ----------- | ------------------------------------------------------------ |
| MANIFEST.MF | 保存除META-INF根目录下文件以外其它各文件的SHA-1+base64编码后的值 |
| CERT.SF     | 在SHA1-Digest-Manifest中保存MANIFEST.MF根目录下文件的SHA-1+base64编码后的值，在后面的各项SHA1-Digest中保存MANIFEST.MF各子项内容SHA-1+Base64编码后的值 |
| CERT.RSA    | 保存用私钥计算出CERT.SF文件的数字签名、证书发布机构、有效期、公钥、所有者、签名算法等信息 |



##### metadata

此文件是在OTA生成脚本里写入到升级包中，包含了一些项目信息的数据，可用于升级包的校验    

```py
def WriteFullOTAPackage(input_zip, output_zip):
  # TODO: how to determine this?  We don't know what version it will
  # be installed on top of.  For now, we expect the API just won't
  # change very often.
  script = edify_generator.EdifyGenerator(3, OPTIONS.info_dict)

  metadata = {"post-build": GetBuildProp("ro.build.fingerprint",
                                         OPTIONS.info_dict),
              "pre-device": GetBuildProp("ro.product.device",
                                         OPTIONS.info_dict),
              "post-timestamp": GetBuildProp("ro.build.date.utc",
                                             OPTIONS.info_dict),
              }
  ...
  metadata["pre-build"] = source_fp//从source包中取到基础版本fingerprint
  metadata["post-build"] = target_fp//从target包中取到目标版本fingerprint
```

而生成的metadata内容如下  

```xml
post-build=Freescale/xxxx/xxxx:4.4.2/XXXX_Gen1.0/175:user/dev-keys
post-timestamp=1543983748
pre-build=Freescale/xxxx/xxxx:4.4.2/XXXX_Gen1.0/173:user/dev-keys
pre-device=xxxx
```

此文件我理解的是用来校验升级包的，包括了校验当前设备是否跟pre-device型号一致，验证时间戳（时间戳的作用后面会专门介绍到），验证两个版本的fingerprint是否符合升级规则。  

这里预告下，后面小节中会介绍到Manifest_SYS.xml文件，这里也就是包含了项目的一些信息，在升级中做校验用的，我觉得跟metadata的功能重复了，而且metadata里也可以加入自己需要定制的校验信息，应该是此厂商定制OTA校验时评估的失误。  

##### Manifest_SYS.xml

此文件是厂商定制用来做升级时的校验，可以看到内容如下  

```xml
<?xml version="1.0" encoding="utf-8"?>
<sys_info>
<type>sys</type>//自定义类型
<project>xxxx</project>//项目名，用于升级校验
<minSdkVersion>3</minSdkVersion>//最小sdk版本，不用做校验
<minCustomSdkVersion>1</minCustomSdkVersion>//custom sdk版本，不用做校验
<vendor>xxxx</vendor>//厂商，不用做校验
<fileName>xxxx-update-ota-2.4.9-B4-user.zip</fileName>//升级包文件名，不用做校验
<systemVersionName>2.4.9</systemVersionName>//目标版本名
<systemVersionCode>180</systemVersionCode>//目标版本code
<isfull>1</isfull>//全量包
<dependencyVersionCode></dependencyVersionCode>//基础版本，全量包不依赖
<systemVersionDescription></systemVersionDescription>//系统版本描述
<ext></ext>
</sys_info>
```

##### update-binary&updater-script

原生的系统是两个文件，update-binary（升级执行程序）和updater-script（升级控制脚本），这里是厂商定制，因为有双系统，所以有两个升级控制脚本来做A/B系统的不同升级。  

├── META-INF  
│   ├── com  
│   │   └── google  
│   │       └── android  
│   │           ├── update-binary  
│   │           ├── updater-script  
│   │           └── updater-scriptB  

update-binary是去解析updater-script脚本中的内容，然后去做执行工作，所以要先来了解下updater-script脚本。这部分会在后面的系列中详细展开分析。  

#### 增量包

增量包和全量包的结构非常类似，形式相同的地方以下就一笔带过，会重点说明下差异的地方  

##### 系统镜像

├── data  
│   └── update_s  
│       └── system  
│           └── media  
│               └── audio  
│                   └── ringtones  
│                       ├── Acheron.ogg  
│                       ├── FreeFlight.wav  
│                       ├── Kuma.ogg  
│                       ├── Sceptrum.wav  
│                       ├── Sedna.ogg  
│                       └── Themos.ogg  
├── fsl_app.s19  
├── logo.bin  
├── patch  
│   └── data  
│       └── update_s  
│           └── system  
│               ├── app  
│               │   ├── aMapAuto.odex.p  
├── recovery.img  
├── u-boot.bin  
├── uImage  
├── upgrade.dat  
└── uramdisk.img  

这部分除system分区外跟全量包类似，另外说明的一点时，在全量升级或是增量升级时，哪些分区需要升级是可以定制的，例如我们在OTA升级中不对Recovery进行升级，在OTA包制作时将其去掉，updater-script的生成中也要对应的去除Recovery分区写入的命令。  

而增量包和全量包最大的差别点就是就是在system分区，全量包里这部分是完整的system分区的内容，而增量包里是patch目录下包含了data/update_s/system，然后下面包含的是system分区下对应文件两个版本之间的差异（.p文件），升级时将此patch打到旧系统的文件中，就生成了一个新文件。  

这里为什么system分区的路径是data/update_s/system？因为在升级时，会将system分区挂载到/data/update_s目录下，这个是在updater-script中控制的，当然可以做定制。  

另外升级包目录下多出来一个data目录，跟上面一样，这里也是system分区的东西，patch包含的是补丁，而这里包含的就是整个文件，那些无法做查分的文件（包括做patch大小超过原始文件的，新增的文件）会直接拷贝到这里，升级时直接拷贝到挂载的system分区下。  

##### 签名

跟全量包一致

##### metadata

跟全量包一致

##### Manifest_SYS.xml

部分字段跟全量包有差异  

```xml
<?xml version="1.0" encoding="utf-8"?>
<sys_info>
<type>sys</type>
<project>xxxx</project>//项目名，做校验用
<vendor>xxxx</vendor>
<isfull>0</isfull>//增量包
<systemVersionName>2.4.5</systemVersionName>//目标版本name
<systemVersionCode>175</systemVersionCode>//目标版本code
<dependencyVersionName>2.4.4</dependencyVersionName>//基础版本name
<dependencyVersionCode>173</dependencyVersionCode>//基础版本name，做校验用
<systemVersionDescription></systemVersionDescription>
<ext></ext>
</sys_info>
```

##### update-binary&updater-script

跟全量包一致，updater-script里的具体内容当然会有差别  