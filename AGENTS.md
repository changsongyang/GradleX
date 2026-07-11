# AGENTS.md

## 项目定位

GradleX 是配合 Gradle 系列文章维护的 Android 教学项目，不是以业务功能为中心的应用。仓库同时展示 Groovy DSL、Kotlin DSL、`buildSrc`、Version Catalog、自定义 Task、脚本插件和二进制插件等写法。修改时优先保证示例清晰、可独立理解和可复现，不要仅为了“统一风格”而合并这些有意并存的方案。

## 构建基线

- README 环境表标注 Android Studio Meerkat；顶部徽章仍写 Koala，仓库文档在这一点上存在不一致
- JDK 17
- Gradle 8.11.1（以 Wrapper 为准）
- Android Gradle Plugin 8.9.0
- 当前主构建使用 Kotlin 2.1.0；`gradle.properties` 中的 `kotlin_version=1.9.20` 是未被主路径引用的旧示例值
- `compileSdk` 35，`minSdk` 23，`targetSdk` 33

始终使用仓库内的 `./gradlew`，不要依赖机器上全局安装的 Gradle。

## 仓库结构

- `app/`：Android 示例应用，也是大部分构建案例的承载模块。
- `plugin/`：`com.yechaoa.plugin.gradleX` 二进制 Gradle 插件的源码和发布配置。
- `build.gradle.kts`、`settings.gradle.kts`：根工程插件、仓库、模块和生命周期示例。
- `gradle/libs.versions.toml`：当前主构建使用的 Version Catalog。
- `buildSrc/`：传统依赖版本管理的教学示例；当前 `Versions`、`Libs` 未被主构建引用，与 Version Catalog 并存是有意设计。
- `task.gradle`：Task、Task 依赖、跳过策略和增量构建示例。
- `plugin.gradle`：脚本形式的自定义插件与嵌套 Extension 示例。
- `channel.gradle`：多渠道和自定义 SourceSet 示例。
- `UseLocalPlugin.gradle`、`useLocal.json`：远程依赖与本地源码之间的替换示例。
- `.github/workflows/android.yml`：以 JDK 17 执行 `./gradlew build` 的 CI。

## 关键构建链路

`app` 默认应用插件 ID `com.yechaoa.plugin.gradleX`，但根工程的 `buildscript` 默认把 JitPack 上的 `com.github.yechaoa.GradleX:plugin:1.8` 放入 classpath。因此，直接构建 `app` 时运行的是远程插件，不是当前工作区 `plugin/` 中的实现；仅仅 `include(":plugin")` 不会把本地插件接入 `app`。

`settings.gradle.kts` 已在 `pluginManagement` 和 `dependencyResolutionManagement` 中配置 JitPack 与 `plugin/build/maven-repo`，并使用 `RepositoriesMode.PREFER_SETTINGS`。在当前 Gradle 8.11.1 配置下，强制刷新依赖时可以从 JitPack 解析远程 classpath，也可以解析发布到上述目录的本地插件。不要仅根据根 `buildscript.repositories` 的列表判断实际仓库来源；需要排查解析问题时使用 `--refresh-dependencies --info` 查看真实请求。

本地插件开发流程：

1. 保持远程 classpath 时执行 `./gradlew :plugin:publish`，产物发布到 `plugin/build/maven-repo`。
2. 临时把根 `build.gradle.kts` 中的插件 classpath 切换为 `com.yechaoa.plugin:gradleX:1.6-SNAPSHOT`，并注释远程坐标。
3. 构建 `app`，确认 classpath 解析到本地坐标、输出来自本地插件，再验证 `gradleX {}` 行为。

除非任务明确要求，不要提交上述远程/本地 classpath 切换，也不要随意修改插件坐标或版本。

`UseLocalPlugin.gradle` 会读取 `useLocal.json`；条目的 `useLocal=true` 时，它会 include 本地模块，并只在 `app` 的 Configuration 上进行 dependency substitution。当前 JSON 中该值为 `false`。`settings.gradle.kts` 还会从 `local.properties` 读取同名配置，但这条活动路径目前只 include `:yutilskt`，`app` 中配套的依赖选择和 substitution 都是注释示例。命令行属性 `-PuseLocal=true` 当前没有被任何活动切换逻辑读取。两套示例不要未经要求合并，描述行为时以实际生效路径为准。

## GradleX 插件

插件入口是 `plugin/src/main/java/com/yechaoa/plugin/base/GradleXPlugin.java`，DSL 定义在 `CommonPluginExtension.java`，命名空间为 `gradleX`。当前功能包括：

- `printDependencies`：解析 Variant 的 CompileClasspath，取得传递模块后按 group 平铺分类打印；输出不保留严格的依赖树结构。
- `analysisSo`：扫描 AAR/JAR 中的 `.so` 文件。
- `checkSnapshot`、`blockSnapshot`：检查 SNAPSHOT 依赖，并可阻断构建；`blockSnapshot` 仅在 `checkSnapshot=true` 时生效。
- `permissionsToRemove`：在 Manifest 处理后修改权限字符串。

