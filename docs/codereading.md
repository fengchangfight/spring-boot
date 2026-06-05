# Spring Boot 源码高层总结

> 基于源码仓库 `spring-projects/spring-boot`（主分支）分析。

---

## 一、项目总览

Spring Boot 是一个**基于 Spring Framework 的应用快速启动框架**，核心理念是"约定优于配置"（Convention over Configuration）和"自动配置"（Auto-Configuration）。它让开发者可以用极少的配置快速创建生产级 Spring 应用。

### 项目规模

- **构建系统**：Gradle（多模块项目，根项目名 `spring-boot-build`）
- **模块总数**：约 500+ 个子模块（含 core、module、starter、smoke-test 等）
- **总计子项目**：核心 ~12 个 + 功能模块 ~110 个 + starter ~130 个 + 测试 ~120 个

---

## 二、顶层模块结构

```
spring-boot/
├── core/                          # 核心引擎（spring-boot, spring-boot-autoconfigure）
├── module/                        # 功能模块（按技术栈拆分的自动配置模块，~110 个）
├── starter/                       # Starter POM（便捷依赖聚合，~130 个）
├── loader/                        # 类加载器（spring-boot-loader, loader-tools）
├── platform/                      # 版本平台（BOM: spring-boot-dependencies）
├── build-plugin/                  # 构建插件（Maven/Gradle/Ant）
├── buildpack/                     # Buildpack 支持
├── cli/                           # Spring Boot CLI
├── configuration-metadata/        # 配置元数据处理器
├── buildSrc/                      # Gradle 构建约定插件
├── test-support/                  # 测试支持库
├── integration-test/              # 集成测试
├── system-test/                   # 系统测试（部署/镜像）
├── smoke-test/                    # 冒烟测试（约 100+ 场景）
├── documentation/                 # 文档生成
└── docs/                          # 项目文档
```

---

## 三、核心模块详解 (`core/`)

### 3.1 `spring-boot` — 核心引擎

**路径**：`core/spring-boot/src/main/java/org/springframework/boot/`

这是 Spring Boot 的**心脏**，包结构如下：

| 包 | 职责 |
|---|---|
| `boot`（根包） | `SpringApplication` 启动入口、`WebApplicationType` 类型检测、`SpringBootConfiguration` 注解 |
| `boot.admin` | `SpringApplicationAdminMXBean` — 通过 JMX 管理应用生命周期 |
| `boot.ansi` | ANSI 颜色输出支持 |
| `boot.availability` | 可用性状态（`ApplicationAvailability`）— Liveness/Readiness |
| `boot.bootstrap` | `BootstrapRegistry` / `BootstrapContext` — 启动早期的注册中心 |
| `boot.builder` | `SpringApplicationBuilder` — 流式 API 构建父子上下文 |
| `boot.cloud` | `CloudPlatform` — 云平台检测（K8s/Cloud Foundry） |
| `boot.context` | **应用上下文核心**（详见 3.2） |
| `boot.convert` | 类型转换服务（`ApplicationConversionService`） |
| `boot.diagnostics` | 启动失败诊断（`FailureAnalyzer` 体系） |
| `boot.env` | 环境与属性源（`EnvironmentPostProcessor`, `PropertySourceLoader`） |
| `boot.info` | `BuildInfo` / `GitInfo` / `JavaInfo` — `/actuator/info` 数据源 |
| `boot.io` | I/O 工具 |
| `boot.json` | JSON 解析抽象（Jackson/Gson/Jsonb） |
| `boot.logging` | **日志系统抽象**（Logback/Log4j2/Java Util Logging） |
| `boot.origin` | 属性来源追踪（`PropertySourceOrigin`） |
| `boot.retry` | 重试模板 |
| `boot.ssl` | SSL/TLS 配置（PEM/JKS 证书加载） |
| `boot.system` | 系统信息（`ApplicationHome`, `JavaVersion`, `ApplicationPid`） |
| `boot.task` | `TaskExecutorBuilder` / `TaskSchedulerBuilder` |
| `boot.thread` | 线程工具（虚拟线程支持） |
| `boot.util` | 通用工具 |
| `boot.validation` | 校验支持（Bean Validation 集成） |
| `boot.web` | **Web 支持**（Servlet/Reactive 上下文、错误处理、嵌入式容器） |

### 3.2 `spring-boot/context` — 应用上下文核心

| 子包 | 关键类 | 职责 |
|------|--------|------|
| `context/annotation` | `ImportCandidates` | 从 `META-INF/spring/*.imports` 加载配置候选 |
| `context/config` | `ConfigDataEnvironmentPostProcessor` | 配置数据加载（application.yml/properties） |
| `context/event` | `EventPublishingRunListener`, `SpringApplicationEvent` | 启动事件桥接 |
| `context/logging` | `LoggingApplicationContextInitializer` | 日志系统初始化 |
| `context/metrics` | `MetricsApplicationListener` | 启动度量采集 |
| `context/properties` | `@ConfigurationProperties`, `Binder`, `ConfigurationPropertySources` | 属性绑定核心 |
| `context/properties/bind` | `Binder`, `Bindable`, `BindHandler` | 类型安全的属性绑定 API |
| `context/properties/source` | `ConfigurationPropertySources`, `SpringConfigurationPropertySource` | 属性源适配 |
| `context/properties/bind/validation` | `ValidationBindHandler` | 属性绑定时的 Bean Validation |

### 3.3 `spring-boot-autoconfigure` — 自动配置引擎

**路径**：`core/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/`

这是 Spring Boot 最**标志性的模块**：

