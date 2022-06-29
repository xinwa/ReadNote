1.多模块通用配置

多 module 项目中共享的通用配置可以在工程根目录下的 build.gradle 文件中进行配置

```groovy
subprojects {
  apply from : "$project.rootDir}"/common_config.gradle
  dependencies {
  }
}
```



2.aar 函数库的引用

依赖模块中 libs 文件目录下的 jar 文件

```groovy
repositories {
  flatDir {
    dirs 'libs'
  }
}
```

3.Maven Central 和 JCenter 分别是标准的 Java & Android Maven 仓库

4.通过 adb pull /data/anr/traces.txt ~destop 命令导出 trace 文件到桌面

5.Java 调用 JavaScrip

```java
mWebView.loadUrl("javascript:toast()")
```

6.JavaScript 调用 Java

a.调用与 WebView 关联的 WebSettings 实例的 setJavaScriptEnabled 为 true

b.调用 WebView 的 addJavascriptInterface 方法将应用中的 Java 对象暴露给 JavaScript

c.在 JavaScript 脚本中调用暴露出来的 Java 对象的方法

7.ClassLoader 机制

PathClassLoader：只能加载已经安装到 Android 系统中的 APK 文件

DexClassLoader：支持加载外部的 APK，Jar 或者 Dex 文件