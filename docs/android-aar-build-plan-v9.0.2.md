# ffmpeg-kit-next v9.0.2 Android AAR 构建方案

> 创建：2026-10-31
> **状态：✅ 已执行完成（2026-11-01 00:22），产物 14,674,893 B，验收全部通过（§6.1）**
> 目标仓库：`D:\3rd-party-projects\ffmpeg-kit-next`（fork `pisces312/ffmpeg-kit-next`，upstream `arthenica/ffmpeg-kit-next`）
> 目标产物：替换 StreamClip 当前使用的 `ffmpeg-kit-8.1.3.aar`（FFmpeg n8.1.3，12,605,249 B）
> 范围：**只产 AAR，不改 StreamClip**。集成另开分支，见 §9。

---

## 1. 结论

可以构建，且比预想省事：**上一轮我们在 8.1.3 AAR 上手工修的 binder 崩溃，上游已自行修复**。

证据（`android/ffmpeg-kit-next-android-lib/src/main/cpp/compat/android/binder.c` 文件头）：

```
This file is based on FFmpeg's compat/android/binder.c at tag n8.1.2, modified so
that it does not shrink the binder threadpool max thread count. Unlike the
standalone ffmpeg CLI, FFmpegKitNext runs inside a host Android app whose
binder threadpool is already started, and shrinking it aborts the process.
```

`setThreadPoolMaxThreadCount` 在本仓库只剩两处注释，无实际调用。因此本次构建**不需要修改任何 C/C++ 源码**，是纯参数构建。

---

## 2. 用户已确认的取舍

| # | 决策 | 影响 |
|---|---|---|
| 1 | **不启用 libopus** | 保持与 8.1.3 AAR 的能力集一致；StreamClip 的 Opus 输出继续走内置实验性编码器（`-strict -2` + 强制 48 kHz 的既有 workaround 保留） |
| 2 | **android-all.jar 预置**，避免构建期 Robolectric 联网下载 | 见 §3.3，GitHub 无此资产，改用 Maven 镜像直下 |
| 3 | **不开 LTO**（`--no-link-time-optimization`） | 与 8.1.3 基线对齐，降低首轮失败面；体积略大，后续可单独试 |
| 4 | **Gradle 用本地缓存 ≥ 8.14.5 的版本**，并同步改 fork 的版本依赖 | 见 §3.2 |

---

## 3. 环境与前置条件（2026-10-31 实测）

### 3.1 编译环境

| 项 | 实测值 | 判定 |
|---|---|---|
| 宿主 | WSL2 Ubuntu 24.04.4 LTS | ✅ 上游脚本仅支持 Linux/macOS |
| CPU / 内存 / 磁盘 | 12 核 / 23 GB / 根分区 897 GB 可用 | ✅ `--jobs=12` |
| NDK | `/home/pisces312/android-ndk-r27d`，`Pkg.Revision = 27.3.13750724` | ✅ 与上游 `android-r27d` profile 一致 |
| Android SDK（Linux 版） | `~/android_sdk`，含 `platforms/android-34`、`build-tools/34.0.0` | ✅ 可在 WSL 内完成 gradle 打包，无需回转 Windows |
| JDK | `/usr/lib/jvm/java-21-openjdk-amd64`（另有 17） | ✅ AGP 8.13.2 要求 17+ |
| 构建工具 | make / autoconf / automake / libtool / pkg-config / cmake / rsync / nasm / yasm / gperf | ✅ 齐备 |
| 缺失工具 | `bison`、`flex`、`autopoint` | ✅ 非必需：FFmpeg/x264/x265/lame 的 configure 均不依赖 |
| 网络 | `git ls-remote github.com/arthenica/FFmpeg` 通；`services.gradle.org` 通；`maven.aliyun.com` 通 | ✅ |

源码获取方式：脚本用 `git clone --depth 1 --branch <tag>`，不会拉 FFmpeg 全历史。

### 3.2 Gradle 版本

本地已有缓存（WSL 侧 `~/.gradle/wrapper/dists`）：

| 版本 | 位置 | 说明 |
|---|---|---|
| **9.4.1**（bin） | `~/.gradle/wrapper/dists/gradle-9.4.1-bin/` | 已解压可用，`gradle --version` 验证通过（Launcher JVM 21.0.12.1） |
| 9.8.1（all） | `~/.gradle/wrapper/dists/gradle-9.8.1-all/` | 含源码包，体积大 |