| 核心类 | 职责 |
|--------|------|
| `AutoConfigurationImportSelector` | **自动配置选择器**：实现 `DeferredImportSelector`，延迟加载 |
| `EnableAutoConfiguration` | 入口注解，`@Import(AutoConfigurationImportSelector.class)` |
| `SpringBootApplication` | 组合注解 = `@SpringBootConfiguration` + `@EnableAutoConfiguration` + `@ComponentScan` |
| `AutoConfiguration` | 标记注解（替代 `@Configuration`，标记自动配置类） |
| `AutoConfigurationSorter` | 排序器（处理 `@AutoConfigureOrder` / `@After` / `@Before`） |

---

## 四、功能模块 (`module/`) 与 Starter (`starter/`)

### 4.1 模块化自动配置（Spring Boot 3.x 新架构）

Spring Boot 3.x 将原来的大单体 `spring-boot-autoconfigure` 拆分成约 **110 个独立模块**：

```
module/
├── spring-boot-webmvc           ← Spring MVC
├── spring-boot-webflux          ← WebFlux 响应式
├── spring-boot-data-jpa         ← JPA
├── spring-boot-data-mongodb     ← MongoDB
├── spring-boot-data-redis       ← Redis
├── spring-boot-security         ← Spring Security
├── spring-boot-actuator         ← Actuator 监控
├── spring-boot-kafka            ← Kafka 消息
├── spring-boot-batch            ← Batch 批处理
├── spring-boot-graphql          ← GraphQL
├── spring-boot-grpc-server      ← gRPC Server
├── spring-boot-jetty            ← Jetty 嵌入式容器
├── spring-boot-tomcat           ← Tomcat 嵌入式容器
├── spring-boot-netty            ← Netty 嵌入式容器
├── spring-boot-micrometer-*     ← Micrometer 可观测性
├── spring-boot-opentelemetry    ← OpenTelemetry
├── spring-boot-devtools         ← 开发工具（热重启）
└── ...                          ← 共 110+ 个模块
```

**设计优势**：按需加载、清晰边界、`autoconfigure-classic` 向后兼容。

### 4.2 Starter 体系（~130 个）

Starter 是**纯 POM 依赖聚合**，零代码：

```
starter/
├── spring-boot-starter              ← 核心（日志 + autoconfigure）
├── spring-boot-starter-web          ← Web（webmvc + tomcat）
├── spring-boot-starter-webflux      ← 响应式 Web
├── spring-boot-starter-data-jpa     ← JPA（含 Hibernate）
├── spring-boot-starter-security     ← Security
├── spring-boot-starter-test         ← 测试（JUnit5 + Mockito + AssertJ）
├── spring-boot-starter-actuator     ← 运维监控
├── spring-boot-starter-parent       ← 父 POM（版本管理）
└── ...
```

---

## 五、核心功能清单（Features）

### 六大核心能力

| # | 能力 | 核心类 | 说明 |
|---|------|--------|------|
| 1 | **自动配置** | `AutoConfigurationImportSelector` | classpath 探测 → 自动装配 Bean |
| 2 | **条件装配** | `@ConditionalOn*` 系列 | 精细控制 Bean 创建（约 25 个条件注解） |
| 3 | **外部化配置** | `@ConfigurationProperties` + `Binder` | 类型安全属性绑定（YAML/Properties/环境变量） |
| 4 | **嵌入式容器** | `WebServerFactory` 体系 | 内嵌 Tomcat/Jetty/Undertow/Netty |
| 5 | **Actuator** | `spring-boot-actuator` | 健康检查、指标、审计、HTTP 跟踪 |
| 6 | **可执行 JAR** | `spring-boot-loader` | Fat JAR 打包，`java -jar` 直接启动 |

### 其他亮点

- **失败分析器**（`FailureAnalyzer`）：启动失败时给出人性化诊断建议
- **DevTools**：热重启、LiveReload、远程调试
- **AOT/Native**：Spring AOT 引擎 → GraalVM Native Image 编译
- **Docker Compose**：`spring-boot-docker-compose` 自动管理服务依赖
- **SSL 自动配置**：PEM/JKS 证书自动加载与热更新
- **虚拟线程**：`spring.threads.virtual.enabled=true`
- **结构化日志**：`logging.structured.format=ecs`
- **gRPC 支持**：`spring-boot-grpc-server/client`
- **可观测性**：Micrometer Metrics + Tracing + OpenTelemetry
- **Checkpoint/Restore**：CRaC（Coordinated Restore at Checkpoint）支持

---

## 六、核心架构设计

### 6.1 启动生命周期（`SpringApplication.run()` 主流程）

```
main() 入口
  │
  ├─ [1] BootstrapContext 初始化
  │     └─ BootstrapRegistryInitializer（SPI）→ 注册启动早期对象
  │
  ├─ [2] SpringApplicationRunListeners.starting()
  │     └─ EventPublishingRunListener → 发布 ApplicationStartingEvent
  │
  ├─ [3] 准备 Environment
  │     ├─ 创建/配置 Environment
  │     ├─ ConfigurationPropertySources.attach(environment)
  │     ├─ EnvironmentPostProcessor（SPI，加载 application.yml 等配置）
  │     └─ 绑定 spring.main.* 属性到 SpringApplication
  │
  ├─ [4] 创建 ApplicationContext
  │     └─ ApplicationContextFactory.create(WebApplicationType)
  │         ├─ SERVLET → AnnotationConfigServletWebServerApplicationContext
  │         ├─ REACTIVE → AnnotationConfigReactiveWebServerApplicationContext
  │         └─ NONE    → AnnotationConfigApplicationContext
  │
  ├─ [5] 准备 Context
  │     ├─ ApplicationContextInitializer 回调（SPI）
  │     ├─ 加载 sources（@Configuration 类）
  │     └─ SpringApplicationRunListeners.contextLoaded()
  │
  ├─ [6] Refresh Context（Spring Framework 核心流程）
  │     ├─ BeanFactoryPostProcessor 执行
  │     ├─ ConfigurationClassPostProcessor 处理 @Configuration + @Bean + @ComponentScan
  │     ├─ DeferredImportSelector 执行 ← AutoConfigurationImportSelector 在此生效
  │     │     └─ 加载 *.imports → 过滤 @Conditional → 注册自动配置 Bean
  │     └─ 实例化所有单例 Bean
  │
  ├─ [7] 启动后回调
  │     ├─ CommandLineRunner / ApplicationRunner 执行
  │     └─ SpringApplicationRunListeners.started() / .ready()
  │
  └─ [兜底] 异常处理
        └─ FailureAnalyzers 诊断 → 人性化错误报告
```

