# 第14章 CI/CD 与发布流程

> 项目通过 GitHub Actions 实现自动化构建与发布。
> pr.yaml 在 PR 时自动测试 + 五平台 canary 构建。
> release.yaml 基于 Git tag 触发正式发布：构建 → 签名 → 创建 GitHub Release。

---

## 14.1 两个 Workflow

| Workflow | 文件 | 触发条件 | 产物 |
|----------|------|----------|------|
| PR Workflow | `.github/workflows/pr.yaml` | PR 打开/同步 + 手动触发 | canary 版本（Artifacts） |
| Release Workflow | `.github/workflows/release.yaml` | git push tag + 手动触发 | 正式版本（GitHub Release） |

---

## 14.2 核心文件清单

| 文件 | 职责 |
|------|------|
| `.github/workflows/pr.yaml` | PR 工作流：测试 + 分析 + 五平台 canary 构建 |
| `.github/workflows/release.yaml` | 发布工作流：测试 + 分析 + 五平台构建 + 签名 + GitHub Release |
| `.github/ISSUE_TEMPLATE/bug.yml` | Bug 报告模板 |
| `.github/ISSUE_TEMPLATE/other.yml` | 其他 Issue 模板 |
| `android/app/build.gradle` | Android 构建配置 |
| `pubspec.yaml` (msix_config) | Windows MSIX 配置 |
| `assets/linux/DEBIAN/` | Linux .deb 打包配置 |

---

## 14.3 PR 工作流

> 源码：`.github/workflows/pr.yaml`

```
触发条件：PR opened/synchronize/reopened/ready_for_review + workflow_dispatch
    ↓
┌─────────────────────────────────────────────────────────────────┐
│  质量门禁（并行）                                                 │
│  ├── flutter-test      → flutter test                           │
│  └── flutter-analyze   → flutter analyze --fatal-warnings       │
└─────────────────────────────────────────────────────────────────┘
    ↓ 全部通过
┌─────────────────────────────────────────────────────────────────┐
│  五平台构建（并行）                                               │
│  ├── android  → flutter build apk --split-per-abi (arm64)       │
│  │             → Kazumi_android_canary.apk                       │
│  ├── windows  → flutter build windows → Compress-Archive        │
│  │             → Kazumi_windows_canary.zip                       │
│  ├── ios      → flutter build ios --no-codesign                 │
│  │             → Kazumi_ios_canary_no_sign.ipa                   │
│  ├── macos    → flutter build macos → create-dmg                │
│  │             → Kazumi_macos_canary.dmg                         │
│  └── linux    → flutter build linux → tar.gz + dpkg-deb         │
│                 → Kazumi_linux_canary.tar.gz + .deb              │
└─────────────────────────────────────────────────────────────────┘
    ↓
产物：上传到 GitHub Actions Artifacts（不发布）
```

### 手动触发选项

```yaml
workflow_dispatch:
  inputs:
    run_android: true/false   # 单独触发 Android 构建
    run_windows: true/false   # 单独触发 Windows 构建
    run_ios: true/false       # 单独触发 iOS 构建
    run_macos: true/false     # 单独触发 macOS 构建
    run_linux: true/false     # 单独触发 Linux 构建
    signpath_sign: true/false # 是否签名 Windows 构建
```

不选任何平台 → 全部构建。选了某个 → 只构建该平台。

---

## 14.4 发布工作流

> 源码：`.github/workflows/release.yaml`

```
触发条件：git push tag（如 v2.1.4）+ workflow_dispatch
    ↓
┌─────────────────────────────────────────────────────────────────┐
│  质量门禁（并行）                                                 │
│  ├── flutter-test      → flutter test                           │
│  └── flutter-analyze   → flutter analyze --fatal-warnings       │
└─────────────────────────────────────────────────────────────────┘
    ↓ 全部通过
┌─────────────────────────────────────────────────────────────────┐
│  四平台构建（并行）                                               │
│  ├── android  → APK (arm64)                                     │
│  ├── ios      → IPA (无签名)                                     │
│  ├── macos    → DMG                                             │
│  └── linux    → tar.gz + .deb                                   │
└─────────────────────────────────────────────────────────────────┘
    ↓ 四平台全部完成
┌─────────────────────────────────────────────────────────────────┐
│  Windows 构建（串行，依赖四平台）                                  │
│  ├── 1. flutter build windows → ZIP                             │
│  ├── 2. SignPath 签名 ZIP                                       │
│  ├── 3. 解压签名 ZIP → 替换未签名文件                             │
│  ├── 4. dart run msix:create → MSIX                             │
│  └── 5. SignPath 签名 MSIX                                      │
└─────────────────────────────────────────────────────────────────┘
    ↓ Windows 完成
┌─────────────────────────────────────────────────────────────────┐
│  发布（release job）                                             │
│  ├── 1. 下载所有平台产物                                         │
│  ├── 2. Android APK 签名                                        │
│  └── 3. 创建 GitHub Release + 上传所有产物                       │
└─────────────────────────────────────────────────────────────────┘
```

