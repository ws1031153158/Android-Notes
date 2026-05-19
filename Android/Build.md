# Gradle
## 三阶段
1.Initialization：确定有哪些 project  
2.Configuration：执行 build.gradle，创建和配置 task（此时还没真正跑任务）  
3.Execution：按 task graph 真正执行 task  
## Task 和 doFirst/doLast
doFirst：task 执行前插入逻辑  
doLast：task 执行后插入逻辑  
## Plugin
Plugin<Project> 是 Gradle 的二进制插件接口，你可以把它理解为：“当插件被应用到某个 Project 时，Gradle 会回调你一次，让你去配置这个项目。”  
```
class DefensorPlugin : Plugin<Project> {
    override fun apply(project: Project) {
        // 这里写插件逻辑
    }
}

// Plugin<Project>：模块/工程插件（最常见）
// Plugin<Settings>：作用在 settings.gradle
// Plugin<Gradle>：作用在整个 Gradle 实例（更底层）
```

流程：  
```
1.业务工程写 apply plugin: 'defensor'（或 plugins { id(...) }）
2.Gradle 根据插件 id 找到描述文件
3.META-INF/gradle-plugins/defensor.properties
4.读取 implementation-class=com.meituan.gradle.defensor.DefensorPlugin
5.反射实例化这个类
6.调用 apply(project)（配置阶段）
7.插件在这里注册 extension、任务、回调
8.到执行阶段，真正跑被注册/挂钩的任务
```
## Transform
Gradle 构建流程中，允许开发者对编译后的 class 文件（字节码）或资源进行自定义处理的一个阶段。这个机制主要通过 Transform API 实现，常用于字节码插桩、代码分析、自动生成代码等场景   
### Transform API 简介
Transform API 是 Android Gradle Plugin（AGP）提供的一个扩展点。它允许开发者在 Java 源码编译成 class 文件后、打包成 dex 文件之前，对 class 文件、jar 包、资源文件等进行处理。
### 工作流程
1.源码（.java/.kt）编译成 class 文件。  
2.Transform 阶段：此时可以通过实现自定义的 Transform，对 class 文件或 jar 包进行扫描、修改、插桩等操作。  
3.处理完成后，继续后续 dex 转换、打包等流程。
### 典型应用场景
1.AOP（面向切面编程）插桩，比如埋点、性能监控。  
2.自动生成代码或修改已有字节码。  
3.资源混淆、资源处理。  
4.代码安全加固。
### 实现方式
开发者可以通过实现 Transform 类，并注册到 Gradle 插件中即可  
<img width="719" alt="截屏2025-06-26 11 50 00" src="https://github.com/user-attachments/assets/d101a8b7-859a-40dc-ab8e-fa971876423e" />
### 生命周期
Transform 阶段位于:Java/Kotlin 源码编译之后,DEX 转换之前,因此，Transform 可以对所有 class 文件做统一处理
# SPI
SPI 全称为 (Service Provider Interface) ，是JDK内置的一种外部服务发现机制。SPI 区别于API，API的接口定义和实现都位于服务提供方，而SPI 的接口则由服务使用方定义，具体的实现由服务方提供。这种机制十分适合用于统一不同服务提供方的接口实现  
# ServiceLoader
java通过引入ServiceLoader来实现SPI机制，ServiceLoader正如其命名，它主要是用来装载各种service provider提供的service。  
ServiceLoader是通过service provider的配置文件来装载服务的，因此在提供了服务接口实现的同时，还需要在jar包的META-INF/services/目录里同时创建一个以服务接口命名的文件。该文件内容就是实现该服务接口的具体实现类。服务使用方只需要调用ServiceLoader.load(Class class)，就会触发相应服务的装载，通过jar包META-INF/services/里的配置文件找到具体的服务实现，并完成实例注入。