**生命周期阶段**：`starting` → `environmentPrepared` → `contextPrepared` → `contextLoaded` → `refresh` → `started` → `ready` → (可选) `failed`

### 6.2 自动配置完整链路

```
@SpringBootApplication
  ├─ @SpringBootConfiguration（= @Configuration）
  ├─ @EnableAutoConfiguration
  │     └─ @Import(AutoConfigurationImportSelector.class)
  │           │
  │           ├─ 实现 DeferredImportSelector（延迟到用户 Bean 注册后执行）
  │           │
  │           ├─ getCandidateConfigurations()
  │           │     └─ ImportCandidates.load(AutoConfiguration.class)
  │           │           └─ 读取 META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
  │           │
  │           ├─ 去重 + 排除
  │           │     ├─ spring.autoconfigure.exclude 属性
  │           │     └─ @EnableAutoConfiguration.exclude / .excludeName
  │           │
  │           ├─ 条件过滤（ConfigurationClassFilter）
  │           │     ├─ OnClassCondition → class 是否存在
  │           │     ├─ OnBeanCondition → Bean 是否存在
  │           │     └─ OnPropertyCondition → 配置属性值是否匹配
  │           │
  │           ├─ 排序
  │           │     └─ @AutoConfigureOrder / @AutoConfigureAfter / @AutoConfigureBefore
  │           │
  │           └─ 返回最终要导入的配置类列表 → 注册为 Bean
  │
  └─ @ComponentScan（扫描用户定义的 @Component/@Service/@Repository 等）
```

### 6.3 条件注解体系（`autoconfigure/condition/`）

Spring Boot 自动配置的**智能程度**来源于条件注解，约 25 个：

| 条件注解 | 对应 Condition 类 | 匹配逻辑 |
|---------|-------------------|---------|
| `@ConditionalOnClass` | `OnClassCondition` | 指定类在 classpath 中存在 |
| `@ConditionalOnMissingClass` | `OnClassCondition` | 指定类在 classpath 中不存在 |
| `@ConditionalOnBean` | `OnBeanCondition` | 容器中存在指定 Bean |
| `@ConditionalOnMissingBean` | `OnBeanCondition` | 容器中不存在指定 Bean |
| `@ConditionalOnProperty` | `OnPropertyCondition` | 配置属性存在且值匹配 |
| `@ConditionalOnResource` | `OnResourceCondition` | 指定资源文件存在 |
| `@ConditionalOnWebApplication` | `OnWebApplicationCondition` | 当前是 Web 应用 |
| `@ConditionalOnExpression` | `OnExpressionCondition` | SpEL 表达式为 true |
| `@ConditionalOnJava` | `OnJavaCondition` | Java 版本匹配 |
| `@ConditionalOnCloudPlatform` | `OnCloudPlatformCondition` | 运行在指定云平台 |
| `@ConditionalOnSingleCandidate` | `OnBeanCondition` | 容器中存在唯一的 Bean |
| `@ConditionalOnWarDeployment` | `OnWarDeploymentCondition` | 以 WAR 方式部署 |
| `@ConditionalOnThreading` | `OnThreadingCondition` | 线程模型（PLATFORM/VIRTUAL） |
| `@ConditionalOnCheckpointRestore` | — | CRaC 检查点恢复模式 |

> **性能优化**：`OnClassCondition` 在自动配置数量 >1 且多核时，用**后台线程并行评估** class 存在性。

---

## 七、设计亮点与模式

### 7.1 SPI 扩展点

Spring Boot 通过 `META-INF/spring/*.imports` 文件实现可插拔扩展：

| SPI 接口 | 触发时机 | 用途 |
|---------|---------|------|
| `ApplicationContextInitializer` | Context 创建后、Refresh 前 | 自定义 Context 初始化 |
| `ApplicationListener` | 各类 Spring 事件 | 事件监听 |
| `SpringApplicationRunListener` | 启动各阶段 | 启动生命周期扩展 |
| `EnvironmentPostProcessor` | Environment 准备后 | 添加/修改属性源 |
| `FailureAnalyzer` | 启动失败 | 诊断失败原因 |
| `BootstrapRegistryInitializer` | 启动最早阶段 | 注册启动期对象 |
| `AutoConfigurationImportFilter` | 自动配置导入 | 快速过滤配置候选 |
| `AutoConfigurationImportListener` | 导入事件 | 监听配置导入 |

### 7.2 设计模式

| 模式 | 应用场景 | 核心类 |
|------|---------|--------|
| **模板方法** | 启动流程 | `SpringApplication.run()` |
| **策略模式** | Context 创建 | `ApplicationContextFactory` |
| **观察者模式** | 启动事件 | `SpringApplicationRunListener` + Spring Events |
| **责任链** | 失败分析 | `FailureAnalyzers` |
| **构建器模式** | 上下文构建 | `SpringApplicationBuilder` |
| **工厂模式** | Web 容器创建 | `WebServerFactory` 体系 |
| **导入选择器** | 延迟自动配置 | `DeferredImportSelector` |

### 7.3 关键设计决策