Windows 侧另有 9.4.1 / 9.6.1，但本次在 WSL 内构建，不参与。

Gradle 缓存中的 AGP 版本：**9.1.1、9.4.1**（无 8.13.2）。

上游 `android/gradle/wrapper/gradle-wrapper.properties` 原值：`gradle-8.14.5-bin`。

> 兼容性结论与最终取值见 §7「实测记录」，以实测为准。

### 3.3 android-all.jar（Robolectric 依赖）

上游 `android.sh` 在打包阶段强制执行：

```bash
./gradlew ffmpeg-kit-next-android-lib:clean \
          ffmpeg-kit-next-android-lib:assembleRelease \
          ffmpeg-kit-next-android-lib:testReleaseUnitTest
```

其中 `testReleaseUnitTest` 会触发 Robolectric，需要 `org.robolectric:android-all-instrumented` 对应 **SDK 34** 的 jar（测试里写死 `@Config(sdk = [34])`）。

**GitHub 上没有这个资产**，`gh` 用不上，实测证据：

| 检查 | 结果 |
|---|---|
| `gh release list -R robolectric/robolectric` | 有 4.16 / 4.17 等 release，但 `assets` 全为空 |
| `gh api repos/robolectric/robolectric/tags` 过滤 `android-all` | 无匹配 tag |
| `gh release list -R robolectric/android-all` | `Could not resolve to a Repository`（该仓库不存在） |
| `gh api orgs/robolectric/repos` | 12 个仓库，无 android-all 相关仓库 |

实际托管在 Maven Central，直链实测可用：

```
https://repo1.maven.org/maven2/org/robolectric/android-all-instrumented/14-robolectric-10818077-i7/
  android-all-instrumented-14-robolectric-10818077-i7.jar   151,142,227 B   HTTP 200
```

阿里云镜像同大小、同 200，实测下载耗时 **14 秒**。

已预置到 WSL 本地 Maven 仓库（Robolectric 会先查此处，命中即不联网）：

```
~/.m2/repository/org/robolectric/android-all-instrumented/14-robolectric-10818077-i7/
  android-all-instrumented-14-robolectric-10818077-i7.jar   151,142,227 B
  android-all-instrumented-14-robolectric-10818077-i7.pom            1,391 B
```

---

## 4. 构建参数

### 4.1 与现有 8.1.3 AAR 的对照

后者参数从 `libavutil.so` 内嵌的 `configuration:` 串反解得到，是事实基准。

| 维度 | 8.1.3 AAR（现状） | v9.0.2 本次 | 说明 |
|---|---|---|---|
| ABI | arm64-v8a | `--arch=arm64-v8a` | next 默认 5 个 ABI 全开，必须显式限定 |
| API level | 24 | `--api-level=26` | 与 StreamClip `minSdk 26` 对齐 |
| GPL | `--enable-gpl --enable-nonfree` | `--enable-gpl` | next 强制 `--enable-version3` 且永不启用 non-free，**许可证比现状更干净** |
| x264 | ✅ | `--enable-lib-x264` | |
| x265 | ✅ | `--enable-lib-x265` | |
| mp3lame | ✅ | `--enable-lib-lame` | next 的 flag 名是 `lame`，落 configure 为 `--enable-libmp3lame` |
| opus | ❌（无） | ❌（不加） | 用户决定不启用 |
| opencore-amr | ❌（无） | ❌（不加） | 8.1.3 的 configure 里本来就没有（历史文档误记为有）；StreamClip 也不产 AMR |
| MediaCodec | `--enable-mediacodec --enable-jni` | `--enable-lib-android-media-codec` | 脚本自动附带 `--enable-jni` |
| zlib | 自动检测启用（`libavcodec.so` 内命中 `inflate`/`pngdec`/`pngenc`） | `--enable-lib-android-zlib` | next 用 `--disable-autodetect`，不显式开就会丢掉 PNG 编解码 |
| 组件范围 | 全量（无 `--disable-everything`） | 同 | 能力面对齐 |
| 优化 | `-Os`，无 LTO，无 `--enable-small` | `--speed` + `--no-link-time-optimization` | `--speed` 用于抵消 next 默认的 `--enable-small`（后者会裁短错误信息） |

