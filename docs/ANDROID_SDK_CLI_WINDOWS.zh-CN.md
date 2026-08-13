# Windows：仅用 Android SDK Command-line Tools 搭建构建环境

面向 **Auto小二** 仓库：装好 JDK + SDK CLI 后，可在项目根目录执行 `.\gradlew.bat assembleDebug` 打出 APK，无需安装 Android Studio。

---

## 〇、开始前确认

| 项目　　 | 要求　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　 |
| ----------| ------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 操作系统 | Windows 10 / 11（本文命令以 **PowerShell** 为准）　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　|
| JDK　　　| **17 或更高**，用于运行 Gradle / AGP 8.x。终端执行 `java -version` 确认；未安装可从 [Adoptium](https://adoptium.net/) 等渠道安装，并设置 **`JAVA_HOME`**。 |
| 网络　　 | 需访问 Google（下载 SDK 组件与 Gradle 依赖）；若不稳定可能出现超时或 HTTP 429，需重试或更换网络/代理。　　　　　　　　　　　　　　　　　　　　　　　　　　 |
| 本工程　 | `compileSdk = 35`（见 `app/build.gradle.kts`），需安装 **Android 15（API 35）Platform** 及匹配的 **Build-Tools**。　　　　　　　　　　　　　　　　　　　　 |

---

## 一、选定 SDK 根目录

自行选一个**不含中文、不含空格**的路径作为 SDK 根目录，例如：

- `D:\Android\sdk`
- `C:\Android\sdk`

下文用 **`D:\Android\sdk`** 举例；若你使用其它路径，全文替换即可。

后续环境变量：

```powershell
# 当前会话临时生效（永久设置见第四节）
$env:ANDROID_HOME = "D:\Android\sdk"
$env:ANDROID_SDK_ROOT = $env:ANDROID_HOME   # 与 ANDROID_HOME 一致即可
```

---

## 二、下载并解压 Command-line Tools

1. 打开官方页面：[Command line tools only](https://developer.android.com/studio#command-line-tools-only)（页面中的 **Windows** 压缩包，文件名类似 `commandlinetools-win-*_latest.zip`）。
2. 解压后你会得到一个 **`cmdline-tools`** 目录（其内部应有 `bin`、`lib` 等）。
3. **目录结构必须满足**（Google 硬性要求，否则 `sdkmanager` 可能报路径错误）：

   在 **`D:\Android\sdk`** 下创建 **`cmdline-tools\latest`**，把解压得到的 **`cmdline-tools` 里的内容** 放进 **`latest`**，使得存在：

   ```
   D:\Android\sdk\cmdline-tools\latest\bin\sdkmanager.bat
   ```

   **错误示例**：把 `bin` 直接放在 `D:\Android\sdk\cmdline-tools\bin`（缺少 `latest` 这一层）。

---

## 三、安装本工程所需的 SDK 组件

在 **PowerShell** 中（先把 `ANDROID_HOME` 指到你的 SDK 根目录）：

```powershell
$env:ANDROID_HOME = "D:\Android\sdk"
$sdkmanager = Join-Path $env:ANDROID_HOME "cmdline-tools\latest\bin\sdkmanager.bat"

# 查看可安装的 build-tools 版本（若下一步报错可对照这里改版本号）
& $sdkmanager --list | Select-String "build-tools"
```

安装 **platform-tools**、**API 35 平台**、**Build-Tools**（优先安装 35.x；若下列版本不存在，用上一命令输出里的最新 `35.x.x` 替换）：

```powershell
& $sdkmanager --install "platform-tools" "platforms;android-35" "build-tools;35.0.0"
```

若提示 **`build-tools;35.0.0`** 不可用，可改为例如 **`build-tools;35.0.1`** 或 **`build-tools;36.0.0`**（以 `sdkmanager --list` 为准；一般安装与 compileSdk 匹配的 35.x 即可）。

### 接受许可协议

```powershell
& $sdkmanager --licenses
```

对每个提示输入 **`y`** 并回车，直到结束。

---

## 四、持久化环境变量（推荐）

**用户级**环境变量（「设置 → 系统 → 关于 → 高级系统设置 → 环境变量」）：

| 变量名 | 值 |
|--------|-----|
| `ANDROID_HOME` | `D:\Android\sdk` |
| `ANDROID_SDK_ROOT` | 与 `ANDROID_HOME` 相同（可选但建议一致） |

在 **Path** 中追加（按你机器实际路径）：

- `%ANDROID_HOME%\cmdline-tools\latest\bin`
- `%ANDROID_HOME%\platform-tools`

**新开一个 PowerShell** 后执行 `sdkmanager --version` 应能运行。

---

## 五、让 Gradle 找到 SDK（本仓库）

项目根目录下的 **`local.properties`** 已被 `.gitignore` 忽略，应在本机创建，**勿提交到 Git**。

在 **`AutoXiaoer` 仓库根目录** 新建 `local.properties`，内容示例：

```properties
sdk.dir=D\:\\Android\\sdk
```

注意：

- 使用 **正斜杠** 最省事：`sdk.dir=D:/Android/sdk`
- 若写 Windows 反斜杠，在 `local.properties` 里需写成 **`sdk.dir=D\:\\Android\\sdk`** 这种转义形式（与 Gradle/Java Properties 约定一致）

---

## 六、验证构建

```powershell
cd D:\code_project\a_company\AutoXiaoer   # 换成你的仓库路径
.\gradlew.bat assembleDebug
```

成功后 APK 一般在：

`app\build\outputs\apk\debug\`  
文件名形如 **`XiaoEr-<versionName>-debug.apk`**（见 `docs/BUILD_APK.zh-CN.md`）。

---

## 七、常见问题

1. **`SDK location not found`**  
   检查 `local.properties` 的 `sdk.dir` 是否指向**真实存在**的目录，且该目录下有 `platforms\android-35` 等子目录。

2. **`Failed to install the following Android SDK packages ...`**  
   多为网络或磁盘权限；关闭占用该目录的程序后重试，或检查杀毒软件拦截。

3. **HTTP 429（下载 Gradle / Google Maven）**  
   与 SDK CLI 无关，属依赖下载限流；隔一段时间重试或换网络。

4. **是否需要 NDK / CMake？**  
   当前 Auto小二以 Kotlin/Android 为主，一般 **不需要**；若日后引入原生库再按需安装。

---

## 八、与 Android Studio 路线对比

| | 仅 CLI | Android Studio |
|--|--------|----------------|
| 体积 | 小 | 大 |
| SDK 安装 | 手动 `sdkmanager` | 图形界面 |
| `local.properties` | 手写 | 常自动生成 |

完成本文步骤后，构建方式与 `docs/BUILD_APK.zh-CN.md` 中命令行章节一致。
