# EasyModbus4j 升级到 JDK 21 依赖与兼容性问题清单

> 整理时间: 2026-09-16  
> 目标版本: OpenJDK 21 / Tencent Kona 21  
> 涉及模块: `easymodbus4j-extension`, `easymodbus4j-commandclient`  
> 父级依赖: `com.github.zengfr.project:parent:0.0.2` -> `superparent:0.0.1` / `bom:0.0.2`

---

## 目录
- [一、项目当前状态与背景](#一项目当前状态与背景)
- [二、编译期硬性阻断问题（必须修复）](#二编译期硬性阻断问题必须修复)
  - [1. maven-compiler-plugin 插件版本过旧](#1-maven-compiler-plugin-插件版本过旧)
  - [2. lombok 1.18.6 导致 Javac 注解处理器崩溃](#2-lombok-1186-导致-javac-注解处理器崩溃)
  - [3. maven-enforcer-plugin 潜在的字节码校验错误](#3-maven-enforcer-plugin-潜在的字节码校验错误)
- [三、运行期隐患与第三方依赖安全问题（强烈建议改造）](#三运行期隐患与第三方依赖安全问题强烈建议改造)
  - [1. io.netty:netty-all:4.1.33.Final 强反射警告与性能](#1-ionettynetty-all4133final-强反射警告与性能)
  - [2. com.alibaba:fastjson:1.2.56 严重安全漏洞与反射限制](#2-comalibabafastjson1256-严重安全漏洞与反射限制)
  - [3. ch.qos.logback:logback-classic:1.2.3 安全与版本升级](#3-chqoslogbacklogback-classic123-安全与版本升级)
  - [4. 其他依赖项陈旧度排查](#4-其他依赖项陈旧度排查)
  - [5. 外部 easymodbus4j-core/codec 二进制兼容性](#5-外部-easymodbus4j-corecodec-二进制兼容性)
- [四、测试套件存在的问题](#四测试套件存在的问题)
- [五、推荐依赖与插件升级对照表](#五推荐依赖与插件升级对照表)
- [六、具体改造实施方案（POM 配置参考）](#六具体改造实施方案pom-配置参考)

---

## 一、项目当前状态与背景

1. 在删除了两个包含老旧实现与语法错误的示例模块（`easymodbus4j-example`、`easymodbus4j-example2`）后，当前项目包含以下两个核心子模块：
   - `easymodbus4j-extension`
   - `easymodbus4j-commandclient`
2. 两个模块均继承自外部 Maven 仓库中的父工程：
   - Parent: `com.github.zengfr.project:parent:0.0.2`
   - Superparent: `com.github.zengfr.project:superparent:0.0.1`
   - BOM: `com.github.zengfr.project:bom:0.0.2`
3. 根目录缺少聚合根 `pom.xml`，各个子模块需单独构建或依赖管理。

---

## 二、编译期硬性阻断问题（必须修复）

在 JDK 21 环境下直接执行 `mvn clean compile` 会立即导致构建失败，根因如下：

### 1. `maven-compiler-plugin` 插件版本过旧
* **当前版本**：继承自 `superparent:0.0.1` 中的 `3.1`（发布于 2013 年）。
* **报错现象**：
  ```text
  [ERROR] Failed to execute goal org.apache.maven.plugins:maven-compiler-plugin:3.1:compile ...
  Fatal error compiling: java.lang.ExceptionInInitializerError: 
  Unable to make field private com.sun.tools.javac.processing.JavacProcessingEnvironment$DiscoveredProcessors 
  accessible: module jdk.compiler does not "opens com.sun.tools.javac.processing" to unnamed module
  ```
* **根本原因**：
  - JDK 16+ 引入了 JEP 396/403，严格封装 JDK 内部模块（`--illegal-access=deny`）。
  - `maven-compiler-plugin:3.1` 通过旧式反射侵入 `com.sun.tools.javac.processing`，在 JDK 21 下被直接拒绝。
  - 3.1 版本也不支持现代 Java 的 `<release>21</release>` 编译参数。
* **整改措施**：
  - 在子模块的 `<build><plugins>` 中显式声明并升级为 **`3.11.0`** 或 **`3.13.0`**。
  - 配置 `<release>21</release>`（或 `<source>21</source><target>21</target>`）。

### 2. `lombok 1.18.6` 导致 Javac 注解处理器崩溃
* **当前版本**：继承自 `parent-0.0.2` 的统一依赖 `org.projectlombok:lombok:1.18.6`（`provided` 作用域）。
* **报错现象**：
  在升级编译器插件后，如果依然使用 `1.18.6`，会在进入编译任务时报同样的反射异常，或抛出字节码版本不匹配：
  `Unsupported class file major version 65` / `NoSuchFieldError: Class JCTree$JCImport does not have member field ...`。
* **根本原因**：
  - 即使当前剩余源码中没有直接编写 `@Data` 等注解，因为父 POM 将 Lombok 声明为默认依赖，Maven 会将 Lombok 放入编译类路径。
  - `javac` 检测到类路径上的 `LombokProcessor` 自动注册服务并激活它，旧版 Lombok 试图操作 JDK 21 的 AST 时发生崩溃。
* **整改措施**：
  - **方案 A（升级）**：在子模块中显式添加 `<dependency>` 覆盖版本，将 `lombok` 提升至 **`1.18.30`** 或 **`1.18.32`**。
  - **方案 B（禁用注解处理器）**：在编译器参数中显式关闭注解处理 `<compilerArgs><arg>-proc:none</arg></compilerArgs>`。

### 3. `maven-enforcer-plugin` 潜在的字节码校验错误
* **当前版本**：`1.4.1` + `extra-enforcer-rules:1.1`。
* **隐患分析**：
  - 规则中包含了 `banDuplicateClasses` 和 `requireJavaVersion`。
  - `extra-enforcer-rules:1.1` 依赖的老旧 ASM 字节码解析库在扫描到带有 JDK 21（Class Version 65）的 Multi-Release JAR 时，可能触发 `IllegalArgumentException`。
* **整改措施**：
  - 如遇校验报错，可在构建时增加参数 `-Denforcer.skip=true`，或在子模块中重写插件声明升级至 `3.4.1`。

---

## 三、运行期隐患与第三方依赖安全问题（强烈建议改造）

虽然解决上述编译阻断后项目能够生成 Jar 包，但在 JDK 21 下运行时存在以下潜在风险：

### 1. `io.netty:netty-all:4.1.33.Final` 强反射警告与性能
* **当前版本**：`4.1.33.Final`（发布于 2019 年初）。
* **隐患分析**：
  1. **Unsafe 弃用警告**：Netty 4.1.33 的 `PlatformDependent0` 依赖直接调用 `sun.misc.Unsafe` 以及直接内存 Cleaner 反射。在 JDK 21 下启动时，控制台将大量打印强封装告警（`WARNING: A terminally deprecated method in sun.misc.Unsafe has been called`）。
  2. **JDK 21 虚拟线程支持**：4.1.33 完全没有对虚拟线程（Virtual Thread）进行优化和适配，甚至某些同步阻塞锁可能导致载体线程（Carrier Thread）被 Pin 住。
* **整改建议**：
  - 升级至 **`4.1.108.Final`** 或 **`4.1.115.Final`**。
  - 如在生产环境运行，建议在 JVM 启动脚本中添加必要的模块开放选项：
    ```text
    --add-opens java.base/java.nio=ALL-UNNAMED
    --add-opens java.base/sun.nio.ch=ALL-UNNAMED
    ```

### 2. `com.alibaba:fastjson:1.2.56` 严重安全漏洞与反射限制
* **当前版本**：`1.2.56`。
* **隐患分析**：
  1. **高危安全漏洞**：Fastjson 1.2.56 存在多起公开的高危远程代码执行（RCE）CVE 漏洞（如 CVE-2022-25845 等反序列化攻击风险）。
  2. **JDK 21 反射限制**：Fastjson 1.x 依赖大量私有字段的直接反射访问，在 JDK 21 强模块封装下极易抛出 `InaccessibleObjectException`。
* **整改建议**：
  - **保守升级**：直接升级到 1.x 的最终补丁版 **`1.2.83`**。
  - **彻底升级**：替换为针对现代 Java（JDK 8~21）重写的 **`com.alibaba.fastjson2:fastjson2:2.0.47+`**。

### 3. `ch.qos.logback:logback-classic:1.2.3` 安全与版本升级
* **当前版本**：`1.2.3`。
* **隐患分析**：
  - 存在 CVE-2021-42550 等配置反序列化相关安全风险。
* **整改建议**：
  - 保持 1.2 分支可升级至 **`1.2.13`**。
  - 若整个项目完全切换至现代 JDK，可升级至 **`1.4.14`** 或 **`1.5.x`**（支持 JDK 11+）。

### 4. 其他依赖项陈旧度排查
* **`com.google.guava:27.0.1-jre`**：可升级至 **`32.1.3-jre`** 或 **`33.0.0-jre`**，避免旧版中空的 `listenablefuture:9999.0` 兼容包干扰。
* **`org.apache.httpcomponents:httpclient:4.5.3`**：可平滑升级至 **`4.5.14`**。
* **`commons-lang3:3.8.1`** 与 **`commons-collections4:4.2`**：当前兼容，可按需升级至 `3.14.0` / `4.4`。

### 5. 外部 `easymodbus4j-core/codec` 二进制兼容性
* **组件**：`com.github.zengfr:easymodbus4j:0.0.5`（包含 core 0.0.5、codec 0.0.5）。
* **分析**：
  - 该核心包以 Class Version 52（Java 8）预先编译。
  - JVM 具有严格的向下兼容保证，JDK 21 可以完美加载 Java 8 编译的 Jar 包，且其公开的 API（编解码、Frame、Handler 等）在 JDK 21 下能够正常执行。

---

## 四、测试套件存在的问题

在模块 `easymodbus4j-commandclient` 中的测试类 [`ClientTest.java`](file:///D:/Development/project/easymodbus4j/easymodbus4j-commandclient/src/test/java/ClientTest.java#L20)：
* **代码缺陷**：
  ```java
  @Test
  public void test() throws Exception {
      for (int i = 0; i < Integer.MAX_VALUE; i++) {
          System.out.println(i);
          Thread.sleep(111);
          ...
      }
  }
  ```
* **影响**：如果不加 `-DskipTests` 执行 `mvn test` 或 `mvn package`，该单元测试将进入近乎无限循环（达 21 亿次循环），导致持续阻塞卡死。
* **建议**：修改循环次数为固定测试边界（例如 `i < 5`）或添加 `@Ignore`。

---

## 五、推荐依赖与插件升级对照表

| 分类 | 构件标识 (GroupId:ArtifactId) | 当前继承版本 | 推荐升级版本 | 改造优先级 |
| :--- | :--- | :---: | :---: | :---: |
| **构建插件** | `org.apache.maven.plugins:maven-compiler-plugin` | 3.1 | **3.13.0** | 🔴 **必须改动** |
| **编译依赖** | `org.projectlombok:lombok` | 1.18.6 | **1.18.30** / **1.18.32** | 🔴 **必须改动** |
| **网络核心** | `io.netty:netty-all` | 4.1.33.Final | **4.1.108.Final** | 🟡 **强烈建议** |
| **数据解析** | `com.alibaba:fastjson` | 1.2.56 | **1.2.83** / **fastjson2** | 🟡 **强烈建议** |
| **日志框架** | `ch.qos.logback:logback-classic` | 1.2.3 | **1.2.13** / **1.4.14** | 🟢 **推荐改动** |
| **工具类库** | `com.google.guava:guava` | 27.0.1-jre | **32.1.3-jre** | 🟢 **推荐改动** |
| **HTTP组件** | `org.apache.httpcomponents:httpclient` | 4.5.3 | **4.5.14** | 🟢 **推荐改动** |

---

## 六、具体改造实施方案（POM 配置参考）

### 1. `easymodbus4j-extension/pom.xml` 配置模版
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>com.github.zengfr</groupId>
	<artifactId>easymodbus4j-extension</artifactId>
	<version>0.0.5</version>

	<properties>
		<skipAssembly>true</skipAssembly>
		<!-- 升级为 Java 21 -->
		<jdk.version>21</jdk.version>
		<maven.compiler.source>21</maven.compiler.source>
		<maven.compiler.target>21</maven.compiler.target>
		<maven.compiler.release>21</maven.compiler.release>
		<!-- 统一版本属性 -->
		<lombok.version>1.18.32</lombok.version>
		<netty.version>4.1.108.Final</netty.version>
		<fastjson.version>1.2.83</fastjson.version>
	</properties>

	<parent>
		<groupId>com.github.zengfr.project</groupId>
		<artifactId>parent</artifactId>
		<version>0.0.2</version>
		<relativePath>../parent/pom.xml</relativePath>
	</parent>

	<dependencies>
		<!-- 覆盖父 POM 的老旧 Lombok，解除编译期 Javac 崩溃 -->
		<dependency>
			<groupId>org.projectlombok</groupId>
			<artifactId>lombok</artifactId>
			<version>${lombok.version}</version>
			<scope>provided</scope>
		</dependency>

		<dependency>
			<artifactId>easymodbus4j</artifactId>
			<groupId>com.github.zengfr</groupId>
			<version>0.0.5</version>
		</dependency>

		<!-- 建议覆盖 Netty 版本消除 Unsafe 反射警告 -->
		<dependency>
			<groupId>io.netty</groupId>
			<artifactId>netty-all</artifactId>
			<version>${netty.version}</version>
		</dependency>
	</dependencies>

	<build>
		<plugins>
			<!-- 升级支持 JDK 21 的编译器插件 -->
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-compiler-plugin</artifactId>
				<version>3.13.0</version>
				<configuration>
					<release>21</release>
				</configuration>
			</plugin>
		</plugins>
	</build>
</project>
```

### 2. `easymodbus4j-commandclient/pom.xml` 配置模版
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.github.zengfr</groupId>
    <artifactId>easymodbus4j-commandclient</artifactId>
    <version>0.0.5</version>
    <name>easymodbus4j-commandclient</name>

    <properties>
        <skipAssembly>true</skipAssembly>
        <jdk.version>21</jdk.version>
        <maven.compiler.source>21</maven.compiler.source>
        <maven.compiler.target>21</maven.compiler.target>
        <maven.compiler.release>21</maven.compiler.release>
        <lombok.version>1.18.32</lombok.version>
        <netty.version>4.1.108.Final</netty.version>
    </properties>

    <parent>
        <groupId>com.github.zengfr.project</groupId>
        <artifactId>parent</artifactId>
        <version>0.0.2</version>
        <relativePath>../parent/pom.xml</relativePath>
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>${lombok.version}</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.netty</groupId>
            <artifactId>netty-all</artifactId>
            <version>${netty.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.httpcomponents</groupId>
            <artifactId>httpclient</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>commons-logging</groupId>
                    <artifactId>commons-logging</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>jcl-over-slf4j</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.13.0</version>
                <configuration>
                    <release>21</release>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

### 3. 可选：在项目根目录创建聚合 `pom.xml`（提升构建体验）
若希望在项目根目录下直接使用一条命令同时构建所有子模块，可在项目根目录新增一个简易聚合 POM：
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.github.zengfr</groupId>
    <artifactId>easymodbus4j-root</artifactId>
    <version>0.0.5</version>
    <packaging>pom</packaging>

    <modules>
        <module>easymodbus4j-extension</module>
        <module>easymodbus4j-commandclient</module>
    </modules>
</project>
```
新增后即可在根目录下直接执行：
```powershell
mvn clean package -DskipTests
```