### 4.2 完整命令

> ⚠️ 构建目录必须是 WSL 原生盘（`~/build/...`），不能在 `/mnt/d` 上直接编，原因见 §7.5 #1。

```bash
# 一次性：把仓库拷到 WSL 原生盘（详见 §7.5 #1 的 rsync 命令）
cd ~/build/ffmpeg-kit-next
git checkout -b pisces/v9.0.2 v9.0.2          # 已在 fork 上建好工作分支

export ANDROID_SDK_ROOT=/home/pisces312/android-sdk-linux
export ANDROID_NDK_ROOT=/home/pisces312/android-ndk-r27d
export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
export HOST_CC=/usr/bin/gcc                    # WSL 无 clang，必须显式指定，见 §7.5 #2

# android.sh 不会替 gradle 配 SDK 路径，且该文件已被 .gitignore 忽略
printf 'sdk.dir=/home/pisces312/android-sdk-linux\nndk.dir=/home/pisces312/android-ndk-r27d\n' \
  > android/local.properties

./android.sh \
  --api-level=26 \
  --arch=arm64-v8a \
  --jobs=12 \
  --speed \
  --no-link-time-optimization \
  --enable-gpl \
  --enable-lib-x264 --enable-lib-x265 --enable-lib-lame \
  --enable-lib-android-media-codec \
  --enable-lib-android-zlib
```

### 4.3 产物路径

`android.sh` 的实际落点是 Gradle 输出目录（**不是** `prebuilt/bundle-android-aar-26-maven/`，那条路径只在跑 maven publish 任务时才会生成，`bundleReleaseAar` 不会产出 POM）：

```
android/ffmpeg-kit-next-android-lib/build/outputs/aar/ffmpeg-kit-next-release.aar
```

已归档副本：

```
D:\3rd-party-projects\ffmpeg-kit-next\prebuilt\ffmpeg-kit-next-9.0.2-arm64.aar
```

AAR 内嵌 `proguard.txt`（`-keep class com.arthenica.ffmpegkit.FFmpegKitConfig { native <methods>; ... }` 等 JNI 保号规则），集成方无需自己再写。

需要 POM 里的依赖声明时直接从源码读，`android/ffmpeg-kit-next-android-lib/build.gradle`：

```gradle
api         'com.arthenica:smart-exception-java:0.2.1'
compileOnly 'androidx.annotation:annotation:1.9.1'
```

---

## 5. 执行步骤与耗时

| 阶段 | 内容 | 预估 | 实测（2026-10-31 ~ 11-01） |
|---|---|---|---|
| 0 | 建分支、写 `local.properties`、预热 gradle、预置 android-all | 10–15 min | 约 40 min（额外踩了镜像假死、Windows 版 SDK、Linux build-tools 三个坑，见 §7） |
| 1 | 下载源码：FFmpeg `n9.0.2`（arthenica/FFmpeg）、x264 `b35605a`、x265 `4.2`、lame `RELEASE__3_100`、libiconv `v1.17`、cpu-features `v0.11.0` | 3–8 min | 每库浅克隆 20–40 s |
| 2 | 外部库编译 → FFmpeg configure + make → `ndk-build` JNI | 35–55 min | 约 30 min（12 并行、单 ABI） |
| 3 | Gradle `assembleRelease` 出 AAR | 5–10 min | 约 2 min |
| 4 | 验收 + 存档 | 5 min | 8 min |

**从第一个库克隆（23:34:21）到 AAR 产出（00:14:53），单次成功构建 41 分钟**；中途因 libiconv man page 与 `--disable-lto` 失败重试 2 次，按墙钟算约 1 小时。

日志：`build.log`（仓库根），失败时先看这个文件尾部。

---

## 6. 验收清单

