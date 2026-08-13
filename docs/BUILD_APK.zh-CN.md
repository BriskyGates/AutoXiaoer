# 修改源码并编译生成 APK（Auto小二）

本文说明在本仓库中**修改代码**后，如何**在本机编译并产出 APK**。适用于 Windows / macOS / Linux；下文命令在 Windows PowerShell 中使用时，将 `./gradlew` 换成 `.\gradlew.bat` 即可。

---

## 一、这个过程复杂吗？

**简要结论：**

| 情况 | 复杂度 |
|------|--------|
| 已安装 Android Studio，做过任意 Android 项目 | **低**：改代码 → 点 Build / 或一条 Gradle 命令即可 |
| 第一次接触 Android 开发 | **中等**：主要时间在安装 JDK、Android SDK、首次同步依赖；熟练之后与上栏相同 |

本项目是**标准 Android Gradle 工程**（Kotlin + Android Gradle Plugin），没有额外的前端打包链或自定义脚本编译链路，因此**不比其他同类原生 App 更复杂**。

---

## 二、工程与构建要点（读一遍有助于排错）

- **类型**：Android 原生应用，语言以 **Kotlin** 为主，构建脚本为 **Gradle Kotlin DSL**（`*.gradle.kts`）。
- **根工程名**：`XiaoEr`（见 `settings.gradle.kts`）。
- **应用模块**：`:app`。
- **包名 / applicationId**：`com.flowmate.autoxiaoer`（见 `app/build.gradle.kts`）。
- **版本**：由 `app/build.gradle.kts` 中 `versionCode`、`versionName` 控制（例如 `0.0.7`）。
- **编译 SDK**：`compileSdk = 35`；**最低系统版本**：`minSdk = 24`。
- **Java/Kotlin 字节码级别**：源码按 **Java 11** 兼容编译（`jvmTarget = "11"`）。
- **运行 Gradle/Android Gradle Plugin 的 JDK**：AGP 8.x 通常要求使用 **JDK 17** 来执行构建（Android Studio 自带 JDK 一般已满足）。若仅用命令行，请确认 `java -version` 与 `JAVA_HOME` 指向 JDK 17+。
- **依赖仓库**：Google、Maven Central、JitPack（见 `settings.gradle.kts`）；首次构建需要能访问外网以下载依赖。
- **代码风格**：根目录配置了 **ktlint**；提交前若跑完整 `check` 任务，需满足规范（见下文「可选检查」）。

---

## 三、环境准备

### 3.1 推荐方式：Android Studio

