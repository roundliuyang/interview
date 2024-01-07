# Maven



## **Maven 规约是什么**



- `/src/main/java/` ：Java 源码。
- `/src/main/resource` ：Java 配置文件，资源文件。
- `/src/test/java/` ：Java 测试代码。
- `/src/test/resource` ：Java 测试配置文件，资源文件。
- `/target` ：文件编译过程中生成的 `.class` 文件、jar、war 等等。
- `pom.xml` ：配置文件

Maven 要负责项目的自动化构建，以编译为例，Maven 要想自动进行编译，那么它必须知道 Java 的源文件保存在哪里，这样约定之后，不用我们手动指定位置，Maven 能知道位置，从而帮我们完成自动编译。

遵循**“约定>>>配置>>>编码”**。即能进行配置的不要去编码指定，能事先约定规则的不要去进行配置。这样既减轻了劳动力，也能防止出错。



## Maven 常用命令



- `mvn archetype：create` ：创建 Maven 项目。
- `mvn compile` ：编译源代码。
- `mvn deploy` ：发布项目。
- `mvn test-compile` ：编译测试源代码。
- `mvn test` ：运行应用程序中的单元测试。
- `mvn site` ：生成项目相关信息的网站。
- `mvn clean` ：清除项目目录中的生成结果。
- `mvn package` ：根据项目生成的 jar/war 等。
- `mvn install` ：在本地 Repository 中安装 jar 。
- `mvn eclipse:eclipse` ：生成 Eclipse 项目文件。
- `mvn jetty:run` 启动 Jetty 服务。
- `mvn tomcat:run` ：启动 Tomcat 服务。
- `mvn clean package -Dmaven.test.skip=true` ：清除以前的包后重新打包，跳过测试类。



用到最多的命令

- `mvn eclipse:clean` ：清除 Project 中以前的编译的东西，重新再来。
- `mvn eclipse:eclipse` ：开始编译 Maven 的 Project 。
- `mvn clean package` ：清除以前的包后重新打包。





## Maven 有哪些优点和缺点



**优点**

- 简化了项目依赖管理。
- 易于上手，对于新手可能一个 `mvn clean package` 命令就可能满足我们的工作。
- 便于与持续集成工具(Jenkins)整合。
- 便于项目升级，无论是项目本身升级还是项目使用的依赖升级。
- 有助于多模块项目的开发，一个模块开发好后，发布到仓库，依赖该模块时可以直接从仓库更新，而不用自己去编译。
- Maven 有很多插件，便于功能扩展，比如生产站点，自动发布版本等。



**缺点**

- Maven 是一个庞大的构建系统，学习难度大。
- ...





##  **LASTEST、RELEASE、SNAPSHOT 的区别**



- LASTEST ：是指某个特定构件最新的发布版或者快照版(SNAPSHOT)，最近被部署到某个特定仓库的构件。
- RELEASE ：是指仓库中最后的一个非快照版本。
- SNAPSHOT ：泛指。如果不 SNAPSHOT ，如果名字不变，本地有了不会从远程拉。如果每次更新都改名字，其他用的人也都改名字，太蛋疼了。