### 为什么 Windows 串行？

```
四平台并行构建（android/ios/macos/linux）
    ↓ 全部完成（needs: [flutter-build-android, flutter-build-ios, ...]）
Windows 构建
    ↓ 完成
release job（needs: [flutter-build-windows]）
```

1. Windows 签名需要下载已构建的产物
2. release job 需要下载所有产物统一发布
3. 所以 Windows 是瓶颈，必须等四平台完成

---

## 14.5 编译时注入的 Secrets

```yaml
env:
  DANDANAPI_APPID: ${{ secrets.DANDANAPI_APPID }}  # 弹弹Play AppID
  DANDANAPI_KEY: ${{ secrets.DANDANAPI_KEY }}      # 弹弹Play Key
  KAZUMI_APPID: ${{ secrets.KAZUMI_APPID }}        # Kazumi 镜像 AppID
  KAZUMI_KEY: ${{ secrets.KAZUMI_KEY }}            # Kazumi 镜像 Key
```

通过 `--dart-define` 注入，编译时嵌入二进制：

```bash
flutter build apk --split-per-abi \
  --dart-define=DANDANAPI_APPID=$DANDANAPI_APPID \
  --dart-define=DANDANAPI_KEY=$DANDANAPI_KEY \
  --dart-define=KAZUMI_APPID=$KAZUMI_APPID \
  --dart-define=KAZUMI_KEY=$KAZUMI_KEY
```

Dart 侧读取：

```dart
const Map<String, String> dandanCredentials = {
    'id': String.fromEnvironment('DANDANAPI_APPID'),
    'value': String.fromEnvironment('DANDANAPI_KEY'),
};
```

---

## 14.6 签名流程

### Android APK 签名

```yaml
# release job 中
- name: Sign APK
  uses: filippoLeporati93/android-release-signer@v1
  with:
    releaseDirectory: build/unsigned
    signingKeyBase64: ${{ secrets.SIGNING_KEY_BASE64 }}  # keystore 文件 base64
    alias: ${{ secrets.KEY_ALIAS }}                       # key alias
    keyStorePassword: ${{ secrets.KEY_STORE_PASSWORD }}   # keystore 密码
  env:
    BUILD_TOOLS_VERSION: "34.0.0"
```

### Windows ZIP 签名（SignPath）

```yaml
- name: sign windows zip
  uses: signpath/github-action-submit-signing-request@v1.1
  with:
    api-token: '${{ secrets.SIGNPATH_API_TOKEN }}'
    organization-id: 'fa047255-4772-4be1-b14f-5cfa62635877'
    project-slug: 'Kazumi'
    signing-policy-slug: 'release-signing'
    artifact-configuration-slug: 'Packet'
    github-artifact-id: '${{ steps.unsigned-windows-zip-artifacts.outputs.artifact-id }}'
    wait-for-completion: true
    output-artifact-directory: 'build/windows/zip_signed_output'
```

### Windows MSIX 签名（SignPath）

```yaml
# 1. 构建未签名 MSIX
- name: Build unsigned msix
  run: dart run msix:create

# 2. 签名 MSIX
- name: sign windows msix
  uses: signpath/github-action-submit-signing-request@v1.1
  with:
    signing-policy-slug: 'release-signing'
    artifact-configuration-slug: 'MSIX'
    github-artifact-id: '${{ steps.unsigned-windows-msix-artifacts.outputs.artifact-id }}'
    wait-for-completion: true
    output-artifact-directory: 'build/windows/msix_signed_output'
```

### MSIX 配置

> 源码：`pubspec.yaml` (msix_config 部分)

```yaml
msix_config:
  display_name: Kazumi
  publisher: CN=SignPath Foundation, O=SignPath Foundation, L=Lewes, S=Delaware, C=US
  logo_path: assets/images/logo/logo_rounded.png
  sign_msix: false           # 本地不签名（CI 中由 SignPath 签名）
  install_certificate: false
  build_windows: false       # 不自动构建（CI 中手动调用）
```

---

## 14.7 各平台构建详解

### Android

```yaml
runs-on: ubuntu-latest
steps:
  - setup-java (JDK 17)
  - flutter pub get
  - flutter build apk --split-per-abi  # 只构建 arm64-v8a
  - cp app-arm64-v8a-release.apk Kazumi_android_${tag}.apk
```

- 只构建 arm64-v8a 架构（覆盖绝大多数现代 Android 设备）
- `--split-per-abi` 按 ABI 分包，减小体积

### Windows

```yaml
runs-on: windows-latest
steps:
  - choco install yq                    # 安装 yq 工具
  - flutter pub get
  - flutter build windows
  - Compress-Archive → ZIP              # 便携版
  - SignPath 签名 ZIP                   # 代码签名
  - dart run msix:create → MSIX         # Microsoft Store 格式
  - SignPath 签名 MSIX                  # MSIX 签名
```

### iOS