1. **DeferredImportSelector**：自动配置延迟到用户 Bean 之后注册，用户配置优先
2. **模块化拆分（3.x）**：巨石 autoconfigure → 110+ 微模块，按需加载
3. **`.imports` 替代 `spring.factories`**：按注解而非按接口发现，2.7 引入 / 3.0 切换，性能更优
4. **Type-safe Binding**：`Binder` API 支持宽松绑定、构造器绑定、嵌套对象、集合/Map
5. **并行条件评估**：`OnClassCondition` 多核环境下多线程并行评估 class 存在性

---

## 八、源码阅读入口速查表

| 你想理解... | 从这些文件开始（路径相对于项目根） |
|-------------|---------------------------------|
| **应用如何启动** | `core/spring-boot/.../boot/SpringApplication.java` → `run()` 方法 |
| **自动配置如何发现** | `core/spring-boot-autoconfigure/.../autoconfigure/AutoConfigurationImportSelector.java` → `getAutoConfigurationEntry()` |
| **条件注解如何工作** | `core/spring-boot-autoconfigure/.../condition/OnClassCondition.java` |
| **属性如何绑定** | `core/spring-boot/.../context/properties/bind/Binder.java` |
| **Web 服务器如何启动** | `core/spring-boot/.../web/servlet/WebServerFactory` 系列 |
| **失败分析如何工作** | `core/spring-boot/.../diagnostics/analyzer/` 包 |
| **日志系统如何配置** | `core/spring-boot/.../logging/LoggingSystem.java` |
| **Actuator 端点机制** | `module/spring-boot-actuator/` → `Endpoint` 抽象 |
| **可执行 JAR 原理** | `loader/spring-boot-loader/.../loader/` 包 |
| **条件注解全览** | `core/spring-boot-autoconfigure/.../condition/*.java`（25 个） |
| **启动事件机制** | `core/spring-boot/.../context/event/EventPublishingRunListener.java` |

---

## 九、关键文件路径汇总

```
完整路径前缀：.../src/main/java/org/springframework/boot/

【核心引擎 spring-boot】
  SpringApplication.java                            — 启动入口，run() 生命周期
  WebApplicationType.java                           — Web 类型检测（NONE/SERVLET/REACTIVE）
  ApplicationContextFactory.java                    — Context 创建策略接口
  SpringApplicationRunListener.java                 — 启动生命周期 SPI
  SpringBootConfiguration.java                      — @SpringBootConfiguration 注解

【属性绑定】
  context/properties/ConfigurationProperties.java   — @ConfigurationProperties 注解
  context/properties/bind/Binder.java               — 属性绑定引擎
  context/properties/source/ConfigurationPropertySources.java — 属性源适配

【自动配置  spring-boot-autoconfigure】
  autoconfigure/AutoConfigurationImportSelector.java — 自动配置选择器
  autoconfigure/EnableAutoConfiguration.java        — @EnableAutoConfiguration 注解
  autoconfigure/SpringBootApplication.java          — @SpringBootApplication 组合注解
  autoconfigure/AutoConfiguration.java              — 自动配置标记注解
  autoconfigure/condition/OnClassCondition.java     — class 存在性条件
  autoconfigure/condition/OnBeanCondition.java      — Bean 存在性条件
  autoconfigure/condition/OnPropertyCondition.java  — 属性值条件
  context/annotation/ImportCandidates.java          — *.imports 文件加载

【日志】
  logging/LoggingSystem.java                        — 日志系统抽象
  logging/logback/LogbackLoggingSystem.java         — Logback 实现
  logging/log4j2/Log4J2LoggingSystem.java           — Log4j2 实现

【Web 支持】
  web/servlet/WebServerFactory.java                 — 嵌入式容器工厂
  web/context/ServletWebServerApplicationContext.java — Servlet Web Context
  web/context/reactive/ReactiveWebServerApplicationContext.java — Reactive Web Context
```


---

## 十、SpringApplication 类深度解析

> **文件**：`core/spring-boot/src/main/java/org/springframework/boot/SpringApplication.java`
> **约 1400 行**，是 Spring Boot 最核心的"启动引导类"。

### 10.1 类的定位

`SpringApplication` 是一个 **从 Java main 方法引导和启动 Spring 应用** 的类。它不是一个 Bean，而是在 Spring IoC 容器创建**之前**运行的"引导器"。它负责：

1. 推断应用类型（Servlet / Reactive / None）
2. 创建 `Environment`（属性源、Profile）
3. 创建 `ApplicationContext`
4. 触发 `SpringApplicationRunListener` 生命周期回调
5. 加载用户配置 → Refresh IoC 容器

### 10.2 字段一览（约 25 个状态字段）

| 字段 | 类型 | 作用 |
|------|------|------|
| `primarySources` | `Set<Class<?>>` | 主配置类（构造函数传入） |
| `mainApplicationClass` | `Class<?>` | 通过 StackWalker 推断的 main 方法所在类 |
| `banner` | `Banner` | 自定义 Banner |
| `resourceLoader` | `ResourceLoader` | 资源加载器 |
| `beanNameGenerator` | `BeanNameGenerator` | Bean 命名策略 |
| `environment` | `ConfigurableEnvironment` | 预配置的 Environment（可选） |
| `headless` | `boolean` | `java.awt.headless` 系统属性（默认 true） |
| `addCommandLineProperties` | `boolean` | 是否添加命令行参数到 Environment |
| `addConversionService` | `boolean` | 是否添加转换服务 |
| `initializers` | `List<ApplicationContextInitializer<?>>` | Context 初始化器（SPI + 手动） |
| `listeners` | `List<ApplicationListener<?>>` | 事件监听器（SPI + 手动） |
| `bootstrapRegistryInitializers` | `List<BootstrapRegistryInitializer>` | 启动早期注册中心初始化器（SPI） |
| `defaultProperties` | `Map<String, Object>` | 默认属性 |
| `additionalProfiles` | `Set<String>` | 额外激活的 Profile |
| `applicationContextFactory` | `ApplicationContextFactory` | Context 创建策略（默认根据 Web 类型） |
| `applicationStartup` | `ApplicationStartup` | 启动度量采集器 |
| `properties` | `ApplicationProperties` | **聚合配置对象**（绑定 spring.main.*） |
| `environmentPrefix` | `String` | 环境变量前缀 |
| `isCustomEnvironment` | `boolean` | 是否用户自定义了 Environment |

