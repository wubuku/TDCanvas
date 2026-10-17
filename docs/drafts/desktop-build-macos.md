# macOS 桌面客户端构建与安装记录

本文记录在 macOS 本机完成 TDCanvas 桌面客户端构建、打包和安装的过程，便于在另一台 macOS 上复现。

## 1. Remote 配置

执行前仓库只有一个 `origin`，地址为 `https://github.com/AICoderTudou/TDCanvas.git`。执行：

```bash
git remote rename origin upstream
git remote add origin https://github.com/wubuku/TDCanvas.git
git config branch.main.remote origin
git config branch.main.merge refs/heads/main
git fetch origin main
```

最终配置：

```text
origin   https://github.com/wubuku/TDCanvas.git
upstream https://github.com/AICoderTudou/TDCanvas.git
```

当前 `main` 分支默认跟踪 `origin/main`。

## 2. 本机构建环境

- macOS 26.6.2（Build 25G83）
- Apple Silicon（`arm64`）
- Node.js v24.6.0
- npm 11.5.1
- Rust 1.98.0-nightly，Cargo 1.98.0-nightly
- 项目 README 要求 Node.js 20+、Rust 1.85+；本机版本满足要求

## 3. 安装前端依赖

```bash
cd web
npm install --legacy-peer-deps
```

本次安装完成后新增 1368 个依赖包。`node_modules` 未纳入 Git，`package-lock.json` 没有产生工作区改动。

## 4. 正式打包脚本与本地构建脚本

README 推荐的 macOS 打包入口是：

```bash
npm run desktop:package:macos
```

仓库脚本会为正式安装包生成更新签名产物，因此需要预先设置以下环境变量：

```text
TDCANVAS_UPDATER_ENDPOINT
TDCANVAS_UPDATER_PUBLIC_KEY
TAURI_SIGNING_PRIVATE_KEY
```

本机未配置这些签名参数；执行正式脚本时按设计在构建开始前停止，并提示使用本地构建命令。不要把私钥写入仓库或本文档。

本次使用仓库提供的无更新签名本地构建：

```bash
npm run desktop:build
```

该命令先执行 `npm run build`，再执行 `tauri build`。构建成功，Vite 只输出动态导入和大 chunk 的优化提示，没有错误。由于直接运行的是本地构建命令，产物为当前 Apple Silicon 架构（`aarch64`）；README 中所说的 Intel + Apple Silicon 通用包由带签名配置的 `desktop:package:macos` 脚本负责。

## 5. 构建产物

构建成功后产物位于：

```text
web/src-tauri/target/release/bundle/macos/TDCanvas.app
web/src-tauri/target/release/bundle/dmg/TDCanvas_0.14.0_aarch64.dmg
```

产物信息：

- 应用标识：`com.tdtv.tdcanvas`
- 版本：`0.14.0`
- `.app` 大小约 20 MB
- `.dmg` 大小约 6.8 MB
- `.dmg` SHA-256：`109378a1f0392553b056879707cda2d738dcfe1e01e7a18a193b850b6c03c8a6`

可用以下命令重新计算校验值：

```bash
shasum -a 256 web/src-tauri/target/release/bundle/dmg/TDCanvas_0.14.0_aarch64.dmg
```

## 6. 安装到本机应用目录

确认 `/Applications/TDCanvas.app` 原先不存在后，执行：

```bash
ditto \
  web/src-tauri/target/release/bundle/macos/TDCanvas.app \
  /Applications/TDCanvas.app
```

已安装位置：

```text
/Applications/TDCanvas.app
```

验证安装信息：

```bash
/usr/libexec/PlistBuddy -c 'Print :CFBundleIdentifier' \
  /Applications/TDCanvas.app/Contents/Info.plist
/usr/libexec/PlistBuddy -c 'Print :CFBundleShortVersionString' \
  /Applications/TDCanvas.app/Contents/Info.plist
```

预期分别为 `com.tdtv.tdcanvas` 和 `0.14.0`。

这是未配置发布签名和更新签名的本地构建，不是可直接分发给其他用户的公证安装包；在另一台机器上复现正式发布包时，需要准备上述更新签名环境变量，并使用 `npm run desktop:package:macos`。
