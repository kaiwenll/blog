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

签名是一套庞大的知识体系，这里只对升级包里涉及到的部分以及自己探究的相关衍生做介绍。META-INF在升级包或是apk解开后都能看到，这个是目录是经过签名工具（可以是signapk）签名产生的，可以做一把试验，把升级包解开，然后把META-INF目录删除之后再打包，然后做签名，签名后会发现这个目录又出现了。这个过程涉及到Signapk的工作原理，后面会通过源码来提及一些签名原理。这篇中我们只简单介绍下升级包里文件的用途，签名系统的详细介绍参考[系统升级系列三—签名](http://chendongqi.me/2018/12/20/SignSystem/)，来看META-INF下的三个紧密相关的文件CERT.RSA、CERT.SF和MANIFEST.MF。  

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



### 升级包制作

这一部分会介绍升级包制作的方法以及升级包生成的原理  

#### 全量包

首先来看整包制作，需准备的环境是已经编译过的源码环境，并执行了source和lunch。直接执行make otapackage即可产生中间的target file和OTA的全量包。因为从代码看的顺序是一段段逆推回去的，我这里也用这种逆向的思路来介绍Makefile里的脚本逻辑。  

##### INTERNAL_OTA_PACKAGE_TARGET

执行make otapackage会走到build/core/Makefile里编译OTA的一段  

```makefile
# -----------------------------------------------------------------
# OTA update package
### 3、name的拼接规则
name := $(TARGET_PRODUCT)-update-ota-$(FILE_NAME_TAG)
ifeq ($(TARGET_BUILD_TYPE),debug)
  name := $(name)_debug
endif

### 2、INTERNAL_OTA_PACKAGE_TARGET拼接规则
INTERNAL_OTA_PACKAGE_TARGET := $(PRODUCT_OUT)/$(name).zip
### 5、INTERNAL_OTA_PACKAGE_TARGET_TEMP的由来
INTERNAL_OTA_PACKAGE_TARGET_TEMP := $(PRODUCT_OUT)/$(name)_nomd5.zip
### 6、厂商定制的更多依赖
$(INTERNAL_OTA_PACKAGE_TARGET): EXTRA_SCRIPT := $(strip $(TARGET_DEVICE_DIR)/ota_updater-script)
$(INTERNAL_OTA_PACKAGE_TARGET): EXTRA_SCRIPTB := $(strip $(TARGET_DEVICE_DIR)/ota_updater-scriptB)
$(INTERNAL_OTA_PACKAGE_TARGET): EXTRA_SCRIPTMCU := $(strip $(TARGET_DEVICE_DIR)/mcu-updater-script)
ifeq ($(UPDATE_MCU_IN_OTA),No)
  $(INTERNAL_OTA_PACKAGE_TARGET): EXTRA_SCRIPTMCU := disable
endif
### 7、密钥对
$(INTERNAL_OTA_PACKAGE_TARGET): KEY_CERT_PAIR := $(DEFAULT_KEY_CERT_PAIR)
### 8、INTERNAL_OTA_PACKAGE_TARGET核心编译依赖
$(INTERNAL_OTA_PACKAGE_TARGET): $(BUILT_TARGET_FILES_PACKAGE) $(DISTTOOLS)
	@echo -e ${CL_YLW}"Package OTA:"${CL_RST}" $@"
	@echo "Server side manifest..."
	$(hide) ./build/tools/releasetools/create_server_manifest \
			-p $(TARGET_DEVICE) \
			-t sys \
			-i $(INTERNAL_OTA_PACKAGE_TARGET) \
			-o $(PRODUCT_OUT)/Manifest_SYS.xml
	@echo "Create ota package."
	$(hide) ./build/tools/releasetools/ota_from_target_files -v -w\
	   -p $(HOST_OUT) \
	   -k $(KEY_CERT_PAIR) \
	   -t $(PRODUCT_OUT)/Manifest_SYS.xml \
	   -e $(EXTRA_SCRIPT) \
	   -f $(EXTRA_SCRIPTB) \
	   -m $(EXTRA_SCRIPTMCU) \
	   $(BUILT_TARGET_FILES_PACKAGE) $(INTERNAL_OTA_PACKAGE_TARGET)
	@echo "Create ota package md5."
	$(hide) ./build/tools/releasetools/create_md5_file \
			-i $(INTERNAL_OTA_PACKAGE_TARGET) \
			-p $(TARGET_DEVICE) \
			-o $(PRODUCT_OUT)/ota_md5.txt
	$(hide) $(MKFILE)  --file $(INTERNAL_OTA_PACKAGE_TARGET) --suffix $(PRODUCT_OUT)/ota_md5.txt --suffix_size 196 --output $@
	$(hide) rm -rf $(PRODUCT_OUT)/Manifest_SYS.xml
	$(hide) rm -rf $(PRODUCT_OUT)/ota_md5.txt
	@echo "ota update zip package done..."
### 1、otapackage 伪命令,即执行 make otapackage时,将编译$(INTERNAL_OTA_PACKAGE_TARGET)目标
.PHONY: otapackage
otapackage: $(INTERNAL_OTA_PACKAGE_TARGET)
```

1、执行make otapackage，会去编译INTERNAL_OTA_PACKAGE_TARGET目标，那么INTERNAL_OTA_PACKAGE_TARGET是什么？  

2、向上找到INTERNAL_OTA_PACKAGE_TARGET的拼接规则，其中PRODUCT_OUT就是out/target/product/\<product_name\>，name的产生再看下一步  

3、TARGET_PRODUCT就是项目的product_name，FILE_NAME_TAG的生成在Makefile的最开始处，见如下代码  

```makefile
### 4、FILE_NAME_TAG规则
ifeq (,$(RELEASE_VERSION))
    FILE_NAME_TAG := "?.?.?"
else
    FILE_NAME_TAG := $(subst $(space),_,$(RELEASE_VERSION))
endif

ifeq (,$(BUILD_NUMBER_HEX))
    FILE_NAME_TAG := $(FILE_NAME_TAG)-"?"
else
    FILE_NAME_TAG := $(FILE_NAME_TAG)-$(BUILD_NUMBER_HEX)
endif

FILE_NAME_TAG := $(FILE_NAME_TAG)-$(TARGET_BUILD_VARIANT)
```

4、FILE_NAME_TAG生成：整个字串的格式是“RELEASE_VERSION-BUILD_NUMBER_HEX-TARGET_BUILD_VARIANT”。先判断RELEASE_VERSION是否配置，如果未配置，则用“?.?.?”来占位，如果配置了，则用RELEASE_VERSION信息来占位（subst方法将RELEASE_VERSION中的空格用'_'替换）；再判断BUILD_NUMBER_HEX是否配置，注意这里是用十六进制的，如果未配置，则用“?”来占位，否则用BUILD_NUMBER_HEX。最后再拼接上TARGET_BUILD_VARIANT。例如，一个RELEASE_VERSION、BUILD_NUMBER_HEX都配置了的user版本就是“2.5.0-B6-user”，一个什么都没配eng版本就是“?.?.?-?-eng”。所以INTERNAL_OTA_PACKAGE_TARGET的完整值就是out/target/product/\<product_name\>/xe110bm-update-ota-2.5.0-B6-user.zip，也就是生成升级包最终的目录和名称。  

5、这个INTERNAL_OTA_PACKAGE_TARGET_TEMP现在没用，这个产生的历史原因如下：厂商定制时对update package文件信息里写入了md5值，安装时从文件信息里读出md5(类似读签名的方式，用offset控制)然后去校验update包是否完整，而google原始的是不会写md5的，所以这边多出一个nomd5的包应该是保留跟原生相同的部分，但是后面确没有真的去实现可选md5的这种方式。  

6、这里比google原生的多出来的一部分，是厂商定制的一部分依赖，对照前面升级包结构剖析可以明白这个定制的上下文实现，因为针对A/B系统的升级，有两个不同的升级脚本，所以这里编译升级包时需要加入两个不同的升级脚本，TARGET_DEVICE_DIR就是源码下的device/fsl/\<product_name\>，所以EXTRA_SCRIPT就是目录下的ota_updater-script，EXTRA_SCRIPTB就是ota_updater-scriptB脚本；另外需要mcu-updater-script脚本用于mcu的升级定制。  

7、秘钥对的赋值：  ```DEFAULT_KEY_CERT_PAIR := $(DEFAULT_SYSTEM_DEV_CERTIFICATE)```，而DEFAULT_SYSTEM_DEV_CERTIFICATE是在config.mk里赋值的  

```makefile
# The default key if not set as LOCAL_CERTIFICATE
ifdef PRODUCT_DEFAULT_DEV_CERTIFICATE
  DEFAULT_SYSTEM_DEV_CERTIFICATE := $(PRODUCT_DEFAULT_DEV_CERTIFICATE)
else
  DEFAULT_SYSTEM_DEV_CERTIFICATE := build/target/product/security/testkey
endif
```

如果配置了PRODUCT_DEFAULT_DEV_CERTIFICATE就用配置的值，否则就用源码security下的testkery了。PRODUCT_DEFAULT_DEV_CERTIFICATE的定义在device/fsl/\<product_name\>下的某个mk脚本里。  

```makefile
PRODUCT_DEFAULT_DEV_CERTIFICATE := \
        build/target/product/security/releasekey
```

所以这里的签名用的就是releasekey，而具体的使用会在后面生成升级包的脚本中讲到。  

8、这里是编译生成INTERNAL_OTA_PACKAGE_TARGET的两个核心依赖，BUILT_TARGET_FILES_PACKAGE就是target-file-package文件，DISTTOOLS是是编译升级包所需要用到的工具。  

##### BUILT_TARGET_FILES_PACKAGE

先来看下BUILT_TARGET_FILES_PACKAGE。  

```makefile
# -----------------------------------------------------------------
# A zip of the directories that map to the target filesystem.
# This zip can be used to create an OTA package or filesystem image
# as a post-build step.
#
### 12、name的生成
name := $(TARGET_PRODUCT)-target_files-$(FILE_NAME_TAG)
ifeq ($(TARGET_BUILD_TYPE),debug)
  name := $(name)_debug
endif
### 11、intermediates的生成
intermediates := $(call intermediates-dir-for,PACKAGING,target_files)
### 10、BUILT_TARGET_FILES_PACKAGE的名称生成
BUILT_TARGET_FILES_PACKAGE := $(intermediates)/$(name).zip
$(BUILT_TARGET_FILES_PACKAGE): intermediates := $(intermediates)
$(BUILT_TARGET_FILES_PACKAGE): \
		zip_root := $(intermediates)/$(name)

# $(1): Directory to copy
# $(2): Location to copy it to
# The "ls -A" is to prevent "acp s/* d" from failing if s is empty.
define package_files-copy-root
  if [ -d "$(strip $(1))" -a "$$(ls -A $(1))" ]; then \
    mkdir -p $(2) && \
    $(ACP) -rd $(strip $(1))/* $(2); \
  fi
endef
### 13、依赖的ota工具，后面会被拷贝到OTA目录下
built_ota_tools := \
	$(call intermediates-dir-for,EXECUTABLES,applypatch)/applypatch \
	$(call intermediates-dir-for,EXECUTABLES,applypatch_static)/applypatch_static \
	$(call intermediates-dir-for,EXECUTABLES,check_prereq)/check_prereq \
	$(call intermediates-dir-for,EXECUTABLES,sqlite3)/sqlite3 \
	$(call intermediates-dir-for,EXECUTABLES,updater)/updater
$(BUILT_TARGET_FILES_PACKAGE): PRIVATE_OTA_TOOLS := $(built_ota_tools)

### 14、依赖的recovery内容相关
$(BUILT_TARGET_FILES_PACKAGE): PRIVATE_RECOVERY_API_VERSION := $(RECOVERY_API_VERSION)
$(BUILT_TARGET_FILES_PACKAGE): PRIVATE_RECOVERY_FSTAB_VERSION := $(RECOVERY_FSTAB_VERSION)

ifeq ($(TARGET_RELEASETOOLS_EXTENSIONS),)
# default to common dir for device vendor
$(BUILT_TARGET_FILES_PACKAGE): tool_extensions := $(TARGET_DEVICE_DIR)/../common
else
$(BUILT_TARGET_FILES_PACKAGE): tool_extensions := $(TARGET_RELEASETOOLS_EXTENSIONS)
endif

# Depending on the various images guarantees that the underlying
# directories are up-to-date.
### 15、依赖的镜像
$(BUILT_TARGET_FILES_PACKAGE): \
		$(TARGET_BOOTLOADER_IMAGE) \
		$(INSTALLED_BOOTIMAGE_TARGET) \
		kernelimage \
		$(INSTALLED_URAMDISK_TARGET) \
		$(INSTALLED_RADIOIMAGE_TARGET) \
		$(INSTALLED_RECOVERYIMAGE_TARGET) \
		$(INSTALLED_SYSTEMIMAGE) \
		$(INSTALLED_USERDATAIMAGE_TARGET) \
		$(INSTALLED_CACHEIMAGE_TARGET) \
		$(INSTALLED_VENDORIMAGE_TARGET) \
		$(INSTALLED_ANDROID_INFO_TXT_TARGET) \
		$(SELINUX_FC) \
		$(built_ota_tools) \
		$(APKCERTS_FILE) \
		$(HOST_OUT_EXECUTABLES)/fs_config \
		| $(ACP)
	### 16、开始生成
	@echo "Package target files: $@"
	$(hide) rm -rf $@ $(zip_root)
	$(hide) mkdir -p $(dir $@) $(zip_root)
	@# Components of the recovery image
	$(hide) mkdir -p $(zip_root)/RECOVERY
	$(hide) mkdir -p $(zip_root)/BOOTABLE_IMAGES
	$(hide) $(call package_files-copy-root, \
		$(TARGET_RECOVERY_ROOT_OUT),$(zip_root)/RECOVERY/RAMDISK)
ifdef INSTALLED_KERNEL_TARGET
	$(hide) $(ACP) $(INSTALLED_KERNEL_TARGET) $(zip_root)/RECOVERY/kernel
endif
ifdef INSTALLED_2NDBOOTLOADER_TARGET
	$(hide) $(ACP) \
		$(INSTALLED_2NDBOOTLOADER_TARGET) $(zip_root)/RECOVERY/second
endif
ifdef BOARD_KERNEL_CMDLINE
	$(hide) echo "$(BOARD_KERNEL_CMDLINE)" > $(zip_root)/RECOVERY/cmdline
endif
ifdef BOARD_KERNEL_BASE
	$(hide) echo "$(BOARD_KERNEL_BASE)" > $(zip_root)/RECOVERY/base
endif
ifdef BOARD_KERNEL_PAGESIZE
	$(hide) echo "$(BOARD_KERNEL_PAGESIZE)" > $(zip_root)/RECOVERY/pagesize
endif
ifdef INSTALLED_RECOVERYIMAGE_TARGET
	$(hide) $(ACP) $(INSTALLED_RECOVERYIMAGE_TARGET) $(zip_root)/RECOVERY/recovery.img
	$(hide) $(ACP) $(INSTALLED_RECOVERYIMAGE_TARGET) $(zip_root)/BOOTABLE_IMAGES/recovery.img
endif

ifdef BUILD_URAMDISK_TARGET
	$(hide) $(ACP) $(BUILD_URAMDISK_TARGET) $(zip_root)/BOOTABLE_IMAGES/uramdisk.img
endif

	$(hide) $(ACP) $(TARGET_DEVICE_DIR)/device_all.fstab $(zip_root)/RECOVERY/device_all.fstab
	$(hide) $(ACP) $(TARGET_DEVICE_DIR)/device_allb.fstab $(zip_root)/RECOVERY/device_allb.fstab

	@# Components of the boot image
	$(hide) mkdir -p $(zip_root)/BOOT
	$(hide) $(call package_files-copy-root, \
		$(TARGET_ROOT_OUT),$(zip_root)/BOOT/RAMDISK)
ifdef INSTALLED_KERNEL_TARGET
	$(hide) $(ACP) $(INSTALLED_KERNEL_TARGET) $(zip_root)/BOOT/kernel
endif
ifdef INSTALLED_UIMAGE_TARGET
	$(hide) $(ACP) $(INSTALLED_UIMAGE_TARGET) $(zip_root)/BOOT/uImage
	$(hide) $(ACP) $(INSTALLED_UIMAGE_TARGET) $(zip_root)/BOOTABLE_IMAGES/uImage
endif
ifdef INSTALLED_2NDBOOTLOADER_TARGET
	$(hide) $(ACP) \
		$(INSTALLED_2NDBOOTLOADER_TARGET) $(zip_root)/BOOT/second
endif
ifdef BOARD_KERNEL_CMDLINE
	$(hide) echo "$(BOARD_KERNEL_CMDLINE)" > $(zip_root)/BOOT/cmdline
endif
ifdef BOARD_KERNEL_BASE
	$(hide) echo "$(BOARD_KERNEL_BASE)" > $(zip_root)/BOOT/base
endif
ifdef BOARD_KERNEL_PAGESIZE
	$(hide) echo "$(BOARD_KERNEL_PAGESIZE)" > $(zip_root)/BOOT/pagesize
endif
	@# Cotents of the mcu image
	$(hide) mkdir -p $(zip_root)/MCU
	@echo make mcu image
	$(hide) $(ACP) $(TARGET_DEVICE_DIR)/mcu/fsl_app.s19 $(zip_root)/MCU/fsl_app.s19
	@echo make mcu upgrade bin
	$(hide) $(ACP) $(TARGET_DEVICE_DIR)/mcu/upgrade.dat $(zip_root)/MCU/upgrade.dat
	@# Cotents of the logo image
	@echo make logo image
	$(hide) $(ACP) $(TARGET_DEVICE_DIR)/logo.bin $(zip_root)/BOOTABLE_IMAGES/logo.bin

	@# Components of the u-boot image
ifdef TARGET_BOOTLOADER_IMAGE
	$(hide) mkdir -p $(zip_root)/BOOTLOADER
	$(hide) mkdir -p $(zip_root)/BOOTABLE_IMAGES
	$(hide) $(ACP) \
		$(TARGET_BOOTLOADER_IMAGE) $(zip_root)/BOOTLOADER/u-boot.bin
	$(hide) $(ACP) \
		$(TARGET_BOOTLOADER_IMAGE) $(zip_root)/BOOTABLE_IMAGES/u-boot.bin
endif

	$(hide) $(foreach t,$(INSTALLED_RADIOIMAGE_TARGET),\
	            mkdir -p $(zip_root)/RADIO; \
	            $(ACP) $(t) $(zip_root)/RADIO/$(notdir $(t));)
	@# Contents of the system image
	$(hide) $(call package_files-copy-root, \
		$(SYSTEMIMAGE_SOURCE_DIR),$(zip_root)/SYSTEM)
	@# Contents of the data image
	$(hide) $(call package_files-copy-root, \
		$(TARGET_OUT_DATA),$(zip_root)/DATA)
ifdef BOARD_VENDORIMAGE_FILE_SYSTEM_TYPE
	@# Contents of the vendor image
	$(hide) $(call package_files-copy-root, \
		$(TARGET_OUT_VENDOR),$(zip_root)/VENDOR)
endif
	@# Extra contents of the OTA package
	$(hide) mkdir -p $(zip_root)/OTA/bin
	$(hide) $(ACP) $(INSTALLED_ANDROID_INFO_TXT_TARGET) $(zip_root)/OTA/
	$(hide) $(ACP) $(PRIVATE_OTA_TOOLS) $(zip_root)/OTA/bin/
	@# Files that do not end up in any images, but are necessary to
	@# build them.
	$(hide) mkdir -p $(zip_root)/META
	$(hide) $(ACP) $(APKCERTS_FILE) $(zip_root)/META/apkcerts.txt
	$(hide)	echo "$(PRODUCT_OTA_PUBLIC_KEYS)" > $(zip_root)/META/otakeys.txt
	$(hide) echo "recovery_api_version=$(PRIVATE_RECOVERY_API_VERSION)" > $(zip_root)/META/misc_info.txt
	$(hide) echo "fstab_version=$(PRIVATE_RECOVERY_FSTAB_VERSION)" >> $(zip_root)/META/misc_info.txt
ifdef BOARD_FLASH_BLOCK_SIZE
	$(hide) echo "blocksize=$(BOARD_FLASH_BLOCK_SIZE)" >> $(zip_root)/META/misc_info.txt
endif
ifdef BOARD_BOOTIMAGE_PARTITION_SIZE
	$(hide) echo "boot_size=$(BOARD_BOOTIMAGE_PARTITION_SIZE)" >> $(zip_root)/META/misc_info.txt
endif
ifdef BOARD_RECOVERYIMAGE_PARTITION_SIZE
	$(hide) echo "recovery_size=$(BOARD_RECOVERYIMAGE_PARTITION_SIZE)" >> $(zip_root)/META/misc_info.txt
endif
ifdef BOARD_SYSTEMIMAGE_PARTITION_SIZE
	$(hide) echo "system_size=$(BOARD_SYSTEMIMAGE_PARTITION_SIZE)" >> $(zip_root)/META/misc_info.txt
endif
ifdef BOARD_USERDATAIMAGE_PARTITION_SIZE
	$(hide) echo "userdata_size=$(BOARD_USERDATAIMAGE_PARTITION_SIZE)" >> $(zip_root)/META/misc_info.txt
endif
	$(hide) echo "tool_extensions=$(tool_extensions)" >> $(zip_root)/META/misc_info.txt
ifdef mkyaffs2_extra_flags
	$(hide) echo "mkyaffs2_extra_flags=$(mkyaffs2_extra_flags)" >> $(zip_root)/META/misc_info.txt
endif
ifdef INTERNAL_USERIMAGES_SPARSE_EXT_FLAG
	$(hide) echo "extfs_sparse_flag=$(INTERNAL_USERIMAGES_SPARSE_EXT_FLAG)" >> $(zip_root)/META/misc_info.txt
endif
	$(hide) echo "default_system_dev_certificate=$(DEFAULT_SYSTEM_DEV_CERTIFICATE)" >> $(zip_root)/META/misc_info.txt
ifdef PRODUCT_EXTRA_RECOVERY_KEYS
	$(hide) echo "extra_recovery_keys=$(PRODUCT_EXTRA_RECOVERY_KEYS)" >> $(zip_root)/META/misc_info.txt
endif
	$(hide) echo "mkbootimg_args=$(BOARD_MKBOOTIMG_ARGS)" >> $(zip_root)/META/misc_info.txt
	$(hide) echo "use_set_metadata=1" >> $(zip_root)/META/misc_info.txt
	$(hide) echo "update_rename_support=1" >> $(zip_root)/META/misc_info.txt
	$(call generate-userimage-prop-dictionary, $(zip_root)/META/misc_info.txt)
	@# Zip everything up, preserving symlinks
	$(hide) (cd $(zip_root) && zip -qry ../$(notdir $@) .)
	@# Run fs_config on all the system, boot ramdisk, and recovery ramdisk files in the zip, and save the output
	$(hide) zipinfo -1 $@ | awk 'BEGIN { FS="SYSTEM/" } /^SYSTEM\// {print "system/" $$2}' | $(HOST_OUT_EXECUTABLES)/fs_config -C -S $(SELINUX_FC) > $(zip_root)/META/filesystem_config.txt
	$(hide) zipinfo -1 $@ | awk 'BEGIN { FS="BOOT/RAMDISK/" } /^BOOT\/RAMDISK\// {print $$2}' | $(HOST_OUT_EXECUTABLES)/fs_config -C -S $(SELINUX_FC) > $(zip_root)/META/boot_filesystem_config.txt
	$(hide) zipinfo -1 $@ | awk 'BEGIN { FS="RECOVERY/RAMDISK/" } /^RECOVERY\/RAMDISK\// {print $$2}' | $(HOST_OUT_EXECUTABLES)/fs_config -C -S $(SELINUX_FC) > $(zip_root)/META/recovery_filesystem_config.txt
	$(hide) (cd $(zip_root) && zip -q ../$(notdir $@) META/*filesystem_config.txt)
### 9、以上为target-files-package的编译程序段
.PHONY: target-files-package
target-files-package: $(BUILT_TARGET_FILES_PACKAGE)
```

9、target-file-package解压开来看，先有个直观的印象  

├── BOOT  
│   ├── base  
│   ├── cmdline  
│   ├── kernel  
│   ├── RAMDISK  
│   └── uImage  
├── BOOTABLE_IMAGES  
│   ├── logo.bin  
│   ├── recovery.img  
│   ├── u-boot.bin  
│   ├── uImage  
│   └── uramdisk.img  
├── BOOTLOADER  
│   └── u-boot.bin  
├── MCU  
│   ├── fsl_app.s19  
│   └── upgrade.dat  
├── META  
│   ├── apkcerts.txt  
│   ├── boot_filesystem_config.txt  
│   ├── filesystem_config.txt  
│   ├── misc_info.txt  
│   ├── otakeys.txt  
│   └── recovery_filesystem_config.txt  
├── OTA  
│   ├── android-info.txt  
│   └── bin  
│       ├── applypatch  
│       ├── applypatch_static  
│       ├── check_prereq  
│       ├── sqlite3  
│       └── updater  
├── RECOVERY  
│   ├── base  
│   ├── cmdline  
│   ├── device_allb.fstab  
│   ├── device_all.fstab  
│   ├── kernel  
│   ├── RAMDISK  
│   │   ├── data  
│   │   ├── default.prop  
│   │   ├── dev  
│   │   ├── etc  
│   │   ├── file_contexts  
│   │   ├── fstab.xe110bm  
│   │   ├── init  
│   │   ├── init.rc  
│   │   ├── init.recovery.xe110bm.rc  
│   │   ├── proc  
│   │   ├── property_contexts  
│   │   ├── res  
│   │   ├── sbin  
│   └── recovery.img  
└── SYSTEM  
​    ├── app  
​    ├── bin  
​    ├── build.prop  
​    ├── etc  
​    ├── fonts  
​    ├── framework  
​    ├── lib  
​    ├── media  
​    ├── priv-app  
​    ├── system  
​    ├── usr  
​    ├── vendor  
​    └── xbin  

我裁剪掉了很多东西，还是包含了基本的结构，这边用的是厂商定制的为例，各个厂商稍有区别，但是设计的原理是一致的。**BOOT**包含了系统启动相关的镜像和环境；**BOOTABLE_IMAGES**包含的是启动相关的镜像；**BOOTLOADER**包含bootloader镜像，这里是uboot；**MCU**包含mcu升级和升级程序；**META**包含apk签名、分区、系统权限等数据信息；**OTA**包含了升级包需要用到的一些工具；**RECOVERY**包含recovery启动需要的镜像、环境等；**SYSTEM**包含了system分区的所有内容。  

10、BUILT_TARGET_FILES_PACKAGE的名称生成依赖intermediates和name  

11、intermediates的生成，调用了intermediates-dir-for方法，传入了PACKAGING和target_files参数，intermediates-dir-for的定义在definitions.mk中  

```makefile
define intermediates-dir-for
$(strip \
    $(eval _idfClass := $(strip $(1))) \
    $(if $(_idfClass),, \
        $(error $(LOCAL_PATH): Class not defined in call to intermediates-dir-for)) \
    $(eval _idfName := $(strip $(2))) \
    $(if $(_idfName),, \
        $(error $(LOCAL_PATH): Name not defined in call to intermediates-dir-for)) \
    $(eval _idfPrefix := $(if $(strip $(3)),HOST,TARGET)) \
    $(if $(filter $(_idfPrefix)-$(_idfClass),$(COMMON_MODULE_CLASSES))$(4), \
        $(eval _idfIntBase := $($(_idfPrefix)_OUT_COMMON_INTERMEDIATES)) \
      , \
        $(eval _idfIntBase := $($(_idfPrefix)_OUT_INTERMEDIATES)) \
     ) \
    $(_idfIntBase)/$(_idfClass)/$(_idfName)_intermediates \
)
endef
```

执行的过程就是目录拼接"out/target/product/\<product_name\>/obj"+"/PACKAGING"+"/target_files"+"_intermediates"，结果就是out/target/product/\<product_name\>/obj/PACKAGING/target_files_intermediates  

12、name的生成，跟步骤3中类似，就是把update-ota换成了target_files，所以name的结果例子就是xe110bm-target_files-2.5.0-B6-user，而最终BUILT_TARGET_FILES_PACKAGE生成的文件就是out/target/product/xe110bm/obj/PACKAGING/target_files_intermediates/xe110bm-target_files-2.5.0-B6-user.zip。而zip_root就是out/target/product/xe110bm/obj/PACKAGING/target_files_intermediates/xe110bm-target_files-2.5.0-B6-user，后面会创建这个目录，然后将需要的内容拷贝进去，最后将此目录打包成zip目标。  

13、依赖的ota工具

14、依赖的recovery信息

15、依赖的系统镜像

16、上面三个依赖我们都假设编译阶段生成是成功的，那么这里就没什么问题了。这边开始真正去生成target-file。首先是移除target-file目录，重新去创建出来，保证之前可能存在造成影响；然后就是按照上面文件目录列出的那样，创建8大类目录，然后往里面写信息。写信息的方式有两种：其一是用echo命令直接写，这个没什么好说的，另一种可以看到调用了```$(ACP)```，从结果来看是将需要的内容（如系统镜像），拷贝到了目标目录下。那么ACP到底是什么？在config.mk里找到了它的定义  

```makefile
# ACP is always for the build OS, not for the host OS
ACP := $(BUILD_OUT_EXECUTABLES)/acp$(BUILD_EXECUTABLE_SUFFIX)
```

其执行的程序就是out/host/linux-x86/bin/acp，它的源码本片中我们就不去看了，明白它做了什么就可以。  

##### OTATOOLS

```makefile
# -----------------------------------------------------------------
# host tools needed to build dist and OTA packages

DISTTOOLS :=  $(HOST_OUT_EXECUTABLES)/minigzip \
	  $(HOST_OUT_EXECUTABLES)/mkbootfs \
	  $(HOST_OUT_EXECUTABLES)/mkbootimg \
	  $(HOST_OUT_EXECUTABLES)/mkfile \
	  $(HOST_OUT_EXECUTABLES)/fs_config \
	  $(HOST_OUT_EXECUTABLES)/mkyaffs2image \
	  $(HOST_OUT_EXECUTABLES)/zipalign \
	  $(HOST_OUT_EXECUTABLES)/bsdiff \
	  $(HOST_OUT_EXECUTABLES)/imgdiff \
	  $(HOST_OUT_JAVA_LIBRARIES)/dumpkey.jar \
	  $(HOST_OUT_JAVA_LIBRARIES)/signapk.jar \
	  $(HOST_OUT_EXECUTABLES)/mkuserimg.sh \
	  $(HOST_OUT_EXECUTABLES)/make_ext4fs \
	  $(HOST_OUT_EXECUTABLES)/simg2img \
	  $(HOST_OUT_EXECUTABLES)/e2fsck

OTATOOLS := $(DISTTOOLS) \
	  $(HOST_OUT_EXECUTABLES)/aapt

.PHONY: otatools
otatools: $(OTATOOLS)
```

OTATOOLS就比较简单了，就是out/host/linux-x86/bin下相关的工具  

##### ota_from_target_files

万事俱备之后，就可以来看到底升级包的生成过程是怎么样的。流程开始的脚本就是前面介绍otapackage编译的一部分，这里把它抠出来。  

```makefile
$(INTERNAL_OTA_PACKAGE_TARGET): $(BUILT_TARGET_FILES_PACKAGE) $(DISTTOOLS)
	@echo "Package OTA: $@"
	@echo "Server side manifest..."
	### 17、厂商定制，生成Manifest_SYS.xml
	$(hide) ./build/tools/releasetools/create_server_manifest \
			-p $(TARGET_DEVICE) \
			-t sys \
			-i $(INTERNAL_OTA_PACKAGE_TARGET) \
			-o $(PRODUCT_OUT)/Manifest_SYS.xml
	@echo "Create ota package."
	### 18、通过ota_from_target_files来生成升级包
	$(hide) ./build/tools/releasetools/ota_from_target_files -v -w\
	   -p $(HOST_OUT) \
	   -k $(KEY_CERT_PAIR) \
	   -t $(PRODUCT_OUT)/Manifest_SYS.xml \
	   -e $(EXTRA_SCRIPT) \
	   -f $(EXTRA_SCRIPTB) \
	   -m $(EXTRA_SCRIPTMCU) \
	   $(BUILT_TARGET_FILES_PACKAGE) $(INTERNAL_OTA_PACKAGE_TARGET)
	@echo "Create ota package md5."
	### 19、厂商定制来给升级包增加md5
	$(hide) ./build/tools/releasetools/create_md5_file \
			-i $(INTERNAL_OTA_PACKAGE_TARGET) \
			-p $(TARGET_DEVICE) \
			-o $(PRODUCT_OUT)/ota_md5.txt
	$(hide) $(MKFILE)  --file $(INTERNAL_OTA_PACKAGE_TARGET) --suffix $(PRODUCT_OUT)/ota_md5.txt --suffix_size 196 --output $@
	### 20、删除用完的文件
	$(hide) rm -rf $(PRODUCT_OUT)/Manifest_SYS.xml
	$(hide) rm -rf $(PRODUCT_OUT)/ota_md5.txt
	@echo "ota update zip package done..."

```

###### create_server_manifest.create_sysinfo

17、在前面升级包结构剖析里讲到了Manifest_SYS.xml，里面是产品信息和系统信息，用来升级时做升级包校验。通过执行定制的./build/tools/releasetools/create_server_manifest脚本来将需要的信息写入到Manifest_SYS.xml中，对于全量包，调用了以下函数来生成  

```python
def create_sysinfo(parent):
    """ Create Manifest.xml for system full ota package.
    """
    top_element = parent.createElement("sys_info")
    parent.appendChild(top_element)

    addSonElement(top_element, "type", OPTIONS.type)
    addSonElement(top_element, "project", OPTIONS.project)
    addSonElement(top_element, "minSdkVersion", OPTIONS.minSdkVersion)
    addSonElement(top_element, "minCustomSdkVersion", OPTIONS.minCustomSdkVersion)
    addSonElement(top_element, "vendor", OPTIONS.vendor)

    dir_path, file_path = os.path.split(OPTIONS.input)
    addSonElement(top_element, "fileName", file_path)

    get_systemversioncode(top_element)

    addSonElement(top_element, "isfull", "1")
    addSonElement(top_element, "dependencyVersionCode", OPTIONS.dependencyVersion)
    addSonElement(top_element, "systemVersionDescription", "")
    addSonElement(top_element, "ext", "")
```

18、重头戏来了，通过ota_from_target_files来生成升级包  

这一部分厂商这边的代码被改动的较多，我们先用google原生的脚本来做基础功能的分析，然后再来看厂商定制的差异点和实现。  

原生部分在Makefile里调用时ota_from_target_files传入的参数就比较简单   

```makefile
$(INTERNAL_OTA_PACKAGE_TARGET): $(BUILT_TARGET_FILES_PACKAGE) $(DISTTOOLS)
	@echo "Package OTA: $@"
	$(hide) ./build/tools/releasetools/ota_from_target_files -v \
	   -p $(HOST_OUT) \
	   -k $(KEY_CERT_PAIR) \
	   $(BUILT_TARGET_FILES_PACKAGE) $@
```

在ota_from_target_files脚本中介绍了参数的用途  

```python
Usage:  ota_from_target_files [flags] input_target_files output_ota_package

  -b  (--board_config)  <file>
      Deprecated.

  -k (--package_key) <key> Key to use to sign the package (default is
      the value of default_system_dev_certificate from the input
      target-files's META/misc_info.txt, or
      "build/target/product/security/testkey" if that value is not
      specified).

      For incremental OTAs, the default value is based on the source
      target-file, not the target build.

  -i  (--incremental_from)  <file>
      Generate an incremental OTA using the given target-files zip as
      the starting build.

  -w  (--wipe_user_data)
      Generate an OTA package that will wipe the user data partition
      when installed.

  -n  (--no_prereq)
      Omit the timestamp prereq check normally included at the top of
      the build scripts (used for developer OTA packages which
      legitimately need to go back and forth).

  -e  (--extra_script)  <file>
      Insert the contents of file at the end of the update script.

  -a  (--aslr_mode)  <on|off>
      Specify whether to turn on ASLR for the package (on by default).
```

而Makefile里传入的-v和-p参数没有介绍，这里补充下：-p后面带打包时用到的工具，这里传入的是HOST_OUT，在envsetup.mk中定义  

```makefile
HOST_OUT := $(HOST_OUT_$(HOST_BUILD_TYPE))
```

这里也就是out/host/linux-x86，-v参数标识冗余模式，也就是打印出打包的过程日志。  

###### ota_from_target_files.main

然后就可以来看ota_from_target_files里的流程了，这是个python脚本，那么从\_\_name\_\_函数来入手  

```python
if __name__ == '__main__':
  try:
    common.CloseInheritedPipes()
    main(sys.argv[1:])
  except common.ExternalError, e:
    print
    print "   ERROR: %s" % (e,)
    print
    sys.exit(1)
```

调用定义的main函数，传入参数为argv数组第二个之后，因为第一个是命令本身。来看main函数    

```python
def main(argv):

  ### 定义一个处理参数的函数
  def option_handler(o, a):
    if o in ("-b", "--board_config"):
      pass   # deprecated
    elif o in ("-k", "--package_key"):
      OPTIONS.package_key = a
    elif o in ("-i", "--incremental_from"):
      OPTIONS.incremental_source = a
    elif o in ("-w", "--wipe_user_data"):
      OPTIONS.wipe_user_data = True
    elif o in ("-n", "--no_prereq"):
      OPTIONS.omit_prereq = True
    elif o in ("-e", "--extra_script"):
      OPTIONS.extra_script = a
    elif o in ("-a", "--aslr_mode"):
      if a in ("on", "On", "true", "True", "yes", "Yes"):
        OPTIONS.aslr_mode = True
      else:
        OPTIONS.aslr_mode = False
    elif o in ("--worker_threads"):
      OPTIONS.worker_threads = int(a)
    else:
      return False
    return True

  ### 19、ParseOptions参数解析并通过option_handler根据参数的内容给OPTIONS的成员变量赋值
  args = common.ParseOptions(argv, __doc__,
                             extra_opts="b:k:i:d:wne:a:",
                             extra_long_opts=["board_config=",
                                              "package_key=",
                                              "incremental_from=",
                                              "wipe_user_data",
                                              "no_prereq",
                                              "extra_script=",
                                              "worker_threads=",
                                              "aslr_mode=",
                                              ],
                             extra_option_handler=option_handler)
  ### 如果参数数量不对，则打印出脚本用法，退出
  if len(args) != 2:
    common.Usage(__doc__)
    sys.exit(1)
  ### 额外的脚本，原生里都是这边生成的，而我们的定制里是带额外脚本的就是前面提到的EXTRA_SCRIPT
  if OPTIONS.extra_script is not None:
    OPTIONS.extra_script = open(OPTIONS.extra_script).read()

  ### 打包开始，打印解压target-files
  print "unzipping target target-files..."
  ### 20、通过common脚本将target-files解压到临时目录下
  OPTIONS.input_tmp, input_zip = common.UnzipTemp(args[0])
    
  OPTIONS.target_tmp = OPTIONS.input_tmp
  ### 21、加载target-files压缩里的某些信息到一个字典中
  OPTIONS.info_dict = common.LoadInfoDict(input_zip)

  # If this image was originally labelled with SELinux contexts, make sure we
  # also apply the labels in our new image. During building, the "file_contexts"
  # is in the out/ directory tree, but for repacking from target-files.zip it's
  # in the root directory of the ramdisk.
  if "selinux_fc" in OPTIONS.info_dict:### selinux权限，misc_info.txt里有的，指定了file_contexts文件位置，关于file_contexts的内容可以参考SeLinux的文章
    OPTIONS.info_dict["selinux_fc"] = os.path.join(OPTIONS.input_tmp, "BOOT", "RAMDISK",
        "file_contexts")
  ### 如果传入了-v/--verbose参数，在ParseOptions里被解析，OPTIONS.verbose就是True，否则为False
  if OPTIONS.verbose:
    print "--- target info ---"
    common.DumpInfoDict(OPTIONS.info_dict)

  if OPTIONS.device_specific is None:
    ### device/fsl/xe110bm/../common(其实就是device/fsl/common)下找到额外的工具
    OPTIONS.device_specific = OPTIONS.info_dict.get("tool_extensions", None)
  if OPTIONS.device_specific is not None:
    OPTIONS.device_specific = os.path.normpath(OPTIONS.device_specific)
    print "using device-specific extensions in", OPTIONS.device_specific
  ### 建立一个有名字的临时文件，因为会被多个进程使用，所以有名字是有必要的，临时文件在使用后还是会被自动删除
  temp_zip_file = tempfile.NamedTemporaryFile()
  ### 输出的压缩包操作对象，可写方式，指定了压缩方式为ZIP_DEFLATED（相对于ZIP_STORE，如果开关机动画的打包就要用这种存储方式，如果压缩了就用不了）
  output_zip = zipfile.ZipFile(temp_zip_file, "w",
                               compression=zipfile.ZIP_DEFLATED)

  if OPTIONS.incremental_source is None:
    ### 22、全量包的方式
    WriteFullOTAPackage(input_zip, output_zip)
    if OPTIONS.package_key is None:### -p参数指定了签名秘钥
      OPTIONS.package_key = OPTIONS.info_dict.get(
          "default_system_dev_certificate",
          "build/target/product/security/testkey")
  else:
    ### -i参数指定增量包的方式
    print "unzipping source target-files..."
    OPTIONS.source_tmp, source_zip = common.UnzipTemp(OPTIONS.incremental_source)
    OPTIONS.target_info_dict = OPTIONS.info_dict
    OPTIONS.source_info_dict = common.LoadInfoDict(source_zip)
    if OPTIONS.package_key is None:
      OPTIONS.package_key = OPTIONS.source_info_dict.get(
          "default_system_dev_certificate",
          "build/target/product/security/testkey")
    if OPTIONS.verbose:
      print "--- source info ---"
      common.DumpInfoDict(OPTIONS.source_info_dict)
    WriteIncrementalOTAPackage(input_zip, source_zip, output_zip)

  output_zip.close()

  SignOutput(temp_zip_file.name, args[1])
  temp_zip_file.close()

  common.Cleanup()

  print "done."
```

这一段是整个制作升级包过程的控制框架  

###### common.ParseOptions

19、将参数读到args中，并根据参数的内容，给OPTIONS的成员变量赋值。例如-k，就将秘钥对路径赋值给package_key，原生里面全量包的时候没有用别的，在后面介绍厂商定制时会有更多的参数被使用到。来看下ParseOptions方法解析参数    

```python  
def ParseOptions(argv,
                 docstring,
                 extra_opts="", extra_long_opts=(),
                 extra_option_handler=None):
  """Parse the options in argv and return any arguments that aren't
  flags.  docstring is the calling module's docstring, to be displayed
  for errors and -h.  extra_opts and extra_long_opts are for flags
  defined by the caller, which are processed by passing them to
  extra_option_handler."""

  try:
    opts, args = getopt.getopt(
        argv, "hvp:s:x:" + extra_opts,
        ["help", "verbose", "path=", "signapk_path=", "extra_signapk_args=",
         "java_path=", "public_key_suffix=", "private_key_suffix=",
         "device_specific=", "extra="] +
        list(extra_long_opts))
  except getopt.GetoptError, err:
    Usage(docstring)
    print "**", str(err), "**"
    sys.exit(2)

  path_specified = False

  for o, a in opts:
    ### -h，-v，-p参数是在这里用的
    if o in ("-h", "--help"):
      Usage(docstring)
      sys.exit()
    elif o in ("-v", "--verbose"):
      OPTIONS.verbose = True
    elif o in ("-p", "--path"):
      OPTIONS.search_path = a
    elif o in ("--signapk_path",):
      OPTIONS.signapk_path = a
    elif o in ("--extra_signapk_args",):
      OPTIONS.extra_signapk_args = shlex.split(a)
    elif o in ("--java_path",):
      OPTIONS.java_path = a
    elif o in ("--public_key_suffix",):
      OPTIONS.public_key_suffix = a
    elif o in ("--private_key_suffix",):
      OPTIONS.private_key_suffix = a
    elif o in ("-s", "--device_specific"):
      OPTIONS.device_specific = a
    elif o in ("-x", "--extra"):
      key, value = a.split("=", 1)
      OPTIONS.extras[key] = value
    else:
      if extra_option_handler is None or not extra_option_handler(o, a):
        assert False, "unknown option \"%s\"" % (o,)

  os.environ["PATH"] = (os.path.join(OPTIONS.search_path, "bin") +
                        os.pathsep + os.environ["PATH"])

  return args
```

###### common.UnzipTemp

20、通过common（releasetools/common.py）的UnzipTemp方法解压target-files到临时目录下  

```python
def UnzipTemp(filename, pattern=None):
  """Unzip the given archive into a temporary directory and return the name.

  If filename is of the form "foo.zip+bar.zip", unzip foo.zip into a
  temp dir, then unzip bar.zip into that_dir/BOOTABLE_IMAGES.

  Returns (tempdir, zipobj) where zipobj is a zipfile.ZipFile (of the
  main file), open for reading.
  """
  ### 用python的tempfile模块在系统tmp目录（/tmp）下创建前缀未targetfiles-的目录
  tmp = tempfile.mkdtemp(prefix="targetfiles-")
  ### 并赋值给OPTIONS.tempfiles
  OPTIONS.tempfiles.append(tmp)

  def unzip_to_dir(filename, dirname):
    cmd = ["unzip", "-o", "-q", filename, "-d", dirname]
    if pattern is not None:
      cmd.append(pattern)
    ### 起子线程去执行，并拿到返回值
    p = Run(cmd, stdout=subprocess.PIPE)
    p.communicate()
    if p.returncode != 0:
      raise ExternalError("failed to unzip input target-files \"%s\"" %
                          (filename,))

  ### 正则表达式去匹配传入的filename里是不是符合“*.zip+*.zip”这种格式的，有的话就对两个zip分别解压
  m = re.match(r"^(.*[.]zip)\+(.*[.]zip)$", filename, re.IGNORECASE)
  if m:
    unzip_to_dir(m.group(1), tmp)
    unzip_to_dir(m.group(2), os.path.join(tmp, "BOOTABLE_IMAGES"))
    filename = m.group(1)
  else:
    ### 我们按照前面的流程下来就是xe110bm-target_files-2.5.0-B6-user.zip这样子，直接解压
    unzip_to_dir(filename, tmp)

  ### 返回临时目录和原压缩包的操作对象（用于后面直接读取压缩包里的文件）
  return tmp, zipfile.ZipFile(filename, "r")
```

###### common.LoadInfoDict

21、 这里就用到了前面UnzipTemp带回的ZipFile返回值来读取压缩包里的内容了，我们来看```common.LoadInfoDict```

```python
def LoadInfoDict(zip):
  """Read and parse the META/misc_info.txt key/value pairs from the
  input target files and return a dict."""

  d = {}   ###定义字典，最后返回的就是这个
  try:
    ### 首先读META/misc_info.txt，它里面的内容见后面
    for line in zip.read("META/misc_info.txt").split("\n"):
      line = line.strip()
      if not line or line.startswith("#"): continue###如果是‘#’则认为是注释，忽略掉
      k, v = line.split("=", 1)###用‘=’前后隔开
      d[k] = v###前面是key，后面是value，存入到字典中
  except KeyError:
    # ok if misc_info.txt doesn't exist
    pass

  # backwards compatibility: These values used to be in their own
  # files.  Look for them, in case we're processing an old
  # target_files zip.

  if "mkyaffs2_extra_flags" not in d:
    try:
      ### 如果前面的misc_info.txt中没有mkyaffs2_extra_flags这项，再去从mkyaffs2-extra-flags.txt尝试读一下，貌似我们的target-files里都没有这个文件。mkyaffs2image是用来打包yaffs2系统格式镜像的工具，出现在早期android系统的系统格式中，现在都是ext4的了
      d["mkyaffs2_extra_flags"] = zip.read("META/mkyaffs2-extra-flags.txt").strip()
    except KeyError:
      # ok if flags don't exist
      pass

  if "recovery_api_version" not in d:### 这个在misc_info.txt里有
    try:
      d["recovery_api_version"] = zip.read("META/recovery-api-version.txt").strip()
    except KeyError:
      raise ValueError("can't find recovery API version in input target-files")

  if "tool_extensions" not in d:### 这个在misc_info.txt里有
    try:
      d["tool_extensions"] = zip.read("META/tool-extensions.txt").strip()
    except KeyError:
      # ok if extensions don't exist
      pass

  if "fstab_version" not in d:### 这个在misc_info.txt里有
    d["fstab_version"] = "1"

  try:
    ### 我们的META下没有imagesizes.txt，但是在misc_info.txt里有blocksize、system_size、userdata_size，而且后面两个还出现了两条重复数据，看来被人改的有问题啊。  
    data = zip.read("META/imagesizes.txt")
    for line in data.split("\n"):
      if not line: continue
      name, value = line.split(" ", 1)
      if not value: continue
      if name == "blocksize":
        d[name] = value
      else:
        d[name + "_size"] = value
  except KeyError:
    pass

  def makeint(key):
    if key in d:
      d[key] = int(d[key], 0)
  ### 转一波数据格式
  makeint("recovery_api_version")
  makeint("blocksize")
  makeint("system_size")
  makeint("userdata_size")
  makeint("cache_size")
  makeint("recovery_size")
  makeint("boot_size")
  makeint("fstab_version")
  ### 加载A/B系统的fstab分区信息，这里是定制过的
  ### 原生的长这样d["fstab"] = LoadRecoveryFSTab(zip, d["fstab_version"])
  d["fstab"] = LoadRecoveryFSTab(zip, "RECOVERY/device_all.fstab",d["fstab_version"])
  d["fstabb"] = LoadRecoveryFSTab(zip, "RECOVERY/device_allb.fstab",d["fstab_version"])
  ### 加载build.prop里的系统属性信息
  d["build.prop"] = LoadBuildProp(zip)
  return d
```

META/misc_info.txt  

```reStructuredText
recovery_api_version=3
fstab_version=2
blocksize=4096
system_size=1572864000
userdata_size=377487360
tool_extensions=device/fsl/xe110bm/../common
default_system_dev_certificate=build/target/product/security/releasekey
mkbootimg_args=
use_set_metadata=1
update_rename_support=1
fs_type=ext4
system_size=1572864000
userdata_size=377487360
resource_size=1551892480
selinux_fc=out/target/product/xe110bm/root/file_contexts
```

###### ota_from_target_files.WriteFullOTAPackage

22、全量包制作里的重中之重，前面都是在准备环境，创建好临时目录，目标压缩文件等周边工作，这里调用WriteFullOTAPackage来真正生成一个全量包的数据    

```python
def WriteFullOTAPackage(input_zip, output_zip):
  ### input_zip就是target-files的包，output_zip是用NamedTemporaryFile生成的临时压缩包
  # TODO: how to determine this?  We don't know what version it will
  # be installed on top of.  For now, we expect the API just won't
  # change very often.
  ### 初始化updater-script脚本，具体见后面的EdifyGenerator
  script = edify_generator.EdifyGenerator(3, OPTIONS.info_dict)
  ### [厂商定制]用于A/B系统不同的升级脚本（差别主要在system分区不同，和升级完之后切换系统指定的参数）
  scriptA = edify_generator.EdifyGenerator(3, OPTIONS.info_dict)
  if OPTIONS.extra_scriptB is not None:
	scriptB = edify_generator.EdifyGenerator(3, OPTIONS.info_dict)
  ### 这里就是在升级包结构剖析里看到的metadata数据来源
  metadata = {"post-build": GetBuildProp("ro.build.fingerprint",
                                         OPTIONS.info_dict),
              "pre-device": GetBuildProp("ro.product.device",
                                         OPTIONS.info_dict),
              "post-timestamp": GetBuildProp("ro.build.date.utc",
                                             OPTIONS.info_dict),
              }

  ### device_specific标示了额外的执行工具和脚本，在原生的ota_from_target_files的main函数里如果OPTIONS.device_specific为空，则用OPTIONS.info_dict.get("tool_extensions", None)这个值，这个也就是META/misc_info.txt里的tool_extensions，这里后续可以尝试下自己加个额外的脚本，看看能不能用来实现最简单的添加方式。在[厂商定制]里，添加了一个-s参数，用来直接指定OPTIONS.device_specific参数
  device_specific = common.DeviceSpecificParams(
      input_zip=input_zip,
      input_version=OPTIONS.info_dict["recovery_api_version"],
      output_zip=output_zip,
      script=script,
      input_tmp=OPTIONS.input_tmp,
      metadata=metadata,
      info_dict=OPTIONS.info_dict)
  ### [厂商定制]创建system分区挂载目录，因为A/B系统升级，是在A系统运行时，去升级B系统，两个系统的system分区不同，所以要去把B系统的system分区先挂载到一个目录下，然后才能写入
  ### 生成到updater-script脚本里的语句就是“assert(run_program("/system/bin/busybox", "mkdir", "-p", "/data/update_s/system"));”
  script.Mkdir(OPTIONS.src_file+"system")
  ### 时间戳校验，-n/--no_prereq参数指定，如果指定了，则omit_prereq=True
  if not OPTIONS.omit_prereq:
    ts = GetBuildProp("ro.build.date.utc", OPTIONS.info_dict)
    ts_text = GetBuildProp("ro.build.date", OPTIONS.info_dict)
    ### 往updater-script里添加时间戳校验的语句（edify语法），详见后面的AssertOlderBuild
    ### 生成到updater-script脚本里的语句就是“(!less_than_int(1544594247, getprop("ro.build.date.utc"))) || abort("Can't install this package (2018年 12月 12日 星期三 13:57:27 CST) over newer build (" + getprop("ro.build.date") + ").");”
    script.AssertOlderBuild(ts, ts_text)
  ###　添加判断产品名称是否一致的语句
  ###　生成到updater-script脚本里的语句就是“getprop("ro.product.device") == "xe110bm" || abort("This package is for \"xe110bm\" devices; this is a \"" + getprop("ro.product.device") + "\".");”
  AppendAssertions(script, OPTIONS.info_dict)
  ### device_specific的使用方法，详见后面  
  device_specific.FullOTA_Assertions()
  device_specific.FullOTA_InstallBegin()
  ### 添加显示升级进度语句  
  ### show_progress(0.500000, 0);
  ### 进度的分割可以根据自己的升级流程来做自定义，比如我这里可以是0.2
  script.ShowProgress(0.5, 0)
  ### 是否格式化data分区，根据-w/--wipe_user_data参数来指定为True
  if OPTIONS.wipe_user_data:
    script.FormatPartition("/data")
  ### 写入selinux权限文件到升级包，后面详见WritePolicyConfig方法
  if "selinux_fc" in OPTIONS.info_dict:
    WritePolicyConfig(OPTIONS.info_dict["selinux_fc"], output_zip)
  ### 格式化system分区，FormatPartition详见后面
  script.FormatPartition("/system")
  ### 挂载system分区，Mount函数跟FormatPartition很像，这里就不多介绍了
  script.Mount("/system")
  ### 解压到指定路径下，这里是原生方式，而[厂商定制]的部分由于双系统升级特性，这里被大改特改  
  script.UnpackPackageDir("recovery", "/system")
  script.UnpackPackageDir("system", "/system")
  ### 用于创建system可执行程序和工具的符号链接，CopySystemFiles和MakeSymlinks详见后面
  symlinks = CopySystemFiles(input_zip, output_zip)
  script.MakeSymlinks(symlinks)

  ### 取到boot_img和recovery_img，GetBootableImage方法这里就不展开了，就是通过传入的参数到target-file解开的临时目录下去BOOT和RECOVERY下找对应的镜像
  boot_img = common.GetBootableImage("boot.img", "boot.img",
                                     OPTIONS.input_tmp, "BOOT")
  recovery_img = common.GetBootableImage("recovery.img", "recovery.img",
                                         OPTIONS.input_tmp, "RECOVERY")
  ### 对boot和recovery镜像做patch，详见后面的MakeRecoveryPatch
  MakeRecoveryPatch(OPTIONS.input_tmp, output_zip, recovery_img, boot_img)

  Item.GetMetadata(input_zip)
  Item.Get("system").SetPermissions(script)
  
  ### 写入boot.img，并往脚本里写入写镜像命令
  common.CheckSize(boot_img.data, "boot.img", OPTIONS.info_dict)
  common.ZipWriteStr(output_zip, "boot.img", boot_img.data)
  script.ShowProgress(0.2, 0)

  script.ShowProgress(0.2, 10)
  script.WriteRawImage("/boot", "boot.img")

  script.ShowProgress(0.1, 0)
  ### device_specific的FullOTA_InstallEnd回调，同样在releasetools.py中可定制
  device_specific.FullOTA_InstallEnd()

  ### 添加额外的脚本到升级脚本中
  if OPTIONS.extra_script is not None:
    script.AppendExtra(OPTIONS.extra_script)

  ### 添加卸载语句，卸载所有挂载的盘符
  script.UnmountAll()
  ### 将脚本添加到升级包中
  script.AddToZip(input_zip, output_zip)
  ### 往升级包中写入metadata数据到META-INF/com/android/metadata文件中，打完收工
  WriteMetadata(metadata, output_zip)
```

###### edify_generator.EdifyGenerator

edify_generator.py的EdifyGenerator类，会调用```__init__```来做初始化，传入的参数“3”和“OPTIONS.info_dict”  

```python
class EdifyGenerator(object):
  """Class to generate scripts in the 'edify' recovery script language
  used from donut onwards."""

  def __init__(self, version, info):
    self.script = []
    self.mounts = set()
    self.version = version
    self.info = info
```

###### edify_generator.AssertOlderBuild

edify_generator.py的AssertOlderBuild方法，添加一行时间戳校验的语句  

```python
def AssertOlderBuild(self, timestamp, timestamp_text):
    """Assert that the build on the device is older (or the same as)
    the given timestamp."""
    self.script.append(
        ('(!less_than_int(%s, getprop("ro.build.date.utc"))) || '
         'abort("Can\'t install this package (%s) over newer '
         'build (" + getprop("ro.build.date") + ").");'
         ) % (timestamp, timestamp_text))
```

###### common.DeviceSpecificParams

common.py里的DeviceSpecificParams类，device_specific的使用方法，还未亲测，可以参考google源码下device/google/dragon/releasetools.py   

```python
def _DoCall(self, function_name, *args, **kwargs):
    """Call the named function in the device-specific module, passing
    the given args and kwargs.  The first argument to the call will be
    the DeviceSpecific object itself.  If there is no module, or the
    module does not define the function, return the value of the
    'default' kwarg (which itself defaults to None)."""
    if self.module is None or not hasattr(self.module, function_name):
      return kwargs.get("default", None)
    return getattr(self.module, function_name)(*((self,) + args), **kwargs)
  def FullOTA_Assertions(self):
    """Called after emitting the block of assertions at the top of a
    full OTA package.  Implementations can add whatever additional
    assertions they like."""
    return self._DoCall("FullOTA_Assertions")

  def FullOTA_InstallBegin(self):
    """Called at the start of full OTA installation."""
    return self._DoCall("FullOTA_InstallBegin")

  def FullOTA_InstallEnd(self):
    """Called at the end of full OTA installation; typically this is
    used to install the image for the device's baseband processor."""
    return self._DoCall("FullOTA_InstallEnd")
```

###### releasetools.py

device/google/dragon/releasetools.py，我们可以用此方法来做额外的命令添加定制，这里为什么是releasetools.py，参考前面DeviceSpecificParams初始化的时候加载的模块名称是releasetools      

```python
# Copyright (C) 2015 The Android Open Source Project
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

"""Add extra commands needed for firmwares update during OTA."""

import common

def FullOTA_InstallEnd(info):
  # copy the data into the package.
  try:
    bootloader_img = info.input_zip.read("RADIO/bootloader.img")
    ec_img = info.input_zip.read("RADIO/ec.bin")
  except KeyError:
    print "no firmware images in target_files; skipping install"
    return
  # copy the data into the package.
  common.ZipWriteStr(info.output_zip, "bootloader.img", bootloader_img)
  common.ZipWriteStr(info.output_zip, "ec.bin", ec_img)

  # emit the script code to trigger the firmware updater on the device
  info.script.AppendExtra(
    """dragon.firmware_update(package_extract_file("bootloader.img"), package_extract_file("ec.bin"));""")

def IncrementalOTA_InstallEnd(info):
  try:
    target_bootloader_img = info.target_zip.read("RADIO/bootloader.img")
    target_ec_img = info.target_zip.read("RADIO/ec.bin")
  except KeyError:
    print "no firmware images in target target_files; skipping install"
    return

  # copy the data into the package irrespective of source and target versions.
  # If previous OTA missed a firmware update, we can try that again on the next
  # OTA.
  common.ZipWriteStr(info.output_zip, "bootloader.img", target_bootloader_img)
  common.ZipWriteStr(info.output_zip, "ec.bin", target_ec_img)

  # emit the script code to trigger the firmware updater on the device
  info.script.AppendExtra(
    """dragon.firmware_update(package_extract_file("bootloader.img"), package_extract_file("ec.bin"));""")
```

###### ota_from_target_files.WritePolicyConfig

ota_from_target_files脚本的WritePolicyConfig将selinux权限文件写入到升级包，传入的参数未file_context文件的路径和升级包的输出压缩包，file_context是从OPTIONS.info_dict["selinux_fc"]的值，其源头就是misc_info.txt里的“selinux_fc=out/target/product/xe110bm/root/file_contexts”  

```python
def WritePolicyConfig(file_context, output_zip):
  f = open(file_context, 'r');
  basename = os.path.basename(file_context)
  common.ZipWriteStr(output_zip, basename, f.read())
```

###### edify_generator.FormatPartition

edify_generator.py的FormatPartition方法，用来生成格式化分区的语句到updater-script脚本中，分区的信息是从fstab中根据传入的分区名来取到的，最后生成语句例“format("ext4", "EMMC", "/dev/block/mmcblk0p2", "0", "/data/update_s/system");”    

```python
def FormatPartition(self, partition):
    """Format the given partition, specified by its mount point (eg,
    "/system")."""

    reserve_size = 0
    fstab = self.info.get("fstab", None)
    if fstab:
      p = fstab[partition]
      self.script.append('format("%s", "%s", "%s", "%s", "%s");' %
                         (p.fs_type, common.PARTITION_TYPES[p.fs_type],
                          p.device, p.length, p.mount_point))
```

###### edify_generator.UnpackPackageDir

edify_generator.py的UnpackPackageDir方法，将指定的目录解压的到目标目录，例如“package_extract_dir("system", "/data/update_s/system");”

```python
def UnpackPackageDir(self, src, dst):
    """Unpack a given directory from the OTA package into the given
    destination directory."""
    self.script.append('package_extract_dir("%s", "%s");' % (src, dst))
```

###### ota_from_target_files.CopySystemFiles

ota_from_target_files的CopySystemFiles

```python
def CopySystemFiles(input_zip, output_zip=None,
                    substitute=None):
  """Copies files underneath system/ in the input zip to the output
  zip.  Populates the Item class with their metadata, and returns a
  list of symlinks.  output_zip may be None, in which case the copy is
  skipped (but the other side effects still happen).  substitute is an
  optional dict of {output filename: contents} to be output instead of
  certain input files.
  """

  symlinks = []

  for info in input_zip.infolist():
    if info.filename.startswith("SYSTEM/"):### target-files目录了，system放在SYSTEM目录下
      basefilename = info.filename[7:]
      if IsSymlink(info):
        symlinks.append((input_zip.read(info.filename),
                         OPTIONS.src_file + "system/" + basefilename))### 其实就是把“SYSTEM/”换成了“system/”
      else:
        info2 = copy.copy(info)
        fn = info2.filename = "system/" + basefilename
        if substitute and fn in substitute and substitute[fn] is None:
          continue
        if output_zip is not None:
          if substitute and fn in substitute:
            data = substitute[fn]
          else:
            data = input_zip.read(info.filename)
          output_zip.writestr(info2, data)
        if fn.endswith("/"):
          Item.Get(fn[:-1], dir=True)
        else:
          Item.Get(fn, dir=False)

  symlinks.sort()//默认按名称排序
  return symlinks
```

这里用到了IsSymlink来判断一个文件是否一个符号链接，这里用移位判断的原理没有仔细去了解，猜测应该是跟文件权限参数相关  

```python
def IsSymlink(info):
  """Return true if the zipfile.ZipInfo object passed in represents a
  symlink."""
  return (info.external_attr >> 16) == 0120777
```

###### edify_generator.MakeSymlinks

这里也没有深究名称都是如何计算出来的，了解其结果就是往updater-script脚本里插入symlink语句，如“symlink("mksh", "/data/update_s/system/bin/sh");”，这个语句执行的结果就是为system/bin/sh可执行文件创建符号链接mksh，可以直接通过mksh来执行，类似于“ln -s”命令，在生成的脚本里会有一大坨工具是被指向到toolbox里的。  

```python
def MakeSymlinks(self, symlink_list):
    """Create symlinks, given a list of (dest, link) pairs."""
    by_dest = {}
    for d, l in symlink_list:
      by_dest.setdefault(d, []).append(l)

    for dest, links in sorted(by_dest.iteritems()):
      cmd = ('symlink("%s", ' % (dest,) +
             ",\0".join(['"' + i + '"' for i in sorted(links)]) + ");")
      self.script.append(self._WordWrap(cmd))
```

###### ota_from_target_files.MakeRecoveryPatch  

```python
def MakeRecoveryPatch(input_tmp, output_zip, recovery_img, boot_img):
  """Generate a binary patch that creates the recovery image starting
  with the boot image.  (Most of the space in these images is just the
  kernel, which is identical for the two, so the resulting patch
  should be efficient.)  Add it to the output zip, along with a shell
  script that is run from init.rc on first boot to actually do the
  patching and install the new recovery image.

  recovery_img and boot_img should be File objects for the
  corresponding images.  info should be the dictionary returned by
  common.LoadInfoDict() on the input target_files.

  Returns an Item for the shell script, which must be made
  executable.
  """

  diff_program = ["imgdiff"]###这里用到了做patch的工具，镜像文件的话用imgdiff效率高，普通文件用bsdiff，关于这两个工具后面专门有一篇介绍  
  path = os.path.join(input_tmp, "SYSTEM", "etc", "recovery-resource.dat")
  if os.path.exists(path):
    diff_program.append("-b")
    diff_program.append(path)
    bonus_args = "-b /system/etc/recovery-resource.dat"
  else:
    bonus_args = ""

  d = common.Difference(recovery_img, boot_img, diff_program=diff_program)
  _, _, patch = d.ComputePatch()
  common.ZipWriteStr(output_zip, "recovery/recovery-from-boot.p", patch)
  Item.Get("system/recovery-from-boot.p", dir=False)

  boot_type, boot_device = common.GetTypeAndDevice("/boot", OPTIONS.info_dict)
  recovery_type, recovery_device = common.GetTypeAndDevice("/recovery", OPTIONS.info_dict)

  ### 生成一个shell脚本来安装recovery
  sh = """#!/system/bin/sh
if ! applypatch -c %(recovery_type)s:%(recovery_device)s:%(recovery_size)d:%(recovery_sha1)s; then
  log -t recovery "Installing new recovery image"
  applypatch %(bonus_args)s %(boot_type)s:%(boot_device)s:%(boot_size)d:%(boot_sha1)s %(recovery_type)s:%(recovery_device)s %(recovery_sha1)s %(recovery_size)d %(boot_sha1)s:/system/recovery-from-boot.p
else
  log -t recovery "Recovery image already installed"
fi
""" % { 'boot_size': boot_img.size,
        'boot_sha1': boot_img.sha1,
        'recovery_size': recovery_img.size,
        'recovery_sha1': recovery_img.sha1,
        'boot_type': boot_type,
        'boot_device': boot_device,
        'recovery_type': recovery_type,
        'recovery_device': recovery_device,
        'bonus_args': bonus_args,
        }
  common.ZipWriteStr(output_zip, "recovery/etc/install-recovery.sh", sh)
  return Item.Get("system/etc/install-recovery.sh", dir=False)
```

很多厂商定制都会去掉这一部分，将boot和recovery镜像全拷贝，然后升级时通过写分区的方式来做升级，这种定制方式的介绍见后面[镜像文件全拷贝升级定制方式](http://chendongqi.me/2018/12/17/UpdatePackages/#镜像文件全拷贝升级定制方式)

#### 增量包



#### 镜像文件全拷贝升级定制方式

会有几种实现方法  

##### 1 修改WriteFullOTAPackage

这里的修改点为去除MakeRecoveryPatch，将img写入到升级包中，然后将写分区镜像的命令去控制升级  

```python
  #MakeRecoveryPatch(OPTIONS.input_tmp, output_zip, recovery_img, boot_img)

  Item.GetMetadata(input_zip)
  Item.Get("system").SetPermissions(script)

  common.CheckSize(boot_img.data, "boot.img", OPTIONS.info_dict)
  common.ZipWriteStr(output_zip, "boot.img", boot_img.data)
  script.WriteRawImage("/boot", "boot.img")
```

以上为完整的第一种方式，比较简单。第二种方式的实现稍微复杂一点，移除MakeRecoveryPatch；去掉需要的镜像，将镜像写入到升级包中；在Makefile里执行ota_from_target_files是指定参数-e，EXTRA_SCRIPT就是额外的脚本位置。就可以把这个脚本添加到最终的升级脚本中。  

```python
 bootloader_img = common.GetBootableImage("u-boot.bin", "u-boot.bin",
                                     OPTIONS.input_tmp, "BOOTLOADER")
    
 common.ZipWriteStr(output_zip, "u-boot.bin", bootloader_img.data, perms=0755)
 if OPTIONS.extra_script is not None:
     scriptA.AppendExtra(OPTIONS.extra_script)
```

##### 2 额外的升级脚本

被放在device/fsl/\<product_name\>下，写镜像的语句就是在这里写的，这样就形成了一个完成的方案。    

```python
assert(run_program("/system/bin/busybox", "mkdir", "-p", "/data/__update__/"));
assert(write_dev("/sys/devices/platform/debug_control/set_writeable", "all=on"));
assert(package_extract_file("u-boot.bin", "/data/__update__/u-boot.bin"));
assert(write_raw_image("/data/__update__/u-boot.bin", "Bootloader"));
assert(write_dev("/sys/devices/platform/debug_control/set_writeable", "all=off"));
run_program("/system/bin/busybox", "rm", "-rf", "/data/__update__/u-boot.bin");
assert(package_extract_file("uImage", "/data/__update__/uImage"));
assert(dd("/data/__update__/uImage", "/dev/block/mmcblk0", "512", "2048", "0"));
assert(package_extract_file("logo.bin", "/data/__update__/logo.bin"));
assert(dd("/data/__update__/logo.bin", "/dev/block/mmcblk0", "512", "32768", "0"));
assert(package_extract_file("uramdisk.img", "/data/__update__/uramdisk.img"));
assert(dd("/data/__update__/uramdisk.img", "/dev/block/mmcblk0", "512", "12288", "0"));
assert(package_extract_file("recovery.img", "/data/__update__/recovery.img"));
run_program("/system/bin/mke2fs", "-O", "has_journal,huge_file,flex_bg,uninit_bg,dir_nlink,extra_isize", "-q", "/dev/block/mmcblk0p3");
assert(dd("/data/__update__/recovery.img", "/dev/block/mmcblk0p3", "512", "0", "0"));
run_program("/system/bin/busybox", "rm", "-rf", "/data/__update__/uImage");
run_program("/system/bin/busybox", "rm", "-rf", "/data/__update__/uramdisk.img");
run_program("/system/bin/busybox", "rm", "-rf", "/data/__update__/recovery.img");
run_program("/system/bin/busybox", "rm", "-rf", "/data/__update__/");
wipe_cache(0xB);
switch_system(1);
```



  ### 问题整理

#### 1 META数据放置不合理

target-files压缩包里的数据有问题，再LoadInfoDict的时候会发现数据分类不合理（有的文件没有，有的数据被放在了一个文件里），而且数据有重复。  

#### 2 关于升级包的md5

我们做了定制，给升级包加了md5，然后预留了一个变量INTERNAL_OTA_PACKAGE_TARGET_TEMP想保存没有加md5的升级包（遵循android原生方式），但是后面没有实现。  

#### 3 A/B系统的脚本是否可以合并

思考此问题的解决方法　　

####  4 OPTIONS.device_specific的额外命令定制

尝试用此方法来做定制，成功的话则可以用于额外命令的统一添加实现  