### 10.3 构造函数：启动前的 5 步准备

```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    this.properties.setWebApplicationType(WebApplicationType.deduce());     // ①
    this.bootstrapRegistryInitializers = new ArrayList<>(
        getSpringFactoriesInstances(BootstrapRegistryInitializer.class));  // ②
    setInitializers(getSpringFactoriesInstances(ApplicationContextInitializer.class)); // ③
    setListeners(getSpringFactoriesInstances(ApplicationListener.class));  // ④
    this.mainApplicationClass = deduceMainApplicationClass();              // ⑤
}
```

① **推断 Web 类型**：classpath 有 `DispatcherServlet` → SERVLET；有 `DispatcherHandler` → REACTIVE  
② **加载 BootstrapRegistry SPI**：最早期，Environment 创建前就能注册对象  
③ **加载 ApplicationContextInitializer SPI**：Context 创建后的回调  
④ **加载 ApplicationListener SPI**：Spring 事件监听器  
⑤ **推断 mainClass**：`StackWalker` 从调用栈中找 `main` 方法所在类

### 10.4 `run()` 方法：7 阶段启动流水线

```
run(String... args)
 │
 ├─ 阶段 0: 前置
 │   ├─ Startup.create()            — 计时器（支持 CRaC checkpoint restore）
 │   ├─ createBootstrapContext()    — BootstrapRegistry 回调
 │   ├─ configureHeadlessProperty()
 │   └─ getRunListeners(args)       — SPI 加载 SpringApplicationRunListener
 │       └─ + 检查 ThreadLocal<SpringApplicationHook>（增强/测试钩子）
 │
 ├─ 阶段 1: listeners.starting(bootstrapContext, mainApplicationClass)
 │   └─ → ApplicationStartingEvent
 │
 ├─ 阶段 2: prepareEnvironment()
 │   ├─ getOrCreateEnvironment()
 │   ├─ configureEnvironment()      — 模板方法：属性源 + Profile
 │   ├─ ConfigurationPropertySources.attach(environment)
 │   ├─ listeners.environmentPrepared() → ApplicationEnvironmentPreparedEvent
 │   └─ bindToSpringApplication()   — 绑定 spring.main.* → ApplicationProperties
 │
 ├─ 阶段 3: 创建/准备 Context
 │   ├─ printBanner(environment)
 │   ├─ createApplicationContext()  — ApplicationContextFactory 策略
 │   └─ prepareContext()
 │       ├─ context.setEnvironment(environment)
 │       ├─ postProcessApplicationContext()
 │       ├─ AOT initializer 注入（Native Image 支持）
 │       ├─ applyInitializers(context)         — SPI 回调
 │       ├─ listeners.contextPrepared()        → ApplicationContextInitializedEvent
 │       ├─ bootstrapContext.close(context)    — 启动期对象转正
 │       ├─ registerSingleton("springApplicationArguments")
 │       ├─ 懒加载 / KeepAlive / PropertySource 排序 后处理器
 │       ├─ load(context, sources)             — 加载 @Configuration 类
 │       └─ listeners.contextLoaded()          → ApplicationPreparedEvent
 │
 ├─ 阶段 4: refreshContext(context)
 │   └─ context.refresh()  ← Spring IoC 容器核心（Bean 创建在此完成）
 │
 ├─ 阶段 5: afterRefresh() — 空方法，供子类覆盖
 │
 ├─ 阶段 6: listeners.started(context, timeTaken) → ApplicationStartedEvent
 │   └─ callRunners() — 执行 CommandLineRunner / ApplicationRunner
 │
 ├─ 阶段 7: listeners.ready(context, timeTaken) → ApplicationReadyEvent
 │
 └─ 异常: handleRunFailure() → FailureAnalyzers 诊断 → listeners.failed()
```

### 10.5 关键 Private 方法详解

| 方法 | 核心逻辑 |
|------|---------|
| `prepareEnvironment()` | 创建 Env → 配置属性源/Profile → 触发 `environmentPrepared` → 绑定 `spring.main.*` → Environment 类型转换（Standard→Reactive） |
| `prepareContext()` | 装配 Context：注入 env、AOT、执行 initializer SPI、注册 Boot 内部单例 Bean、加载 sources |
| `createApplicationContext()` | `applicationContextFactory.create(webType)` → Servlet/Reactive/Standard 三种 Context |
| `getOrCreateEnvironment()` | 优先自定义 env → `ApplicationContextFactory.createEnvironment()` → 兜底 `ApplicationEnvironment` |
| `configurePropertySources()` | 命令行参数 → `SimpleCommandLinePropertySource`（最高优先级）；合并 defaultProperties |
| `bindToSpringApplication()` | `Binder.get(env).bind("spring.main", Bindable.ofInstance(properties))` — 用属性文件覆盖 SpringApplication 设置 |
| `getRunListeners()` | `SpringFactoriesLoader` 加载 `SpringApplicationRunListener` + 检查 `SpringApplicationHook` |
| `printBanner()` | 根据 bannerMode 打印（CONSOLE/LOG/OFF），支持自定义 `Banner` 实现 |
| `callRunners()` | 按 `@Order` 排序后依次调用 `ApplicationRunner` 和 `CommandLineRunner` |
| `handleRunFailure()` | 发布 `ApplicationFailedEvent` → `FailureAnalyzers` 诊断 → 包装异常重新抛出 |
| `load()` | 支持三种 source：`AnnotatedBeanDefinitionReader`（类）、`XmlBeanDefinitionReader`（XML）、`ClassPathBeanDefinitionScanner`（包名） |