```yaml
runs-on: macos-latest
steps:
  - flutter pub get
  - flutter build ios --release --no-codesign  # 无签名构建
  - mkdir Payload
  - cp -R Runner.app Payload/Runner.app
  - codesign --force --sign - --preserve-metadata=identifier,entitlements *.framework
  - zip -q -r Kazumi_ios_${tag}_no_sign.ipa Payload
```

- 无签名（需要 Apple 开发者账号才能签名）
- 用户需要通过侧载安装（AltStore、Sideloadly 等）

### macOS

```yaml
runs-on: macos-latest
steps:
  - flutter pub get
  - flutter build macos --release
  - npm install --global create-dmg
  - create-dmg build/macos/Build/Products/Release/Kazumi.app
  - mv Kazumi*.dmg Kazumi_macos_${tag}.dmg
```

- 使用 `create-dmg` 创建 DMG 安装包
- 无 Apple 公证（需要开发者账号）

### Linux

```yaml
runs-on: ubuntu-latest
steps:
  - sudo apt-get install -y clang cmake libgtk-3-dev ...  # 大量依赖
  - flutter pub get
  - flutter build linux
  - tar -zcvf Kazumi_linux_${tag}_amd64.tar.gz            # 便携版
  - dpkg-deb --build Kazumi_linux_${tag}_amd64             # .deb 包
```

- 构建两种格式：tar.gz（便携）+ .deb（Debian 系安装）
- .deb 包含 postinst/postrm 脚本和 .desktop 文件

---

## 14.8 GitHub Actions 关键概念

| 概念 | 说明 | 示例 |
|------|------|------|
| `on` | 触发条件 | `push: tags: ["*"]` |
| `jobs` | 并行任务 | `flutter-test` + `flutter-analyze` |
| `needs` | 依赖关系 | `needs: [flutter-test, flutter-analyze]` |
| `steps` | 串行步骤 | `checkout` → `pub get` → `build` |
| `runs-on` | 运行环境 | `ubuntu-latest` / `windows-latest` / `macos-latest` |
| `secrets` | 加密变量 | `SIGNING_KEY_BASE64` / `SIGNPATH_API_TOKEN` |
| `artifacts` | job 间产物传递 | `upload-artifact` / `download-artifact` |
| `--dart-define` | 编译时注入 | `--dart-define=KAZUMI_APPID=xxx` |
| `env` | 环境变量 | `DANDANAPI_APPID: ${{ secrets.DANDANAPI_APPID }}` |

---

## 14.9 发布产物

| 平台 | 格式 | 签名 | 说明 |
|------|------|------|------|
| Android | `.apk` (arm64) | ✅ android-release-signer | 仅 arm64 |
| Windows | `.zip` | ✅ SignPath | 便携版 |
| Windows | `.msix` | ✅ SignPath | Microsoft Store 格式 |
| macOS | `.dmg` | ❌ | 无 Apple 公证 |
| iOS | `.ipa` | ❌ | 无签名，需侧载 |
| Linux | `.tar.gz` | ❌ | 便携版 |
| Linux | `.deb` | ❌ | Debian 系安装包 |

---

## 14.10 发布流程示例

```bash
# 1. 更新版本号
# pubspec.yaml: version: 2.1.4+120

# 2. 提交
git add .
git commit -m "release: v2.1.4"
git push

# 3. 打 tag
git tag v2.1.4
git push origin v2.1.4

# 4. GitHub Actions 自动触发 release.yaml
#    → 测试 → 构建五平台 → 签名 → 创建 GitHub Release

# 5. 去 GitHub Releases 页面查看发布结果
```

---

## 14.11 与 Java/Vue 项目的对比

| 概念 | Kazumi (GitHub Actions) | Spring Boot | Vue |
|------|------------------------|-------------|-----|
| CI 工具 | GitHub Actions | Jenkins / GitHub Actions | GitHub Actions |
| 测试 | flutter test | JUnit + Maven | Jest + npm test |
| 静态分析 | flutter analyze | SpotBugs / Checkstyle | ESLint |
| 构建 | flutter build | Maven / Gradle | npm build |
| 签名 | SignPath + android-release-signer | jarsigner | 无 |
| 发布 | softprops/action-gh-release | Maven Central / Docker Hub | npm publish / Vercel |
| 多平台 | 5 平台矩阵 | 通常单平台（JAR） | 通常单平台（静态文件） |
| 产物格式 | APK/ZIP/MSIX/DMG/IPA/tar.gz/deb | JAR/WAR | HTML/JS/CSS |

---

## 全部完成

14 章文档全部写完，覆盖了 Kazumi 项目的方方面面：

```
入门（01-04）：项目结构、启动流程、路由系统、PopularPage 链路
进阶（05-06）：规则引擎(XPath解析)、视频提取(WebView拦截)
高级（07-10）：硬件解码、弹幕系统、收藏同步(Event Sourcing)、下载管理(M3U8引擎)
拓展（11-14）：UI组件、超分辨率(Anime4K)、平台原生桥接(MethodChannel)、CI/CD(GitHub Actions)
```
