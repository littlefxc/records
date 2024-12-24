---
title: SpringBoot 添加本地 jar 文件的操作步骤
status: Done
Tags:
  - Java
---


> 转载自 [https://blog.csdn.net/qq_42046421/article/details/126528076](https://blog.csdn.net/qq_42046421/article/details/126528076)

在平时我们做项目中，需要用到jar包文件，有时候是不能从maven远程仓库拉取的，这时候就得考虑用到jar文件安装到本地maven库中，再添加依赖，今天小编分步骤给大家介绍下SpringBoot 添加本地 jar 文件的流程，一起看看吧

## 前言

有时候我们在项目中，会用到一些本地 jar 包文件，比如隔壁公司自己打包的；

此时无法从maven远程仓库拉取；

那么我们可以考虑把 jar 文件安装到本地 maven 库中，然后再添加依赖。

### 步骤

1. 添加 jar 文件到项目中

    在 resources 目录中创建一个 lib 目录，将本地 jar 放进去

2. 安装 jar 包到 maven 本地仓库

    这里我们可以利用 maven-install-plugin 插件来安装， pom.xml如下：

    ```xml
    <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-install-plugin</artifactId>
        <version>2.5.2</version>
        <executions>
            <execution>
                <id>install-foyue-common-3.2.0.jar-jar</id>
                <!-- 阶段：设定在 mvn clean 时执行安装,如果设定为 validate，那么就是在 mvn validate 时才安装 -->
                <phase>clean</phase>
                <configuration>
                    <!-- 路径：这就是刚才添加的 jar 路径 -->
                    <file>${project.basedir}/src/main/resources/lib/foyue-common-3.2.0.jar</file>
                    <!-- 属性：下面的这三个属性，就是后面我们添加依赖时的值 -->
                    <groupId>com.foyue</groupId>
                    <artifactId>foyue-common</artifactId>
                    <version>3.2.0</version>
                    <packaging>jar</packaging>
                    <generatePom>true</generatePom>
                </configuration>
                <goals>
                    <!-- 目标：安装外部的 jar 文件到 maven 本地仓库 -->
                    <goal>install-file</goal>
                </goals>
            </execution>
        </executions>
    </plugin>
    ```

    运行`mvn clean`后，会打印如下日志：

    ```log
    "C:\Program Files\Java\jdk1.8.0_321\bin\java.exe" -Dmaven.multiModuleProjectDirectory=D:\ruoyi\RuoYi-Vue-Oracle -Dmaven.home=C:\apache-maven-3.6.3 -Dclassworlds.conf=C:\apache-maven-3.6.3\bin\m2.conf "-Dmaven.ext.class.path=C:\Program Files\JetBrains\IntelliJ IDEA 2020.1\plugins\maven\lib\maven-event-listener.jar" "-javaagent:C:\Program Files\JetBrains\IntelliJ IDEA 2020.1\lib\idea_rt.jar=2054:C:\Program Files\JetBrains\IntelliJ IDEA 2020.1\bin" -Dfile.encoding=UTF-8 -classpath C:\apache-maven-3.6.3\boot\plexus-classworlds-2.6.0.jar;C:\apache-maven-3.6.3\boot\plexus-classworlds.license org.codehaus.classworlds.Launcher -Didea.version2020.1 -s C:\apache-maven-3.6.3\conf\settings.xml -DskipTests=true clean
    [INFO] Scanning for projects...
    [INFO] 
    [INFO] --------------------------< com.ruoyi:ruoyi >---------------------------
    [INFO] Building ruoyi 3.8.3
    [INFO] --------------------------------[ jar ]---------------------------------
    [INFO] 
    [INFO] --- maven-clean-plugin:3.1.0:clean (default-clean) @ ruoyi ---
    [INFO] 
    [INFO] --- maven-install-plugin:2.5.2:install-file (install-foyue-common-3.2.0.jar-jar) @ ruoyi ---
    [INFO] Installing D:\ruoyi\RuoYi-Vue-Oracle\src\main\resources\lib\foyue-common-3.2.0.jar to C:\Users\Heaven\.m2\repository\com\foyue\foyue-common\3.2.0\foyue-common-3.2.0.jar
    [INFO] Installing C:\Users\Heaven\AppData\Local\Temp\mvninstall3878861988301601324.pom to C:\Users\Heaven\.m2\repository\com\foyue\foyue-common\3.2.0\foyue-common-3.2.0.pom
    [INFO] ------------------------------------------------------------------------
    [INFO] BUILD SUCCESS
    [INFO] ------------------------------------------------------------------------
    [INFO] Total time:  0.659 s
    [INFO] Finished at: 2022-08-25T15:07:51+08:00
    [INFO] ------------------------------------------------------------------------
    ```

    重点是这一行：

    ```log
    [INFO] Installing D:\ruoyi\RuoYi-Vue-Oracle\src\main\resources\lib\foyue-common-3.2.0.jar to C:\Users\Heaven\.m2\repository\com\foyue\foyue-common\3.2.0\foyue-common-3.2.0.jar
    ```

    可以看到，将我们本地的 foyue-common-3.2.0.jar 安装到了 maven 本地仓库中。

3. 添加依赖

    ```xml
    <dependency>
        <groupId>com.foyue</groupId>
        <artifactId>foyue-common</artifactId>
        <version>3.2.0</version>
    </dependency>
    ```

    此时程序就可以正常使用 foyue-common-3.2.0.jar 包了，而且 maven 打包也会把 foyue-common-3.2.0.jar 打包进去。