1. 安装 [Android Studio](https://developer.android.com/studio)（建议使用较新版本，与 AGP 8.13.x 匹配）。
2. 首次启动时按向导安装 **Android SDK**（含 Build-Tools、Platform SDK 等）。
3. 用 Android Studio **Open** 打开本仓库根目录（包含 `settings.gradle.kts` 的目录）。
4. 等待 **Gradle Sync** 完成；若提示缺少 SDK 组件，按提示安装即可。

同步成功后，IDE 通常会生成或维护 `local.properties`，其中包含 `sdk.dir=...`。**不要**把含本机绝对路径的 `local.properties` 提交到 Git（若仓库未忽略，请注意隐私与团队协作规范）。

### 3.2 纯命令行：只用 SDK Command-line Tools（可选）

不安装 Android Studio 时，可在 Windows 上仅用官方 **Command-line Tools** + **`sdkmanager`** 安装 Platform / Build-Tools，再配置 **`ANDROID_HOME`** 与项目根目录 **`local.properties`**（`sdk.dir=...`）。本仓库已忽略 `local.properties`，请勿提交。

**逐步操作（目录结构、安装包版本、`local.properties` 写法）见：** [ANDROID_SDK_CLI_WINDOWS.zh-CN.md](./ANDROID_SDK_CLI_WINDOWS.zh-CN.md)。

---

## 四、修改源码时可以改哪里

按需求常见入口包括（路径均在 `app/src/main/` 下）：

- **界面与资源**：`res/layout/`、`res/values/`、`res/drawable/` 等。
- **逻辑代码**：`java/` 或 `kotlin/` 下的包（实际目录以工程为准，命名空间为 `com.flowmate.autoxiaoer`）。
- **清单与组件**：`AndroidManifest.xml`。
- **构建参数**：`app/build.gradle.kts`（版本号、混淆开关、`debug`/`release` 配置等）。

修改后无需额外「导出」步骤，直接进入下一节的构建即可。

---

## 五、用 Android Studio 生成 APK

1. 菜单 **Build → Make Project**（或快捷键编译）确认无编译错误。
2. 生成 APK：
   - **调试包**：**Build → Build Bundle(s) / APK(s) → Build APK(s)**，并选择 **debug** 变体；  
     或使用 Gradle 面板执行 `:app → Tasks → build → assembleDebug`。
   - **发布包**：执行 **assembleRelease**（见下文签名说明）。

产物路径见第八节。

---

## 六、用命令行生成 APK

在仓库根目录执行：

**调试 APK（最常用，无需自有签名证书）：**

```powershell
.\gradlew.bat assembleDebug
```

**发布 APK：**

```powershell
.\gradlew.bat assembleRelease
```

类 Unix 系统：

```bash
./gradlew assembleDebug
./gradlew assembleRelease
```

首次运行会从 `gradle/wrapper/gradle-wrapper.properties` 下载指定版本 Gradle（当前为 **8.13**），耗时取决于网络。

---

## 七、可选：构建前做静态检查

仓库启用了 ktlint。若在 CI 或本地希望与「完整检查」对齐，可执行：

```powershell
.\gradlew.bat ktlintCheck
```

或通过：

```powershell
.\gradlew.bat check
```

（`check` 通常包含单元测试与其它校验任务，耗时更长。）

---

## 八、编译产物在哪里？文件名是什么？

本应用在 `app/build.gradle.kts` 中为 APK **自定义了文件名**：

- 形如：**`XiaoEr-<versionName>-<buildType>.apk`**
- 例如调试包：**`XiaoEr-0.0.7-debug.apk`**（具体版本以你本地的 `versionName` 为准）

常见输出目录：

- Debug：`app/build/outputs/apk/debug/`
- Release：`app/build/outputs/apk/release/`

---

## 九、Debug 与 Release、签名说明

### 9.1 Debug

- 使用 Android 默认 **debug 签名**，适合自测、安装到手机调试。
- `debug` 构建会将应用显示名设为 **「小二 Dev」**（通过 `resValue` 注入），与 release 区分。

### 9.2 Release

`app/build.gradle.kts` 中的逻辑要点：

- 若 **`app/release.keystore` 存在**，则 **release** 使用名为 `release` 的签名配置，并从环境变量读取密码：
  - `KEYSTORE_PASSWORD`
  - `KEY_ALIAS`
  - `KEY_PASSWORD`
- 若 **`release.keystore` 不存在**，则 **release 会回退为 debug 签名**（便于本地打出 release 包做试验，但**不适合**上架商店）。

若要正式发布，请自行生成 keystore，放置为 `app/release.keystore`，并在构建环境中配置上述环境变量。

---

## 十、Debug 构建与 `dev_profiles.json`（可选）

若仓库**根目录**存在 **`dev_profiles.json`**，在 **debug** 构建合并资源时，会将其复制到 **`app/src/debug/assets/`** 并打进调试包，便于开发配置。

若文件不存在，构建会跳过该步骤，不影响打包。

---

## 十一、常见问题

1. **`SDK location not found`**  
   用 Android Studio 打开项目生成 `local.properties`，或手动创建并设置 `sdk.dir` 指向本机 Android SDK。

2. **JDK 版本报错**  
   使用 Android Studio 内置 JDK，或将 `JAVA_HOME` 设为 **JDK 17+**，再执行 Gradle。

3. **依赖下载失败 / JitPack 超时**  
   检查网络与代理；JitPack 用于如 `sherpa-onnx` 等依赖，需可访问 `https://jitpack.io`。

4. **安装到手机时「已存在签名不同的同名应用」**  
   卸载旧包或使用相同签名；debug 与 release 签名不一致时不能覆盖安装。

5. **只想快速验证修改**  
   优先使用 **`assembleDebug`** + 安装 debug APK，迭代最快。

6. **Google Maven 返回 `429 Too Many Requests`**（例如无法下载 `com.android.tools.build:gradle-common-api`）  
   这是 `https://dl.google.com/dl/android/maven2/` 的**限流或瞬时压力过大**，与源码无关。可稍后重试、换一个网络环境或合规代理；若已通过 Android Studio 成功同步过同一版本 AGP，有时本地 Gradle 缓存已有构件，重试也可能直接通过。

---

## 十二、小结

- **改源码**：在 `app` 模块内按常规 Android 方式修改即可。  
- **出包**：Android Studio 一键 Build APK，或命令行 **`assembleDebug` / `assembleRelease`**。  
- **复杂度**：环境就绪后只是一条命令或一次菜单操作；首次搭建 Android 开发环境会占用大部分时间。

若你希望把本文并入主 `README.md`，可在 README 中增加指向本文件的链接，避免重复维护。