### 10.6 公开 API：Fluent 定制入口

| 类别 | 方法 | 默认值 |
|------|------|--------|
| Banner | `setBanner()`, `setBannerMode()` | CONSOLE |
| 属性 | `setDefaultProperties()`, `setAdditionalProfiles()` | — |
| Context | `setApplicationContextFactory()`, `setBeanNameGenerator()` | DEFAULT |
| 行为 | `setLazyInitialization()`, `setAllowCircularReferences()`, `setKeepAlive()` | false/false/false |
| SPI | `addInitializers()`, `addListeners()` | — |
| 度量 | `setApplicationStartup()` | DEFAULT |
| 环境 | `setEnvironment()`, `setEnvironmentPrefix()` | — |

### 10.7 静态便利方法

| 方法 | 用途 | 版本 |
|------|------|------|
| `run(Class, String...)` | 一行启动（最常用） | 1.0.0 |
| `run(Class[], String[])` | 多 source 启动 | 1.0.0 |
| `main(String[])` | 通过 `--spring.main.sources` 指定配置类 | 1.0.0 |
| `exit(ApplicationContext, ExitCodeGenerator...)` | 优雅退出 + 退出码 | 1.0.0 |
| `getShutdownHandlers()` | 注册 JVM 关闭回调 | 2.5.1 |
| `withHook(SpringApplicationHook, Runnable)` | 测试中拦截启动流程 | 3.0.0 |
| `from(ThrowingConsumer<String[]>)` | Augmented API（增强启动） | 3.1.0 |

### 10.8 内部类体系（9 个内部类/接口）

| 内部类 | 可见性 | 职责 |
|--------|--------|------|
| `Startup` / `StandardStartup` / `CoordinatedRestoreAtCheckpointStartup` | pkg-private | 启动计时：标准模式 vs CRaC 恢复模式 |
| `PropertySourceOrderingBeanFactoryPostProcessor` | private | BFP：确保 Boot 属性源排在 `@PropertySource` 之后 |
| `AbandonedRunException` | public | 静默退出异常（3.0+），不触发 FailureAnalyzers |
| `NativeImageRequirementsException` | pkg-private | GraalVM Native Image Java 版本检查（≥25） |
| `SingleUseSpringApplicationHook` | private | 确保 Hook 只执行一次（AtomicBoolean） |
| `KeepAlive` | private | 3.2+：监听 ContextRefreshedEvent 启动非守护线程 |
| `Augmented` | public | 3.1+：`.from(main).with(Config).run(args)` 流式 API |
| `Running` | public interface | Augmented 的运行结果 |
| `FactoryAwareOrderSourceProvider` | private | 为 @Bean 工厂方法提供 Order 排序源 |

### 10.9 设计模式

| 模式 | 体现 |
|------|------|
| **模板方法** | `run()` 骨架；`configureProfiles()`, `configurePropertySources()`, `afterRefresh()` 等 protected 扩展点 |
| **策略模式** | `ApplicationContextFactory.create(WebApplicationType)` → 不同 Context 实现 |
| **观察者** | `SpringApplicationRunListener` 7 个回调 + 6 种 Spring Event |
| **责任链** | `FailureAnalyzers`：多个 FailureAnalyzer 依次尝试分析异常 |
| **钩子** | `SpringApplicationHook` + `ThreadLocal`：测试中拦截/增强启动 |
| **构建器** | 大量 setter → `run()` 的伪构建器风格 |

### 10.10 启动事件与 Listener 生命周期

| 阶段 | `SpringApplicationRunListener` 回调 | 对应 Spring Event |
|------|-------------------------------------|-------------------|
| 0 | `starting(ConfigurableBootstrapContext)` | `ApplicationStartingEvent` |
| 2 | `environmentPrepared(bootstrapContext, environment)` | `ApplicationEnvironmentPreparedEvent` |
| 3 | `contextPrepared(context)` | `ApplicationContextInitializedEvent` |
| 3 | `contextLoaded(context)` | `ApplicationPreparedEvent` |
| 6 | `started(context, Duration)` | `ApplicationStartedEvent` |
| 7 | `ready(context, Duration)` | `ApplicationReadyEvent` |
| ❌ | `failed(context, Throwable)` | `ApplicationFailedEvent` |

### 10.11 AOT / Native Image / CRaC 支持

- **AOT**：`prepareContext()` 中调用 `addAotGeneratedInitializerIfNecessary()`，如果 AOT 模式则注入 `${mainClass}__ApplicationContextInitializer`
- **Native Image**：`NativeImageRequirementsException` 检查 Java ≥ 25
- **CRaC**：`Startup.create()` 根据 classpath 自动选择 `CoordinatedRestoreAtCheckpointStartup`；日志显示 "Restored" 而非 "Started"

---

### 10.12 总结：10 条设计心得

1. **启动是精心编排的流水线** — 7 阶段，每阶段都有 SPI 回调
2. **构造函数只做必要准备** — 推断类型、加载 SPI、推断 mainClass，不干重活
3. **`run()` 是模板方法** — 骨架固定，通过 protected 方法和 SPI 扩展
4. **SPI 是核心扩展机制** — `SpringFactoriesLoader` + `META-INF/spring/*.imports`
5. **`ApplicationProperties` 聚合配置** — 通过 `Binder` 绑定 `spring.main.*`，允许属性文件覆盖编程设置
6. **线程安全考虑周全** — `ThreadLocal<Hook>`、`AtomicBoolean`（SingleUseHook）、`AtomicReference`（KeepAlive）
7. **AOT 是一等公民** — 构造函数就检查，Context 准备阶段自动注入 AOT initializer
8. **CRaC 支持优雅内嵌** — `Startup` 抽象类多态，不影响主流程代码
9. **错误处理分层** — `AbandonedRunException`（静默）vs 普通异常（FailureAnalyzers 诊断）
10. **`Augmented` API 是测试利器** — `SpringApplication.from(main).with(TestConfig.class).run(args)` 一行增强启动





