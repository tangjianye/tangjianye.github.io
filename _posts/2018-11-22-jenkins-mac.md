---
layout: post  
title:  MAC Jenkins节点配置  
date:   2018-11-22 
tags: [Tools]  
categories: Tools  
author: JayaTang  
description: MAC Jenkins节点配置   
---
文档主要介绍Jenkins主从节点配置，文档的前提条件是有一台centos7虚拟机，虚拟机是master主机，mac机配置slave节点。具体的Jenkins配置请参考另外一篇文章[《Centos搭建Android CI环境》](https://tangjianye.github.io/android/2017/05/23/android-centos-jenkins)。现在以主机已经搭建最小化的Jenkins环境，从机已经搭建android和ios编译环境为例，介绍Jenkins节点配置。 

## 环境介绍
- 主机环境介绍：主机Jenkins运行在tomcat中。Jenkins本身安装的环境仅包括java环境和gradle环境。
  ```bash
  # set java environment
  export JAVA_HOME=/usr/java
  export JRE_HOME=/usr/java/jre
  export CLASS_PATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
  export PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin

  # set gradle
  export GRADLE_HOME=/usr/local/gradle/gradle-3.3
  export PATH=$PATH:$GRADLE_HOME/bin

  # set tomcat
  export TOMCAT_HOME=/usr/local/tomcat/apache-tomcat-7.0.79

  # set jenkins
  export JENKINS_HOME=/usr/local/tomcat/apache-tomcat-7.0.79/webapps/jenkins
  ```

- 从机环境介绍：从机mac可以不安装jenkins，只是配置好android和ios的编译环境，此处的IOS编译环境尽量是已经可以通过 xcode 编译项目。
  ```bash
  # set android sdk
  export ANDROID_HOME=/Users/aorise/Library/Android/sdk
  export PATH=$PATH:$ANDROID_HOME/tools:$ANDROID_HOME/platform-tools
  
  # set gradle
  GRADLE_HOME=/Users/aorise/.gradle/wrapper/dists/gradle-4.1-all/bzyivzo6n839fup2jbap0tjew/gradle-4.1;
  export GRADLE_HOME
  export PATH=$PATH:$GRADLE_HOME/bin
  
  # set tomcat
  export TOMCAT_HOME=/users/aorise/Library/ApacheTomcat
  export JENKINS_HOME=/users/aorise/Library/ApacheTomcat/webapps/jenkins
  ```

## 打开MAC电脑远程登录
MAC电脑进入 '系统偏好设置' -> '共享' -> '远程登录' 打开远程登录。对应的账号设置可以被主节点访问后，mac 节点的 jenkins 工作区间要配置到对应的账号名下。本文中 mac 账号是 aorise ,对应配置为从节点的 jenkins 工作区间是 '/Users/aorise/Library/ApacheTomcat/webapps/jenkins'


## 主机配置
- 安装 Xcode integration：ios编译(貌似不安装也可以)，本文采用 shell 脚本编译，没有采用 xcode 的工具   
- 安装 description setter plugin 插件：生成二维码(同步打开 jenkins -> 系统配置 -> 全局安全配置 -> Markup Formatter -> Safe HTML)
```bash
.*qrcodeHistory\\/(\S{64})
<img src='http://www.pgyer.com/app/qrcodeHistory/\1' alt="二维码解析失败"/>
```
- 安装 Git Parameter Plug-In 插件：参数构建配置git参数
- 使用 Archive the artifacts 插件：存档构建输出物
```bash
platform/build/outputs/apk/sample/release/*.apk
build/Education/*.ipa
```

![全局工具配置](/assets/img/jenkins-mac/全局工具配置.png) 

### 节点配置    
在系统管理 / 节点管理 创建新节点。      
![全局工具配置](/assets/img/jenkins-mac/MAC节点配置.png)     

### 编译命令配置
记住ios工程一定要是已经可以直接通过xcode编译，也即相关的签名文件已经记录在工程文件里面。

![全局工具配置](/assets/img/jenkins-mac/ios-education-configure.png) 

实际编译脚本 'jenins-build.sh'    
```bash
#!/bin/sh

echo "\n"
echo "================= jenkins-build start ================="
#project名字
PRODUCT="Education"
#编译的类型 Release Debug
CONFIG="Release"
#蒲公英上的User Key
uKey="6964024629aa398dac6797a7c2b76a47"
#蒲公英上的API Key
apiKey="7ad0915d93547db42601cddb2e7aacc0"
#证书 
#IDENTITY="iPhone Distribution: Hunan aosheng Information technology co., LTD (37Z2L334TB)"
#描述文件UUID 
#PROFILE="5da01fd7-ac7b-43bf-bd83-6c57e1e622d8"


echo "MAC HOME:${HOME}"
echo "WORKSPACE:${WORKSPACE}"
echo "GIT_BRANCH:${GIT_BRANCH}"
echo "JOB_NAME:${JOB_NAME}"
echo "SIGNATURE:${SIGNATURE}"

if [[ ${SIGNATURE} ]] ; then 
  CONFIG="Release"
else 
  CONFIG="Debug"    
fi
echo "CONFIG:${CONFIG}"

echo "\n"
echo "================= 获取IPA版本信息 ================="
# 取版本号
bundleShortVersion=$(/usr/libexec/PlistBuddy -c "print CFBundleShortVersionString" "${WORKSPACE}/${PRODUCT}/Info.plist")
# 取build值
bundleVersion=$(/usr/libexec/PlistBuddy -c "print CFBundleVersion""${WORKSPACE}/${PRODUCT}/Info.plist")
# 日期
DATE="$(date +%Y%m%d)"
# IPA文件名字
IPANAME="${PRODUCT}-V${bundleShortVersion}-${DATE}-${CONFIG}.ipa"
echo "IPANAME:${IPANAME}"

echo "\n"
echo "================= 编译IPA文件 ================="
mkdir ${WORKSPACE}/build/;

# 解锁对login.keychain的访问，codesign会用到
/usr/bin/security unlock-keychain -p "aorise" /Users/aorise/Library/Keychains/login.keychain-db
# 清除
xcodebuild clean -workspace ${WORKSPACE}/${PRODUCT}.xcworkspace -scheme ${PRODUCT} -configuration ${CONFIG};
# 打包签名
xcodebuild archive -workspace ${WORKSPACE}/${PRODUCT}.xcworkspace -scheme ${PRODUCT} -archivePath ${WORKSPACE}/build/${PRODUCT}.xcarchive;
# 导出ipa
xcodebuild -exportArchive -archivePath ${WORKSPACE}/build/${PRODUCT}.xcarchive -exportOptionsPlist ${WORKSPACE}/keystore/ExportOptions${CONFIG}.plist -allowProvisioningUpdates -exportPath ${WORKSPACE}/build/${PRODUCT}/;

mv ${WORKSPACE}/build/${PRODUCT}/${PRODUCT}.ipa ${WORKSPACE}/build/${PRODUCT}/${IPANAME};

echo "\n"
echo "================= APP发布到蒲公英 ================="
FILENAME=`find ${WORKSPACE}/build/${PRODUCT}/ -type f -name "*.ipa"`
curl -F "file=@${FILENAME}" -F "uKey=${uKey}" -F "_api_key=${apiKey}" http://www.pgyer.com/apiv1/app/upload


echo "\n"
echo "================= jenkins-build end ================="
```

## 注意事项

- 从机sh脚本可以独立编译通过，通过主机编译就不可以？
> 打开系统钥匙串，解锁 '系统' 钥匙串一次。如果还不行，对登录用户的证书里面 MAC 编译对应的证书访问属性开启 '允许所有应用访问此项目'