| # | 检查项 | 判定标准 |
|---|---|---|
| 1 | ABI 唯一 | `unzip -l *.aar` 的 `jni/` 下只有 `arm64-v8a/` |
| 2 | `.so` 清单 | 10 个：`libav{codec,device,filter,format,util}`、`lib{swresample,swscale}`、`libffmpegkit`、`libffmpegkit_abidetect`、`libc++_shared` |
| 3 | configure 串（从 `libavutil.so` 反解） | 含 `--enable-gpl`、`--enable-libx264`、`--enable-libx265`、`--enable-libmp3lame`、`--enable-mediacodec`、`--enable-jni`、`--enable-zlib`；不含 `--enable-libopus`、`--enable-nonfree` |
| 4 | MediaCodec 编解码器 | `libavcodec.so` 内命中 `h264_mediacodec`、`hevc_mediacodec` |
| 5 | **binder 线程池** | `strings -n 3 libffmpegkit.so \| grep -c setThreadPoolMaxThreadCount` **必须为 0**（沿用 8.1.3 升级时固化的必查项） |
| 6 | 版本串 | `libffmpegkit.so` 内为 `9.0.2` |
| 7 | AAR 体积 | 记录实际值，与 8.1.3 的 12,605,249 B 对比 |
| 8 | sha256 | 记录，供 StreamClip 集成时核对 |

### 6.1 实测结果（2026-11-01 00:22，全部通过）

| # | 检查项 | 实测 |
|---|---|---|
| 1 | ABI 唯一 | `jni/` 下只有 `arm64-v8a` ✅ |
| 2 | `.so` 清单 | 10 个，与 8.1.3 逐一对应 ✅ |
| 3 | configure 串 | 见下方实录 ✅ |
| 4 | MediaCodec | `h264_mediacodec`、`hevc_mediacodec` 各命中 1 次 ✅ |
| 5 | binder 线程池 | `libffmpegkit/libavcodec/libavutil/libavformat` 计数均为 **0** ✅ |
| 6 | 版本串 | `libffmpegkit.so` 内 `9.0.2` + `n9.0.2` ✅ |
| 7 | 体积 | **14,674,893 B**（8.1.3 为 12,605,249 B，+2.07 MB / +16.4%） |
| 8 | sha256 | `13231cef870922eed307b722f4ef7d34a0b660edcd8acdf4713a88e3b20a3051` |

`.so` 逐个大小：

```
18603304 libavcodec.so        4246512 libavfilter.so    749040 libavutil.so
 3640632 libavformat.so       1794776 libc++_shared.so   567920 libffmpegkit.so
   63072 libavdevice.so         98680 libswresample.so  1225648 libswscale.so
   37960 libffmpegkit_abidetect.so
```

configure 实录（从 `libavutil.so` 反解，已按 `--` 折行过滤只看关键项）：

```
--enable-version3 --enable-cross-compile --enable-pic --enable-jni --enable-optimizations
--enable-swscale --enable-shared --enable-pthreads --enable-v4l2-m2m
--enable-libmp3lame --enable-iconv --enable-libx264 --enable-libx265 --enable-zlib
--enable-mediacodec --enable-gpl
--disable-autodetect --disable-static --disable-debug --disable-programs --disable-doc
--disable-sdl2 --disable-openssl --disable-vaapi --disable-vdpau --disable-nvenc
--disable-mediacodec  ← 无（正确）
--enable-libopus       ← 无（按要求不加）
--enable-nonfree       ← 无（next 永不启用）
--enable-lto           ← 无（LTO 已关）
```

体积增量的构成：libavcodec 18.6 MB（8.1.3 约 15 MB，FFmpeg 9.0 本身变大）+ libiconv（lame 依赖链带出，约 1 MB）+ zlib 静态链接。

### 6.2 已确认的集成前提

| 项 | 实测结论 |
|---|---|
| 包名 / 类名 | 仍是 `com.arthenica.ffmpegkit`，63 个 class，`FFmpegKit` / `FFmpegKitConfig` / `FFmpegSession` / `FFprobeKit` 全在 |
| 关键方法 | `FFmpegKit.execute(String)`、`executeWithArguments(String[])`、`FFmpegKitConfig.getFFmpegVersion()`、`ffmpegExecute()` 签名与 8.1.3 一致 |
| minSdk / compileSdk | **26** / 34（比预判的 24 更省事，与 StreamClip 的 minSdk 26 完全一致） |
| 传递依赖 | `api com.arthenica:smart-exception-java:0.2.1`，`files(...aar)` 不传递，集成时须显式声明 |
| JNI 保号规则 | AAR 自带 `proguard.txt`，无需额外配置 |
| binder 回归 | next 上游已自行去掉 `setThreadPoolMaxThreadCount`，**本次构建未改任何 native 源码** |