---

## 十一、SPI 机制详解

Spring Boot 的强大灵活性的核心来源就是 **SPI（Service Provider Interface）**。

### 11.1 什么是 SPI

> **SPI = 接口定义在"框架"里，实现在"插件"里。框架调用接口时，自动发现 classpath 里所有插件的实现。**

跟 API 的区别：

| | API | SPI |
|---|-----|-----|
| **谁定义** | 框架 | 框架 |
| **谁实现** | 框架 | **第三方 / 用户** |
| **谁调用** | 用户调用框架 | **框架调用插件** |
| **比喻** | 遥控器 → 电视 | 插头标准 → 各家电器 |

### 11.2 Spring Boot 里的 SPI 怎么工作

**第 1 步：框架定义接口**

```java
// Spring Boot 定义的接口（在 spring-boot.jar 里）
public interface ApplicationContextInitializer {
    void initialize(ConfigurableApplicationContext context);
}
```

**第 2 步：插件提供实现，写进注册文件**

各个 starter 在自己的 jar 包里放一个文件：

```
META-INF/spring/org.springframework.context.ApplicationContextInitializer.imports
```

文件内容就是实现类的全限定名，一行一个：

```
com.example.mystarter.MyInitializer
```

**第 3 步：框架用 SpringFactoriesLoader 加载**

```java
// SpringApplication 构造函数里做的：
List<ApplicationContextInitializer> initializers = 
    SpringFactoriesLoader.forDefaultResourceLocation(classLoader)
        .load(ApplicationContextInitializer.class);
```

`SpringFactoriesLoader` 做的事情：
1. 从所有 jar 包里找 `META-INF/spring/org.springframework.context.ApplicationContextInitializer.imports`
2. 读取里面的类名
3. 反射创建实例
4. 排序后返回

**第 4 步：框架在合适的时机调用**

```java
// SpringApplication.prepareContext() 里：
for (ApplicationContextInitializer initializer : initializers) {
    initializer.initialize(context);  // 每个插件都有机会"加料"
}
```

### 11.3 `.imports` vs 老版 `spring.factories`

| | spring.factories（2.x） | .imports（3.x） |
|---|---|---|
| **文件位置** | `META-INF/spring.factories` | `META-INF/spring/接口名.imports` |
| **文件数量** | 所有 SPI 写在一个文件 | 每个接口一个文件 |
| **格式** | `接口名=实现类1,实现类2` | 每行一个实现类 |
| **加载时机** | 每次调用都要加载全部 | 按需加载单个接口 |
| **性能** | 慢 | 快 |

### 11.4 Spring Boot 里有哪些 SPI

都是框架定义接口、插件实现、框架调用的模式：

| SPI 接口 | 什么时候被调用 | 谁来实现（举例） |
|----------|---------------|-----------------|
| `ApplicationContextInitializer` | Context 创建后、Refresh 前 | 各 starter（如 Web 环境的 `ServletWebServerApplicationContext` 初始化） |
| `ApplicationListener` | 任何 Spring 事件发生时 | 框架自己 + 用户自定义 |
| `SpringApplicationRunListener` | 启动每一步 | `EventPublishingRunListener`（唯一默认实现） |
| `EnvironmentPostProcessor` | 配置加载后、Environment 定型前 | `ConfigDataEnvironmentPostProcessor`（加载 application.yml） |
| `FailureAnalyzer` | 启动失败时 | `PortInUseFailureAnalyzer`（"端口被占用"）、`NoSuchBeanDefinitionFailureAnalyzer` 等 |
| `BootstrapRegistryInitializer` | 比 Environment 还早 | 需要启动早期注册对象的场景 |
| `AutoConfigurationImportFilter` | 自动配置条件过滤 | `OnClassCondition`、`OnBeanCondition`、`OnPropertyCondition` |

### 11.5 一个真实例子：端口被占用的诊断

当你启动时端口被占用，Spring Boot 打印：

```
***************************
APPLICATION FAILED TO START
***************************

Description:
Web server failed to start. Port 8080 was already in use.
```

这背后是 SPI：

1. **`FailureAnalyzer` 是 SPI 接口**（框架定义）
2. **`PortInUseFailureAnalyzer` 是 SPI 实现**（Spring Boot 自带）
3. 启动失败时，框架调用 `SpringFactoriesLoader` 加载所有 `FailureAnalyzer`
4. 逐个调用 `analyze(failure, ...)`
5. `PortInUseFailureAnalyzer` 发现异常是 `PortInUseException`，返回人性化描述

**如果没有 SPI**，你只能看到臭长的 Java 异常堆栈。有了 SPI，每个场景都能给出对症的诊断。

### 11.6 一句话总结

> **SPI = 框架定接口、插件写实现、配置文件登记、框架反射加载并调用。** Spring Boot 的自动配置、启动事件、失败诊断、配置加载……几乎每个核心功能都是靠 SPI 串联起来的。没有 SPI 就没有 Spring Boot 的"智能"。



---

## 十二、启动过程补充：扫描、refresh、两个 Listener

前面章节解释了启动流程的框架，这一章把最近讨论的三个关键问题补上。

### 12.1 `@ComponentScan` 扫描到底在干什么

你把 `MyApp.class` 传给 `SpringApplication.run()`，`MyApp` 上有 `@SpringBootApplication`（包含 `@ComponentScan`）。

**扫描 = Spring 打开你的项目文件夹，按包路径一个一个 `.class` 文件看过来，发现带了特定注解的就抓进容器管理。**

具体找这些注解：

| 注解 | 意思 |
|------|------|
| `@Component` | 普通 Bean |
| `@Service` | 业务逻辑 |
| `@Repository` | 数据库操作 |
| `@Controller` / `@RestController` | Web 接口 |
| `@Configuration` | 配置类（里面可能有 `@Bean` 方法） |

