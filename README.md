# 问题描述

使用一个上传文件的Java代码，打包成war包部署到App Service for Windows环境后，发现无法访问。报错404！

![image](https://img2024.cnblogs.com/blog/2127802/202510/2127802-20251026134651684-1099849689.png)

如果在本地启动，是正常的。

![image](https://img2024.cnblogs.com/blog/2127802/202510/2127802-20251026135053924-568860760.png)

这是什么原因呢？难道是部署时出现了错误?

# 问题解答

按照Azure App Service的部署文档，直接使用AZ CLI来部署 war 包 (部署 WAR、JAR 或 EAR 包 : [https://docs.azure.cn/zh-cn/app-service/deploy-zip?tabs=cli#-deploy-war-jar-or-ear-packages](https://github.com))

> 使用 az webapp deploy 命令将 WAR 包部署到 Tomcat 或 JBoss EAP。 为 `--src-path` 指定本地 Java 包的路径。
>
> `az webapp deploy --resource-group  --name  --src-path ./.war`

部署好之后， 在App Service的Kudu站点中，查看文件部署的文件结构：

![image](https://img2024.cnblogs.com/blog/2127802/202510/2127802-20251026141443574-2014371719.png)

文件已经部署到App Service的wwwroot目录中，脚本部署显示也是正常。但是为什么能运行呢？

继续查看kudu中的logfiles/Application/中查看到日志: contalina 启动失败，报错 “ Error WARNING: Unknown version string [5.0]. Default version will be used. ”， 非常困惑！

2025-10-26T07:31:43 PID[7732] Error NOTE: Picked up JDK\_JAVA\_OPTIONS: --add-opens=java.base/java.lang=ALL-UNNAMED --add-opens=java.base/java.lang.invoke=ALL-UNNAMED --add-opens=java.base/java.lang.reflect=ALL-UNNAMED --add-opens=java.base/java.io=ALL-UNNAMED --add-opens=java.base/java.util=ALL-UNNAMED --add-opens=java.base/java.util.concurrent=ALL-UNNAMED --add-opens=java.rmi/sun.rmi.transport=ALL-UNNAMED2025-10-26T07:31:43 PID[7732] Error Picked up \_JAVA\_OPTIONS: -javaagent:'C:\Program Files (x86)\SiteExtensions\JavaApplicationInsightsAgent\3.5.1\java\applicationinsights-agent-codeless.jar'2025-10-26T07:31:44 PID[7732] Error OpenJDK 64-Bit Server VM warning: Sharing is only supported for boot loader classes because bootstrap classpath has been appended2025-10-26T07:32:08 PID[7732] Error Oct 26, 2025 7:32:08 AM org.apache.catalina.startup.ClassLoaderFactory validateFile2025-10-26T07:32:08 PID[7732] Error WARNING: Problem with directory [D:\DWASFiles\Sites\lbwartest02\Temp\JavaFiles\c8fdd919-ff9e-41d8-ab61-e757726050cd\tomcat\lib], exists: [false], isDirectory: [false], canRead: [false]2025-10-26T07:32:08 PID[7732] Error Oct 26, 2025 7:32:08 AM org.apache.catalina.startup.ClassLoaderFactory validateFile2025-10-26T07:32:08 PID[7732] Error WARNING: Problem with directory [D:\DWASFiles\Sites\lbwartest02\Temp\JavaFiles\c8fdd919-ff9e-41d8-ab61-e757726050cd\tomcat\lib], exists: [false], isDirectory: [false], canRead: [false]2025-10-26T07:32:08 PID[7732] Error Oct 26, 2025 7:32:08 AM org.apache.catalina.startup.ClassLoaderFactory validateFile2025-10-26T07:32:08 PID[7732] Error WARNING: Problem with directory [C:\home\site\libs], exists: [false], isDirectory: [false], canRead: [false]2025-10-26T07:32:08 PID[7732] Error Oct 26, 2025 7:32:08 AM org.apache.catalina.startup.ClassLoaderFactory validateFile2025-10-26T07:32:08 PID[7732] Error WARNING: Problem with directory [C:\home\site\libs], exists: [false], isDirectory: [false], canRead: [false]2025-10-26T07:32:21 PID[7732] Error Oct 26, 2025 7:32:21 AM org.apache.catalina.startup.HostConfig deployDescriptor2025-10-26T07:32:21 PID[7732] Error WARNING: The path attribute with value [] in deployment descriptor [D:\DWASFiles\Sites\lbwartest02\Temp\JavaFiles\c8fdd919-ff9e-41d8-ab61-e757726050cd\site\wwwroot\ROOT.xml] has been ignored2025-10-26T07:32:25 PID[7732] Error Oct 26, 2025 7:32:25 AM org.apache.tomcat.util.descriptor.web.WebXml setVersion2025-10-26T07:32:25 PID[7732] Error WARNING: Unknown version string [5.0]. Default version will be used.

根据version上猜测，可能与Java 或 Tomcat的版本相关。 于是修改Tomcat的版本到10.0 之后，再次查看日志，就会发现更加详细的版本冲突的错误信息：java.lang.UnsupportedClassVersionError: org/springframework/web/SpringServletContainerInitializer has been compiled by a more recent version of the Java Runtime (class file version 61.0), this version of the Java Runtime only recognizes class file versions up to 55.0 (unable to load class [org.springframework.web.SpringServletContainerInitializer])

```
2025-10-26T07:40:44  PID[6224] Error       NOTE: Picked up JDK_JAVA_OPTIONS:  --add-opens=java.base/java.lang=ALL-UNNAMED --add-opens=java.base/java.io=ALL-UNNAMED --add-opens=java.base/java.util=ALL-UNNAMED --add-opens=java.base/java.util.concurrent=ALL-UNNAMED --add-opens=java.rmi/sun.rmi.transport=ALL-UNNAMED
2025-10-26T07:40:44  PID[6224] Error       Picked up _JAVA_OPTIONS: -javaagent:'C:\Program Files (x86)\SiteExtensions\JavaApplicationInsightsAgent\3.5.1\java\applicationinsights-agent-codeless.jar'
2025-10-26T07:40:46  PID[6224] Error       OpenJDK 64-Bit Server VM warning: Sharing is only supported for boot loader classes because bootstrap classpath has been appended
2025-10-26T07:41:15  PID[6224] Error       Oct 26, 2025 7:41:15 AM org.apache.catalina.startup.ClassLoaderFactory validateFile
2025-10-26T07:41:15  PID[6224] Error       WARNING: Problem with directory [D:\DWASFiles\Sites\lbwartest02\Temp\JavaFiles\147da13b-a28f-4b35-a90a-3800a3cae95b\tomcat\lib], exists: [false], isDirectory: [false], canRead: [false]
2025-10-26T07:41:15  PID[6224] Error       Oct 26, 2025 7:41:15 AM org.apache.catalina.startup.ClassLoaderFactory validateFile
2025-10-26T07:41:15  PID[6224] Error       WARNING: Problem with directory [D:\DWASFiles\Sites\lbwartest02\Temp\JavaFiles\147da13b-a28f-4b35-a90a-3800a3cae95b\tomcat\lib], exists: [false], isDirectory: [false], canRead: [false]
2025-10-26T07:41:16  PID[6224] Error       Oct 26, 2025 7:41:16 AM org.apache.catalina.startup.ClassLoaderFactory validateFile
2025-10-26T07:41:16  PID[6224] Error       WARNING: Problem with directory [C:\home\site\libs], exists: [false], isDirectory: [false], canRead: [false]
2025-10-26T07:41:16  PID[6224] Error       Oct 26, 2025 7:41:16 AM org.apache.catalina.startup.ClassLoaderFactory validateFile
2025-10-26T07:41:16  PID[6224] Error       WARNING: Problem with directory [C:\home\site\libs], exists: [false], isDirectory: [false], canRead: [false]
2025-10-26T07:41:32  PID[6224] Error       Oct 26, 2025 7:41:32 AM org.apache.catalina.startup.HostConfig deployDescriptor
2025-10-26T07:41:32  PID[6224] Error       WARNING: The path attribute with value [] in deployment descriptor [D:\DWASFiles\Sites\lbwartest02\Temp\JavaFiles\147da13b-a28f-4b35-a90a-3800a3cae95b\site\wwwroot\ROOT.xml] has been ignored
2025-10-26T07:41:39  PID[6224] Error       Oct 26, 2025 7:41:39 AM org.apache.catalina.startup.HostConfig deployDescriptor
2025-10-26T07:41:39  PID[6224] Error       SEVERE: Error deploying deployment descriptor [D:\DWASFiles\Sites\lbwartest02\Temp\JavaFiles\147da13b-a28f-4b35-a90a-3800a3cae95b\site\wwwroot\ROOT.xml]
2025-10-26T07:41:39  PID[6224] Error       java.lang.IllegalStateException: Error starting child
2025-10-26T07:41:39  PID[6224] Error       	at org.apache.catalina.core.ContainerBase.addChildInternal(ContainerBase.java:729)
2025-10-26T07:41:39  PID[6224] Error       	at org.apache.catalina.core.ContainerBase.addChild(ContainerBase.java:698)
2025-10-26T07:41:39  PID[6224] Error       	at org.apache.catalina.core.StandardHost.addChild(StandardHost.java:747)
2025-10-26T07:41:39  PID[6224] Error       	at org.apache.catalina.startup.HostConfig.deployDescriptor(HostConfig.java:693)
2025-10-26T07:41:39  PID[6224] Error       	at org.apache.catalina.startup.HostConfig$DeployDescriptor.run(HostConfig.java:1979)
2025-10-26T07:41:39  PID[6224] Error       	at java.base/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:515)
2025-10-26T07:41:39  PID[6224] Error       	at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
2025-10-26T07:41:39  PID[6224] Error       	at org.apache.tomcat.util.threads.InlineExecutorService.execute(InlineExecutorService.java:75)
2025-10-26T07:41:39  PID[6224] Error       	at java.base/java.util.concurrent.AbstractExecutorService.submit(AbstractExecutorService.java:118)
2025-10-26T07:41:39  PID[6224] Error       	at org.apache.catalina.startup.HostConfig.deployDescriptors(HostConfig.java:586)
2025-10-26T07:41:39  PID[6224] Error       	at org.apache.catalina.startup.HostConfig.deployApps(HostConfig.java:476)
2025-10-26T07:41:39  PID[6224] Error       	at org.apache.catalina.startup.HostConfig.start(HostConfig.java:1708)
2025-10-26T07:41:39  PID[6224] Error       	at org.apache.catalina.startup.HostConfig.lifecycleEvent(HostConfig.java:320)
2025-10-26T07:41:39  PID[6224] Error       	at org.apache.catalina.util.LifecycleBase.fireLifecycleEvent(LifecycleBase.java:123)
2025-10-26T07:41:39  PID[6224] Error       	at org.apache.catalina.util.LifecycleBase.setStateInternal(LifecycleBase.java:423)
2025-10-26T07:41:39  PID[6224] Error       	at org.apache.catalina.util.LifecycleBase.setState(LifecycleBase.java:366)
2025-10-26T07:41:39  PID[6224] Error       	at org.apache.catalina.core.ContainerBase.startInternal(ContainerBase.java:946)
2025-10-26T07:41:39  PID[6224] Error       	at org.apache.catalina.core.StandardHost.startInternal(StandardHost.java:886)
2025-10-26T07:41:39  PID[6224] Error       	at org.apache.catalina.util.LifecycleBase.start(LifecycleBase.java:183)
2025-10-26T07:41:39  PID[6224] Error       	at org.apache.catalina.core.ContainerBase$StartChild.call(ContainerBase.java:1396)
2025-10-26T07:41:39  PID[6224] Error       	at org.apache.catalina.core.ContainerBase$StartChild.call(ContainerBase.java:1386)
2025-10-26T07:41:40  PID[6224] Error       	at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
2025-10-26T07:41:40  PID[6224] Error       	at org.apache.tomcat.util.threads.InlineExecutorService.execute(InlineExecutorService.java:75)
2025-10-26T07:41:40  PID[6224] Error       	at java.base/java.util.concurrent.AbstractExecutorService.submit(AbstractExecutorService.java:140)
2025-10-26T07:41:40  PID[6224] Error       	at org.apache.catalina.core.ContainerBase.startInternal(ContainerBase.java:919)
2025-10-26T07:41:40  PID[6224] Error       	at org.apache.catalina.core.StandardEngine.startInternal(StandardEngine.java:265)
2025-10-26T07:41:40  PID[6224] Error       	at org.apache.catalina.util.LifecycleBase.start(LifecycleBase.java:183)
2025-10-26T07:41:40  PID[6224] Error       	at org.apache.catalina.core.StandardService.startInternal(StandardService.java:432)
2025-10-26T07:41:40  PID[6224] Error       	at org.apache.catalina.util.LifecycleBase.start(LifecycleBase.java:183)
2025-10-26T07:41:40  PID[6224] Error       	at org.apache.catalina.core.StandardServer.startInternal(StandardServer.java:930)
2025-10-26T07:41:40  PID[6224] Error       	at org.apache.catalina.util.LifecycleBase.start(LifecycleBase.java:183)
2025-10-26T07:41:40  PID[6224] Error       	at org.apache.catalina.startup.Catalina.start(Catalina.java:795)
2025-10-26T07:41:40  PID[6224] Error       	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
2025-10-26T07:41:40  PID[6224] Error       	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
2025-10-26T07:41:40  PID[6224] Error       	at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
2025-10-26T07:41:40  PID[6224] Error       	at java.base/java.lang.reflect.Method.invoke(Method.java:566)
2025-10-26T07:41:40  PID[6224] Error       	at org.apache.catalina.startup.Bootstrap.start(Bootstrap.java:345)
2025-10-26T07:41:40  PID[6224] Error       	at org.apache.catalina.startup.Bootstrap.main(Bootstrap.java:476)
2025-10-26T07:41:40  PID[6224] Error       Caused by: org.apache.catalina.LifecycleException: Failed to start component [StandardEngine[Catalina].StandardHost[localhost].StandardContext[]]
2025-10-26T07:41:40  PID[6224] Error       	at org.apache.catalina.util.LifecycleBase.handleSubClassException(LifecycleBase.java:440)
2025-10-26T07:41:40  PID[6224] Error       	at org.apache.catalina.util.LifecycleBase.start(LifecycleBase.java:198)
2025-10-26T07:41:40  PID[6224] Error       	at org.apache.catalina.core.ContainerBase.addChildInternal(ContainerBase.java:726)
2025-10-26T07:41:40  PID[6224] Error       	... 37 more
2025-10-26T07:41:40  PID[6224] Error       Caused by: java.lang.UnsupportedClassVersionError: org/springframework/web/SpringServletContainerInitializer has been compiled by a more recent version of the Java Runtime (class file version 61.0), this version of the Java Runtime only recognizes class file versions up to 55.0 (unable to load class [org.springframework.web.SpringServletContainerInitializer])
2025-10-26T07:41:40  PID[6224] Error       	at org.apache.catalina.loader.WebappClassLoaderBase.findClassInternal(WebappClassLoaderBase.java:2515)
2025-10-26T07:41:41  PID[6224] Error       	at org.apache.catalina.loader.WebappClassLoaderBase.findClass(WebappClassLoaderBase.java:877)
2025-10-26T07:41:42  PID[6224] Error       	at org.apache.catalina.loader.WebappClassLoaderBase.loadClass(WebappClassLoaderBase.java:1413)
2025-10-26T07:41:42  PID[6224] Error       	at org.apache.catalina.loader.WebappClassLoaderBase.loadClass(WebappClassLoaderBase.java:1257)
2025-10-26T07:41:42  PID[6224] Error       	at java.base/java.lang.Class.forName0(Native Method)
2025-10-26T07:41:42  PID[6224] Error       	at java.base/java.lang.Class.forName(Class.java:398)
2025-10-26T07:41:42  PID[6224] Error       	at org.apache.catalina.startup.WebappServiceLoader.loadServices(WebappServiceLoader.java:226)
2025-10-26T07:41:42  PID[6224] Error       	at org.apache.catalina.startup.WebappServiceLoader.load(WebappServiceLoader.java:197)
2025-10-26T07:41:42  PID[6224] Error       	at org.apache.catalina.startup.ContextConfig.processServletContainerInitializers(ContextConfig.java:1840)
2025-10-26T07:41:42  PID[6224] Error       	at org.apache.catalina.startup.ContextConfig.webConfig(ContextConfig.java:1298)
2025-10-26T07:41:42  PID[6224] Error       	at org.apache.catalina.startup.ContextConfig.configureStart(ContextConfig.java:986)
2025-10-26T07:41:42  PID[6224] Error       	at org.apache.catalina.startup.ContextConfig.lifecycleEvent(ContextConfig.java:303)
2025-10-26T07:41:42  PID[6224] Error       	at org.apache.catalina.util.LifecycleBase.fireLifecycleEvent(LifecycleBase.java:123)
2025-10-26T07:41:42  PID[6224] Error       	at org.apache.catalina.core.StandardContext.startInternal(StandardContext.java:5085)
2025-10-26T07:41:42  PID[6224] Error       	at org.apache.catalina.util.LifecycleBase.start(LifecycleBase.java:183)
2025-10-26T07:41:42  PID[6224] Error       	... 38 more
2025-10-26T07:41:42  PID[6224] Error
```

所以，根据以上错误 Java Runtime (class file version 61.0) 表示这是 **Java 17** 编译出来的类文件版本，而当前 JVM 只支持到 `55.0，`这是 **Java 11** 的类文件版本。 所以需要升级的是Java 的版本，而非 Tomcat 版本。

于是，把Java 版本调整到 17 后，再次访问接口。终于，成功了！！！

因为Java版本的问题导致访问404，从Java 11 + Tomcat 9.0 到 Java 11 + Tomcat 10.0  最后到 Java 17 + Tomcat 10.0。

![image]()

接口访问效果展示：

![image]()

# 参考资料

部署 WAR、JAR 或 EAR 包 ：[https://docs.azure.cn/zh-cn/app-service/deploy-zip?tabs=cli#-deploy-war-jar-or-ear-packages](https://github.com):[veee加速器官网](https://veee6.com)

> PS: 如果向调用 wardeploy接口，可参考：
>
> curl -X POST -u waruser:Pwd --data-binary @"demo.war" https://xxxxxx.scm.chinacloudsites.cn/api/wardeploy