---

## 7. 实测记录

> 构建过程中逐条补充，作为下次升级的对照基线。

### 7.1 Gradle 兼容性（已实测通过）

**结论：AGP 8.13.2 可以在 Gradle 9.4.1 上正常工作**，无需降级、无需改 AGP 版本。

| 验证项 | 结果 |
|---|---|
| `gradle help`（插件 apply + 配置阶段） | BUILD SUCCESSFUL in 44s |
| `./gradlew ffmpeg-kit-next-android-lib:assembleRelease`（资源处理 + Kotlin 编译 + 打包，28 个 task） | BUILD SUCCESSFUL in 44s |
| 唯一告警 | `Deprecated Gradle features were used ... incompatible with Gradle 10`（不影响构建，是 Gradle 10 的迁移提示） |

`android/gradle/wrapper/gradle-wrapper.properties` 已改：

```diff
-distributionUrl=https\://services.gradle.org/distributions/gradle-8.14.5-bin.zip
+distributionUrl=https\://services.gradle.org/distributions/gradle-9.4.1-bin.zip
```

选择 9.4.1 而非更高版本的原因：本机 WSL 已有解压好的 9.4.1，且与本地缓存的 AGP 9.x 同代；9.8.1 只有 `-all` 包，无必要。

> 缓存命中技巧：wrapper 的目录名是 distributionUrl 的 MD5（base36）。本机原有目录 `45w7kj7s8jzqenl33wrwa0aoj` 对应另一个 URL，改回官方 URL 后 hash 变为 `arn2x92ynaizyzdaamcbpbhtj`。把已解压的 `gradle-9.4.1` 复制到新 hash 目录并 `touch gradle-9.4.1-bin.zip.ok`，wrapper 即视为已下载，不会联网。

### 7.2 网络：必须走镜像，直连会假死

首次 `gradle help` **跑了 18 分钟零进展**（Java 进程累计 CPU 仅 30 秒、缓存无新文件、保持 12 条到 Cloudflare `104.18.18.12:443` 的连接）。这不是 Gradle 或 AGP 的问题，是 Maven Central / Google Maven 直连被卡。

注入 `~/.gradle/init.d/aliyun-mirror.gradle`（只影响本机，不入库）：

```groovy
def applyMirrors = { repos ->
    repos.clear()
    repos.mavenLocal()
    repos.maven { url = "https://maven.aliyun.com/repository/google" }
    repos.maven { url = "https://maven.aliyun.com/repository/public" }
    repos.maven { url = "https://maven.aliyun.com/repository/gradle-plugin" }
}

settingsEvaluated { settings ->
    applyMirrors(settings.pluginManagement.repositories)
}

gradle.beforeProject { project ->
    applyMirrors(project.buildscript.repositories)
    applyMirrors(project.repositories)
}
```

镜像优先级靠 `clear()` + 上游 build.gradle 后追加实现；命中不到时仍会回落官方仓库。改完同一命令 **44 秒**完成。

阿里云镜像已确认含：`com.android.tools.build:gradle:8.13.2`、`org.jetbrains.dokka:dokka-gradle-plugin:1.9.20`（均 HTTP 200）。

### 7.3 WSL 里的 `~/android_sdk` 是 Windows 版 SDK 的软链

```
~/android_sdk -> /mnt/d/dev/android_sdk/     (build-tools 里全是 aapt.exe / apksigner.bat)
```

直接拿它做 `sdk.dir` 会失败：

```
Build-tool 35.0.0 is missing AAPT at /mnt/d/dev/android_sdk/build-tools/35.0.0/aapt
Installed Build Tools revision 35.0.0 is corrupted.
```

处置：建独立 Linux SDK，只补 build-tools，platforms/platform-tools 软链复用（jar 与 adb 跨平台通用）：

```bash
curl -sSL -o bt35.zip https://dl.google.com/android/repository/build-tools_r35_linux.zip   # 60 MB
unzip -q bt35.zip -d bt35                       # 解压出 android-15/
mkdir -p ~/android-sdk-linux/build-tools
mv bt35/android-15 ~/android-sdk-linux/build-tools/35.0.0
ln -sfn /mnt/d/dev/android_sdk/platforms     ~/android-sdk-linux/platforms
ln -sfn /mnt/d/dev/android_sdk/platform-tools ~/android-sdk-linux/platform-tools
```