**实例**：

```
com.example/
├── MyApp.java               ← @SpringBootApplication，扫描起点
├── controller/
│   └── UserController.java  ← @RestController  → 抓进容器
├── service/
│   └── UserService.java     ← @Service         → 抓进容器
└── utils/
    └── StringUtils.java     ← 没注解           → 跳过
```

> **注意**：`prepareContext` 阶段只是"标记来源"，真正的扫描动作发生在 `refresh()` 里，由 `ConfigurationClassPostProcessor` 完成。

---

### 12.2 `context.refresh()` 内部 12 步

`refresh()` 是 Spring Framework 提供的，不是 Boot 写的。Boot 只是触发它。

```java
public void refresh() {
    // ① 准备：记时间，校验配置
    prepareRefresh();

    // ② 拿到 BeanFactory
    obtainFreshBeanFactory();

    // ③ 给工厂装"基础设施"（类加载器、表达式解析器、类型转换器...）
    prepareBeanFactory(beanFactory);

    // ④ 留给子类扩展
    postProcessBeanFactory(beanFactory);

    // ⑤ ★ 关键：执行 BeanFactoryPostProcessor
    //   ConfigurationClassPostProcessor 在这里干活：
    //     - 解析 @Configuration 类
    //     - 执行 @ComponentScan（扫描所有 @Component/@Service/@Controller...）
    //     - 处理 @Import(AutoConfigurationImportSelector.class)
    //         → 加载 200+ 自动配置类候选
    //         → @ConditionalOnClass / @ConditionalOnBean 条件过滤
    //         → 剩下的注册为 Bean 定义
    invokeBeanFactoryPostProcessors(beanFactory);

    // ⑥ 注册 BeanPostProcessor（拦截 Bean 创建过程）
    registerBeanPostProcessors(beanFactory);

    // ⑦ 国际化消息源
    initMessageSource();

    // ⑧ 事件广播器
    initApplicationEventMulticaster();

    // ⑨ ★ 关键：Spring Boot 在这启动 Web 服务器（Tomcat/Netty）
    onRefresh();

    // ⑩ 注册所有 ApplicationListener
    registerListeners();

    // ⑪ ★ 最耗时：按"图纸"把所有单例 Bean 造出来
    finishBeanFactoryInitialization(beanFactory);

    // ⑫ 收尾：发 ContextRefreshedEvent
    finishRefresh();
}
```

**三步最关键**：

| 步骤 | 做什么 | 花的时间 |
|------|--------|---------|
| ⑤ `invokeBeanFactoryPostProcessors` | 扫描 + 自动配置解析 + 条件过滤 → 生成 Bean 定义（图纸） | 中等 |
| ⑨ `onRefresh` | **启动内嵌 Web 服务器**，绑定端口 | 快 |
| ⑪ `finishBeanFactoryInitialization` | 按图纸把 Bean 挨个 new 出来、注入依赖 | **最耗时** |

---

### 12.3 `SpringApplicationRunListener` vs `ApplicationListener`

两个名字很像，但完全是两种东西：

| | SpringApplicationRunListener | ApplicationListener |
|---|---|---|
| **管什么** | **只管启动**的 7 个阶段 | 监听 Spring 容器**所有事件**（启动、关闭、自定义...） |
| **谁调用** | `SpringApplication.run()` 直接调用 | Spring 容器的事件广播器调用 |
| **能用时机** | 启动最早阶段就能用（比 Context 还早） | 必须在 Context 创建后才能收到 |
| **加载方式** | SPI（`SpringFactoriesLoader`） | SPI + 手动 `addListeners()` + `@EventListener` 注解 |
| **默认实现** | `EventPublishingRunListener` | 没有默认，你自己写 |

**关键：默认的 `EventPublishingRunListener` 把它们桥接起来**：

```
SpringApplication 调用                    Spring 容器广播
        │                                       │
  SpringApplicationRunListener          ApplicationListener
        │                                       │
  ready() ────发布 ApplicationReadyEvent────→ MyListener.onReady()
```

- `SpringApplicationRunListener` = **信使**（知道启动到哪一步了）
- `ApplicationListener` = **收信人**（对某些事件感兴趣）
- `EventPublishingRunListener` = **翻译官**（把启动步骤翻译成 Spring 事件）

---

### 12.4 启动全过程时间线（一张图说清）

```
时间 →

[SpringApplication 构造函数]
  ├─ 猜 Web 类型
  ├─ SPI 加载 3 种插件
  └─ StackWalker 推断 mainClass

[run() 开始]
  ├─ starting                  ← SpringApplicationRunListener 第 1 次回调
  │
  ├─ prepareEnvironment
  │   └─ 读 application.yml、命令行参数、环境变量 → 合并到 Environment
  │   └─ environmentPrepared   ← SpringApplicationRunListener 第 2 次回调
  │
  ├─ createApplicationContext  ← 根据 Web 类型创建对应容器
  │
  ├─ prepareContext
  │   ├─ 把 Environment、MyApp.class 塞进 Context
  │   ├─ contextPrepared       ← 第 3 次回调
  │   └─ contextLoaded         ← 第 4 次回调
  │
  ├─ ★ refresh()  ← 最重的一步（Spring Framework 提供）
  │   ├─ ⑤ 扫描 + 自动配置条件过滤 → 生成 Bean 定义
  │   ├─ ⑨ 启动 Web 服务器（Tomcat/Netty 绑定端口）
  │   └─ ⑪ 挨个创建所有单例 Bean（最耗时）
  │
  ├─ started                   ← 第 5 次回调
  ├─ callRunners（CommandLineRunner / ApplicationRunner）
  │
  └─ ready                     ← 第 6 次回调（最后一次）
       ↓
     应用就绪，可以接客
```