该实现通过 `AppExtension`、`applicationVariants` 和旧 Variant output/Manifest 接口操作构建，这些属于 legacy/deprecated AGP API；它还通过 Gradle 的 `afterEvaluate` 读取 DSL，后者不是 AGP 废弃 API，但不利于惰性配置和配置缓存兼容性。插件实现假定目标模块应用了 `com.android.application`，目前不支持 Android Library 或非 Android 模块。除非任务目标就是迁移 API，否则不要顺手重写整套实现。修改功能时需要覆盖 debug/release 以及开启多个选项的组合，避免在配置阶段无条件解析所有依赖。

注意：当前“移除权限”实现调用 `String.replaceAll(permission, "android.permission.INTERNET")`，会把 `permission` 当作正则表达式，并非按 XML 结构删除权限节点。描述或测试现有行为时应保持准确；只有任务明确要求修复时才改变语义。

## 编码与配置风格

- Kotlin、Kotlin DSL 和 Java 使用 4 空格缩进；Groovy 沿用现有 4 空格风格。
- Kotlin 优先使用惯用 DSL 和类型安全 API，不新增无必要的分号或强制类型转换。
- Java 保持当前简单直接的 Gradle API 调用风格；新增公共 DSL 字段时添加中文说明和明确默认值。
- 注释以中文为主，面向学习者说明“为什么”和启用方式。不要添加只复述代码的注释。
- 教学代码常通过注释切换不同方案。新增案例时应给出最短启用步骤，并保证默认配置仍可构建。
- 保留文件当前使用的 DSL：不要无任务依据地把 `.gradle` 批量迁移为 `.gradle.kts`，也不要把 Java 插件实现整体改写为 Kotlin。
- 版本来源多样是教学主题。新增主路径依赖时优先使用 `gradle/libs.versions.toml`；演示其他版本管理方式时明确标注用途，不要悄悄制造新的权威版本源。
- 不提交 `local.properties`、签名文件、`keystore.properties`、访问令牌或其他机器/账户相关配置。
- 不手工编辑 `build/`、`.gradle/` 等生成目录。除非任务专门讨论产物元数据，也不要更新已生成的 APK/AAR 信息。

## 修改原则

- 先确定改动属于教学脚本、示例 App、插件实现还是发布流程，避免跨层级重构。
- 保留与系列文章对应的示例和注释；删除或大幅改写示例前需确认任务明确要求。
- 修改依赖解析、Variant、Manifest 或产物逻辑时，关注 Gradle 配置缓存、任务惰性配置和增量构建，不要在配置阶段做不必要的文件 IO 或依赖解析。
- 仓库启用了并行构建、构建缓存和配置缓存；配置缓存目前通过旧兼容属性 `org.gradle.unsafe.configuration-cache` 开启。开启配置不代表现有 `afterEvaluate`、`projectsEvaluated` 等示例已完全兼容，新增相关逻辑前应实际验证。
- 保持改动聚焦，不顺带格式化整份大型 Gradle 脚本。
- 尊重工作区已有未提交改动，不覆盖或回退与当前任务无关的内容。

## 验证

根据改动范围选择最小但充分的验证：

```bash
# 仅验证工程配置和模块加载
./gradlew projects --console=plain

# App 源码、资源或 app 构建配置
./gradlew :app:assembleDebug

# 插件源码或发布配置（编译并执行 Gradle 插件校验）
./gradlew :plugin:build

# 本地插件发布
./gradlew :plugin:publish

# 当前有效的单元测试主要来自 app
./gradlew test

# 与 CI 一致的完整验证
./gradlew build
```

涉及动态参数时还应验证对应命令，例如：

```bash
./gradlew :app:assembleDebug -PVersionName=3.0 -PVersionCode=3
./gradlew :app:assembleRelease -PisRelease=true
./gradlew :app:assembleRelease -PisRelease=true -PonlyArm64=true
```

Android 仪器测试只有在设备或模拟器可用时运行。若因外部仓库、Android SDK、签名文件或本地兄弟仓库缺失无法验证，应在交付说明中明确列出未执行项及原因，不要伪造成功结果。

`plugin/src/test` 下现有文件是 Kotlin/JUnit 模板，但 `plugin` 模块没有应用 Kotlin JVM 插件，也没有声明 JUnit 或 Gradle TestKit，因此这些文件当前不会被 `:plugin:build` 或 `test` 编译执行。`app` 默认又使用远程 1.8 插件，所以普通 `:app:assembleDebug` 也不能验证工作区中的插件改动。验证 GradleX 插件行为时，必须先完成上文的本地插件发布与 classpath 接线，再构建 `app` 并检查对应输出；不要把 `./gradlew test` 的成功当作插件逻辑已经被测试。

## 文档同步

如果修改了插件 DSL、插件坐标、支持的构建环境、使用步骤或新增教学章节，同步更新 `README.md`。命令、版本号和默认开关必须与实际生效的构建脚本一致。