注意 Google 的命名规则在新版本上从连字符改成了下划线：`build-tools_r34-linux.zip` 可用，`build-tools_r35-linux.zip` 是 404，正确名是 `build-tools_r35_linux.zip`。可用仓库清单核对：

```bash
curl -s https://dl.google.com/android/repository/repository2-3.xml | grep -o "build-tools_r3[0-9][^\"<]*linux[^\"<]*" | sort -u
```

选择 35.0.0 而非 34.0.0 的原因：AGP 8.13.2 默认请求 build-tools 35.0.0（失败信息里就是它），装同版本可避免再改 `build.gradle`。

### 7.4 构建耗时

（待填）

### 7.5 踩坑

**#1 不能停在 `/mnt/d` 上构建（关键）**

在 `/mnt/d`（WSL 的 9p/DrvFs 挂载）上执行时，FFmpeg 源码 clone 在 checkout 阶段崩溃：

```
fatal: sha1 file '.../src/ffmpeg/.git/index.lock' write error: Bad file descriptor
warning: Clone succeeded, but checkout failed.
ERROR: Failed to clone https://github.com/arthenica/FFmpeg -> n9.0.2
```

1 万多个文件并发落盘时 9p 无法保证 fd 语义。**解法：仓库整体放到 WSL 原生 ext4 盘构建**，产物再拷回 D 盘。

```bash
rsync -a --exclude "/src/" --exclude "/android/build/" --exclude "/android/.gradle/" \
         --exclude "/prebuilt/" --exclude "/build.log" --exclude "/.tmp/" \
  /mnt/d/3rd-party-projects/ffmpeg-kit-next/ ~/build/ffmpeg-kit-next/
```

注意 exclude 必须写成 `/src/` 而不能写 `src/`——后者会连带排除 Android 模块自己的 `src/main/cpp`，导致 JNI 源码整目录丢失（`git status` 一片 `D`）。

**#2 HOST_CC 必须显式给**

`scripts/function-android.sh:1172` 是 `set_host_cc "$(command -v clang)"`，硬编码找 `clang`。WSL 未装 clang（只有 gcc 13.3）时直接报：

```
(*) No C compiler found for the build machine. Install clang or set HOST_CC
```

`set_host_cc` 的逻辑是 `${HOST_CC:-$1}`，所以 `export HOST_CC=/usr/bin/gcc` 即可，不必装 clang。注释里也写明一旦 export 后续所有库都会沿用。

**#3 android-all.jar：GitHub 没有，gh 用不上**（详见 §3.3）

**#4 构建摘要里会多出一个 `libiconv`**

```
Libraries: android-zlib, android-media-codec, lame, libiconv, x264, x265
```

它来自 `lame` 的依赖链（详见 #6），不是显式指定、也不是 zlib 带的。8.1.3 的 configure 里没有 `--enable-iconv`，属本次新增能力，体积增加约 1 MB，不影响 StreamClip 现有功能。

**#5 Gradle 首次配置会「假死」18 分钟**（详见 §7.2）

看起来像卡死，其实是 Maven Central 连接挂住。判断方法：看 Java 进程累计 CPU 时间——18 分钟只用了 30 秒 CPU、缓存无新文件、有稳定的 Cloudflare 443 连接，就是网络问题，不要怀疑 AGP 版本。

**#6 `lame` 会通过依赖链拉进 `libiconv`，libiconv 有三个连环坑**

`scripts/function.sh` 的 `lame)` 分支里有 `set_virtual_library "libiconv" $2`，所以启用 MP3 编码就必然编 libiconv。

**坑 6a：man page 生成失败。** `Makefile.devel` 的默认目标包含 `man/iconv.1.html`，需要 `groff -Thtml`：

```
groff: fatal error: cannot load 'DESC' description file for device 'html'
make: *** [Makefile.devel:218: man/iconv.1.html] Error 3
```

Ubuntu 只装了 `groff-base`，没有 html device。装完整 `groff` 需要 sudo，改用 stub（man page 对 Android 静态库无意义）：

