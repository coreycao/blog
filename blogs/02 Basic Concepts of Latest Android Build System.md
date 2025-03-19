# Android 项目构建系统的一些名词解释

## Gradle Plugin

Android 项目的开发使用 Gradle 构建系统进行管理，使用到 Kotlin 和 Compose 这些技术栈时需要在构建系统中添加诸多依赖和配置，这些依赖和配置的来源、作用、版本兼容等令人混淆，此文对其中一些不太熟悉的内容进行简单梳理。

Gradle 插件（Gradle Plugin）是扩展 Gradle 构建系统功能的模块化组件。Android 项目中需要使用的最基本的插件有：

- Android Gradle Plugin（AGP）
- Kotlin Plugin

此外，复杂项目中往往出于各种架构、优化等需求还会开发诸多自定义插件来拓展功能。

### AGP

Android Gradle Plugin（AGP） 是 Android 项目开发的基本插件，它提供了 Android 项目的构建和打包功能。

在项目的顶级 build.gradle.kts 指定插件版本：

```kotlin
plugins {
    id("com.android.application") version "8.9.0" apply false
}
```

注意语法中的```apply false```，其作用是声明该插件但不立即应用。Android 的根项目通常是不需要应用 AGP 的，在顶级 build.gradle.kts 中统一管理插件版本可以避免插件版本不一致的问题。子模块中应用插件时不再需要指定版本，Gradle 会自动使用顶级 build.gradle.kts 中指定的版本。

