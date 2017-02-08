### 发布aar 到私服 Artifacts 

😂😂😂😂😂  
开车了 开车了  

首先上一个 Artifacts 成功运行后的截图

![Untitled Image](https://raw.githubusercontent.com/sdaduanbilei/posts/master/images/Users/scorpio/Documents/posts/images/QQ20170208-094613.png)


上车

1.在项目<font color= "red">libray 的build.gradle</font> 最后面添加发布脚本；

```java
apply from: 'https://raw.githubusercontent.com/sdaduanbilei/artifactoy_push/master/artifactoy_push.gradle'
```
或者下载 此脚本放入项目
```java
apply from:'../artifactoy_push.gradle'
```
2.在项目的 gradle.properties 设置初始化信息，对于多项目的可以 可以在每个library 创建此文件并进行配置，即可以单独控制每个library 信息；
```java
artifactory_groupid=com.xxx // 公司名、包名
artifactory_version=1.0.0 // 版本
artifactory_user=admin // artifactory 默认admin 账号
artifactory_password= password // admin 默认密码
artifactory_contextUrl=http://192.168.253.182:8081/artifactory  // 服务器地址artifactory 
```
<font color = "red">注:对于多library 的 项目级别的 gradle.properties 必须进行配置或者每个library 都进行配置</font>

3.发布aar
```java
gradle assembleRelease artifactoryPublish
```

![Untitled Image](https://raw.githubusercontent.com/sdaduanbilei/posts/master/images/Users/scorpio/Desktop/QQ20170208-102242.png)

😯😯  都发布成功了。

然后在  artifactory 就可以看到 此项目
![Untitled Image](https://raw.githubusercontent.com/sdaduanbilei/posts/master/images/Users/scorpio/Desktop/QQ20170208-102648.png)

4.引用aar
在项目的  build.gradle 中添加本地maven参考地址
```java
	allprojects {
    repositories {
        jcenter()
        maven { url "https://jitpack.io" }
        maven { url 'http://192.168.253.182:8081/artifactory/libs-release-local/' }
    }
}
```
在app 的 build.gradle 中添加compile
```java
compile(group: 'com.xxx', name: 'BaseLib', version: '1.0.6', ext: 'aar')
```
然后就可以正常使用；

到站了 到站了