```bash
mkdir -p ~/.local/bin
cat > ~/.local/bin/groff <<'EOF'
#!/bin/sh
# Host has groff-base only (no html device). Must emit at least one line:
# the Makefile pipes us through `grep -v`, which returns 1 on empty input.
echo "<!-- groff stub: man page intentionally empty -->"
EOF
chmod +x ~/.local/bin/groff
```

`~/.local/bin` 在 PATH 靠前位置，不需要改仓库脚本。

**坑 6b：stub 不能什么都不输出。** 第一版 stub 是 `exit 0`，结果卡在下一行：

```
groff -mandoc -Thtml man/iconv.1 > man/iconv.1.html-t
grep -v '<!-- CreationDate' < man/iconv.1.html-t > man/iconv.1.html   ← Error 1
```

`grep -v` 在空输入上返回 1。stub 必须输出至少一行**不被** `-v` 过滤掉的内容。

**坑 6c：重跑必须带 `--reconf-lib-libiconv`。** 脚本在多轮构建中会执行 `make distclean`，它删掉 autogen 生成的 `libcharset/configure`；而 `libiconv.sh` 的重生成条件是 `[[ ! -f configure ]] || RECONF`，主 `configure` 还在，于是跳过 autogen 直接失败：

```
configure: WARNING: no configuration information is in libcharset
make[1]: *** No rule to make target 'all'.  Stop.
make: *** [Makefile:41: lib/localcharset.h] Error 2
```

arthenica/libiconv 的 git 仓库**不携带**预生成的 `configure`（首次构建会自动跑 autogen），所以只要中途失败过，后续每次重跑都要加 `--reconf-lib-libiconv`。

> 副作用：`libiconv` 是 8.1.3 AAR 没有的组件（旧 kit 的 lame 没接 iconv），本次会多出约 1 MB，属新增能力而非回归。

**#7 上游 bug：FFmpeg 9.x 已移除 `--disable-lto`，但 `--no-link-time-optimization` 仍在传它**

`scripts/android/ffmpeg.sh` 里原来是：

```bash
if [[ -z ${NO_LINK_TIME_OPTIMIZATION} ]]; then
  DEBUG_OPTIONS="--disable-debug --enable-lto"
else
  DEBUG_OPTIONS="--disable-debug --disable-lto"   # ← FFmpeg 9.x 不认这个参数
fi
```

FFmpeg 9.x 的 configure 拒绝该选项，构建直接停在 configure 阶段：

```
Unknown option "--disable-lto".
See ./configure --help for available options.
```

FFmpeg 9.x 里 LTO 默认就是关的（只有 `--enable-lto`，没有对应的 disable），所以「关 LTO」分支只需要不传任何 LTO 参数。已在 fork 的 `pisces/v9.0.2` 分支修正：

```diff
   if [[ -z ${NO_LINK_TIME_OPTIMIZATION} ]]; then
     DEBUG_OPTIONS="--disable-debug --enable-lto"
   else
-    DEBUG_OPTIONS="--disable-debug --disable-lto"
+    DEBUG_OPTIONS="--disable-debug"
   fi
```

> 这是 next v9.0.2 自身的缺陷，不是配置错误——任何用 `--no-link-time-optimization` 的人都会命中。可考虑给上游提 issue/PR。

**#8 后台跑构建时，`build.log` 可能停在 `testReleaseUnitTest`，但 AAR 其实已经好了**

用 `nohup ./android.sh ... &` 挂后台后，一旦外层 `wsl.exe -e bash -lc` 会话结束，整个进程组会被 WSL 回收。而 `android.sh` 的任务顺序是：

```
... :bundleReleaseAar  →  :assembleRelease  →  :testReleaseUnitTest
```

AAR 在 `assembleRelease` 阶段就已落盘，**测试是最后一步、纯附加**。所以看到日志末尾卡在：

```
> Task :ffmpeg-kit-next-android-lib:testReleaseUnitTest
Picked up JAVA_TOOL_OPTIONS: ''-Duser.language=en -Duser.country=US
```

先别当成失败，直接看产物：

```bash
ls -l android/ffmpeg-kit-next-android-lib/build/outputs/aar/ffmpeg-kit-next-release.aar
```