随着 AGP 的不断升级，其所需的最低 Gradle 版本也在不断提升；另一方面 AGB 的版本也与 Android Studio 的兼容性有关系。具体的版本可以参考 [Android Developers 的文档](https://developer.android.com/build/releases/gradle-plugin?hl=zh-cn)。

### Kotlin Plugin

```org.jetbrains.kotlin.android```这个 Gradle 插件用于启用 Kotlin 语言在 Android 项目中的支持。Android 项目的开发已经离不开要使用 Kotlin，因此该插件也是 Android 项目构建中的必备插件。除了提供对 Kotlin 语言的支持外，该插件还使得我们可以在项目中使用 Kotlin DSL替代 Groovy 来配置构建任务和扩展。

### Compose-Compiler Plugin

```org.jetbrains.kotlin.plugin.compose```是用于编译和优化 Compose UI 性能的插件。若项目中使用到了 Compose，则需要该插件。

需要注意的是，```org.jetbrains.kotlin.plugin.compose```兼容的 Kotlin 版本为 2.0.0+，而对于更早版本的 Kotlin，需要通过在模块的 build.gradle.kts 中添加以下依赖来使用 Compose：

```Kotlin
android {
    buildFeatures {
        compose = true
    }

    composeOptions {
        kotlinCompilerExtensionVersion = "1.5.15"
    }

    kotlinOptions {
        jvmTarget = "19"
    }
}
```

Compose, Compose Compiler Plugin, Kotlin 三者之间也存在一些需要留意的版本兼容性问题，具体可以参见[官方的文档](https://developer.android.com/jetpack/androidx/releases/compose-kotlin?hl=zh-cn)。

## BOM

[BOM](https://developer.android.com/develop/ui/compose/bom?hl=zh-cn) 是 Bill of Materials 的缩写，意为物料清单。在 Gradle 中，BOM 是一种用于管理依赖版本的机制。

BOM 允许我们在一个地方定义依赖的版本，然后在其他项目中引用这些版本，这样可以避免不同项目中依赖版本不一致的问题。

例如，为项目添加 Compose 相关依赖时，需要依赖 ```ui-graphics```, ```ui-tooling-preview```等多个库，利用 BOM，则可以如下添加依赖

```kotlin
    implementation(platform("androidx.compose:compose-bom:2025.02.00"))
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.ui:ui-graphics")
    implementation("androidx.compose.ui:ui-tooling-preview")
```

又或者，在项目中使用到了 Ktor，则可以如下添加依赖:

```kotlin
    // ktor
    implementation(platform("io.ktor:ktor-bom:3.1.0"))
    implementation("io.ktor:ktor-client-android")
    implementation("io.ktor:ktor-client-serialization")
    implementation("io.ktor:ktor-client-logging")
    implementation("io.ktor:ktor-client-content-negotiation")
    implementation("io.ktor:ktor-serialization-kotlinx-json")
```

[官方的文档](https://developer.android.com/develop/ui/compose/bom/bom-mapping?hl=zh-cn)里给出了 Compose BOM 版本与具体库版本之间的对应关系。

## Version Catalogs

Version Catalogs 是 Gradle 7.6.0 引入的新特性，用于管理依赖版本，而在新版的 AGP 的支持下，[Android 项目可以使用该特性方便地进行依赖配置](https://developer.android.com/build/migrate-to-catalogs?hl=zh-cn).

使用方式也比较简单，在根项目的 gradle 文件夹中，创建一个名为 libs.versions.toml 的文件。Gradle 默认会在 libs.versions.toml 文件中查找目录。

toml 文件的语法规则也清晰易懂，一份 libs.versions.toml 的示例如下：

```toml
[versions]
agp = "8.9.0"
kotlin = "2.1.10"
coreKtx = "1.15.0"
junit = "4.13.2"
junitVersion = "1.2.1"
espressoCore = "3.6.1"
appcompat = "1.7.0"
material = "1.12.0"
avtivityCompose = "1.10.1"
composeBom = "2025.03.00"
coroutinesCore = "1.10.1"
ktorBom = "3.1.0"
lifecycleRuntimeKtx = "2.8.7"
koin = "4.0.0"
coil = "3.0.4"

[libraries]
androidx-core-ktx = { group = "androidx.core", name = "core-ktx", version.ref = "coreKtx" }
junit = { group = "junit", name = "junit", version.ref = "junit" }
androidx-junit = { group = "androidx.test.ext", name = "junit", version.ref = "junitVersion" }
androidx-espresso-core = { group = "androidx.test.espresso", name = "espresso-core", version.ref = "espressoCore" }
androidx-appcompat = { group = "androidx.appcompat", name = "appcompat", version.ref = "appcompat" }
material = { group = "com.google.android.material", name = "material", version.ref = "material" }
androidx-activity-compose = { group = "androidx.activity", name = "activity-compose", version.ref = "avtivityCompose" }
androidx-compose-bom = { group = "androidx.compose", name = "compose-bom", version.ref = "composeBom" }
androidx-ui = { group = "androidx.compose.ui", name = "ui" }
androidx-ui-graphics = { group = "androidx.compose.ui", name = "ui-graphics" }
androidx-ui-tooling = { group = "androidx.compose.ui", name = "ui-tooling" }
androidx-ui-tooling-preview = { group = "androidx.compose.ui", name = "ui-tooling-preview" }

# coroutine
coroutines-core = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-core", version.ref = "coroutinesCore" }

# ktor
ktor-bom = { group = "io.ktor", name = "ktor-bom", version.ref = "ktorBom" }
ktor-android = { group = "io.ktor", name = "ktor-client-android" }
ktor-content-negotiation = { group = "io.ktor", name = "ktor-client-content-negotiation" }
ktor-logging = { group = "io.ktor", name = "ktor-client-logging" }
ktor-serialization = { group = "io.ktor", name = "ktor-client-serialization" }
ktor-serialization-ktxJson = { group = "io.ktor", name = "ktor-serialization-kotlinx-json" }

# lifecycle
lifecycle-runtime-ktx = { group = "androidx.lifecycle", name = "lifecycle-runtime-ktx", version.ref = "lifecycleRuntimeKtx" }
lifecycle-runtime-compose = { group = "androidx.lifecycle", name = "lifecycle-runtime-compose", version.ref = "lifecycleRuntimeKtx" }
lifecycle-viewmodel-ktx = { group = "androidx.lifecycle", name = "lifecycle-viewmodel-ktx", version.ref = "lifecycleRuntimeKtx" }

# Koin for Android
koin-androidx-compose = { group = "io.insert-koin", name = "koin-androidx-compose", version.ref = "koin" }

# Coil
coil-compose = { group = "io.coil-kt.coil3", name = "coil-compose", version.ref = "coil" }
coil-okhttp = { group = "io.coil-kt.coil3", name = "coil-network-okhttp", version.ref = "coil" }

[plugins]
android-application = { id = "com.android.application", version.ref = "agp" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
compose-compiler = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }

```