产物存在且完整（含 `META-INF/ R.txt classes.jar jni/ proguard.txt res/`）即视为构建成功，测试单独补跑：

```bash
./gradlew ffmpeg-kit-next-android-lib:testReleaseUnitTest --console=plain
```

想要构建不被会话回收，用 `setsid` 脱离进程组，或把整个 `android.sh` 放到 `tmux` 里跑。

---

## 8. 风险与已知差异

| # | 项 | 影响 | 处置 |
|---|---|---|---|
| 1 | FFmpeg 9.0 movenc 时间基变更 | 无损剪辑输出行为可能变化 | 集成阶段回归项，本次不涉及 |
| 2 | next #51：MediaCodec 编码产物异常（作者称属上游或设备 codec，未定论） | 硬编压缩是 StreamClip 核心功能 | 集成阶段真机必测 |
| 3 | next #77：9.0.0 取消不生效，9.0.1 已修 | 取消是核心功能 | 集成阶段实测验证 |
| 4 | AAR 为 Kotlin 实现，POM 声明 `smart-exception-java:0.2.1` | 用 `implementation(files(...aar))` 不会传递依赖 → 运行期 `NoClassDefFoundError` | 集成时在 StreamClip 显式加该依赖 |
| 5 | AAR `minSdk **26**` / `compileSdk 34`（实测，原预判 24 有误），app 为 26 / 37 | 无影响，minSdk 恰好一致 | 无 |
| 6 | `--enable-version3` 强制、无 nonfree | 许可表述需更新 | 集成时微调 `GPL_COMPLIANCE.md`（现状写的是 nonfree 组合） |
| 7 | `android.sh` 强制跑 `testReleaseUnitTest` | Robolectric 失败会让整个脚本 exit 1；后台跑时进程被回收也会停在测试步 | **AAR 不受影响**（见 §7.5 #8）；需要测试结果时单独补跑 gradle 任务 |
| 8 | AAR 体积 +2.07 MB（+16.4%） | APK 会同步变大 | 增量来自 FFmpeg 9.0 本体 + libiconv + zlib，非配置失误 |

---

## 9. 后续：StreamClip 集成（本次不做）

预判改动面，供评估"大改动"的量级：

| 文件 | 改动 |
|---|---|
| `app/build.gradle.kts` | AAR 路径换成 `libs/ffmpeg-kit-next-9.0.2-arm64.aar`；新增 `implementation("com.arthenica:smart-exception-java:0.2.1")` |
| `util/OssComponents.kt` | 原生组件版本改为 FFmpeg 9.0.2 / FFmpegKitNext 9.0.2；x264 `b35605a`、x265 4.2、LAME 3.100、libiconv 1.17、cpu_features 0.11.0 |
| `GPL_COMPLIANCE.md` | 来源改为 ffmpeg-kit-next v9.0.2；去掉 `--enable-nonfree` 相关表述，新增 libiconv(LGPL-2.1) 与 zlib |
| `audio/AudioEncoder.kt` | 无强制改动（opus 仍走内置编码器），既有 workaround 保留 |

Kotlin 调用代码**预期零改动**：`com.arthenica.ffmpegkit` 包名与 8 类 API（`executeAsync`、`cancel`、`FFprobeKit.execute`、`ReturnCode.isSuccess`、`session.returnCode/sessionId`、`FFmpegKitConfig.getFFmpegVersion`、`StatisticsCallback`、`LogCallback`）在 v9.0.2 全部保留（§6.2 已用 javap 逐个核对）。唯一不兼容项是 `saf:` → `ffkitsaf:`，但 StreamClip 经 `FileUtils` 转真实路径，未使用该协议。

### 9.1 集成时必测项

1. 4K HDR `hevc_mediacodec` 压缩 —— next #51 报告过硬编产物异常
2. 取消正在执行的任务 —— next #77（9.0.0 取消不生效，9.0.1 修）
3. 无损剪辑输出的时长/时间基 —— FFmpeg 9.0 movenc 时间基变更
4. `FFmpegKitConfig.getFFmpegVersion()` 返回 `9.0.2`（关于页版本显示依赖它）
5. 音频编码 MP3/FLAC/M4A/WAV 四项回归（libiconv 接入后 lame 行为可能变化）
