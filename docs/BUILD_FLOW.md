# 构建流程详解

## 一、整体架构

```
你本地                              GitHub                              用户
  │                                   │                                   │
  │ ① git tag v1.0.0                 │                                   │
  │ ② git push origin v1.0.0        │                                   │
  │ ────────────────────────────────► │                                   │
  │                                   │ ③ Actions 检测到 v* tag           │
  │                                   │    自动启动 3 台虚拟机             │
  │                                   │                                   │
  │                                   │ ④ 三台虚拟机并行构建               │
  │                                   │    ubuntu-latest   → .deb/.AppImg │
  │                                   │    macos-latest     → .dmg        │
  │                                   │    windows-latest   → .msi        │
  │                                   │                                   │
  │                                   │ ⑤ release job 汇总产物            │
  │                                   │    创建 GitHub Release 并上传     │
  │                                   │                                   │
  │                                   │ ⑥ Release 页面自动出现下载链接    │
  │                                   │ ────────────────────────────────► │
  │                                   │                                   │ ⑦ 下载安装
```

核心依赖：

| 组件 | 作用 |
|------|------|
| [Pake](https://github.com/tw93/Pake) | 把网页打包成桌面应用的 CLI 工具（基于 Rust + Tauri） |
| GitHub Actions | 提供 Linux / macOS / Windows 三平台虚拟机，免费运行构建任务 |
| GitHub Releases | 永久存储构建产物（.dmg / .msi / .deb / .AppImage） |

---

## 二、触发机制

workflow 文件：`.github/workflows/build.yml`

```yaml
on:
  workflow_dispatch:        # 手动触发 — 在 Actions 页面点按钮
    inputs:                 # 手动触发时可配置的参数
      url: ...              #   要打包的网址
      app_name: ...         #   应用名称
      width: ...            #   窗口宽度
      height: ...           #   窗口高度
      platforms: ...        #   选择构建平台（all / linux / macos / windows）
      hide_title_bar: ...   #   macOS 隐藏标题栏
      multi_arch: ...       #   macOS 通用二进制
      debug: ...            #   开启 DevTools
  push:
    tags: ["v*"]            # 自动触发 — 推送 v 开头的 tag 时
```

### 两条触发路径

| 路径 | 触发方式 | 构建范围 | 产物去向 |
|------|---------|---------|---------|
| 自动 | `git push origin v1.0.0` | 三个平台全量构建 | GitHub Release（永久保存） |
| 手动 | Actions 页面点 Run workflow | 可选单个平台 | Actions Artifacts（90 天有效） |

### 为什么手动触发不走 Release

```yaml
release:
  if: startsWith(github.ref, 'refs/tags/')
```

手动触发时 `github.ref` 是 `refs/heads/main`（不匹配 `refs/tags/`），release job 被跳过，产物只保存在 Artifacts 中。

---

## 三、Job 执行流程（4 个 Job 详解）

```
push tag v1.0.0
      │
      ├── build-linux   (ubuntu-latest)  ──┐
      ├── build-macos   (macos-latest)  ──┤  并行执行
      ├── build-windows (windows-latest) ──┘
      │
      └── release       (ubuntu-latest)    ← 等上面三个全部完成
```

每个 build job 条件执行逻辑：

```yaml
if: |
  github.event_name == 'push' ||                      # tag 推送 → 必定执行
  github.event.inputs.platforms == 'all' ||            # 手动选了 all
  github.event.inputs.platforms == '<当前平台>'         # 手动选了本平台
```

---

### 3.1 build-linux — Linux 构建

**运行环境**：`ubuntu-latest`（GitHub 托管的最新 Ubuntu 虚拟机）

**超时限制**：25 分钟

#### 步骤 1：检出 Pake 源码

```yaml
- uses: actions/checkout@v4
  with:
    repository: tw93/Pake
```

不检出 ProNovel 仓库自身，而是检出 `tw93/Pake` 的源码。因为 Pake CLI 就在那个仓库里，需要编译它来打包应用。

#### 步骤 2：安装 Rust 工具链

```yaml
- uses: dtolnay/rust-toolchain@stable
  with:
    toolchain: stable
```

Pake 底层是 Tauri（Rust 框架），需要 Rust 编译器来构建二进制文件。

#### 步骤 3：初始化 Pake 构建环境

```yaml
- uses: ./.github/actions/setup-env
  with:
    mode: build
```

调用 Pake 仓库自带的 setup 脚本（`tw93/Pake/.github/actions/setup-env`），它会：
- 安装 pnpm 包管理器
- 安装 Node.js 依赖（`pnpm install`）
- 安装 Linux 系统依赖（webkit2gtk、libappindicator 等 Tauri 需要的库）

#### 步骤 4：编译 Pake CLI

```yaml
- run: pnpm run cli:build
```

把 TypeScript 写的 CLI 入口编译成 `dist/cli.js`。后续通过 `node dist/cli.js` 调用。

#### 步骤 5：恢复 Rust 缓存

```yaml
- uses: actions/cache/restore@v4
  id: cache
  with:
    path: |
      ~/.cargo/bin/          # Rust 编译好的二进制
      ~/.cargo/registry/index/
      ~/.cargo/registry/cache/  # 下载的 crate 源码
      ~/.cargo/git/db/         # git 依赖
      src-tauri/target/        # Tauri 编译产物
    key: Linux-cargo-pronovel-<Cargo.lock 的 SHA256>
```

- **首次构建**：缓存未命中（cache miss），跳过此步骤，全程编译大约 10-15 分钟
- **后续构建**：缓存命中（cache hit），跳过下载和编译已缓存的 crate，大约 5 分钟

缓存 key 中的 `<Cargo.lock 的 SHA256>` 保证：Pake 依赖更新 → Cargo.lock 变化 → 缓存自动失效 → 重新全量编译。

#### 步骤 6：调用 Pake CLI 打包

```bash
node dist/cli.js "https://pronovel.cc" \
  --name "pronovel" \
  --width "1200" \
  --height "780" \
  --targets deb,appimage
```

Pake CLI 在这一步做的事：

1. 用 Tauri 脚手架创建一个原生窗口应用
2. 配置 webview 加载 `https://pronovel.cc`
3. 设置窗口大小为 1200×780
4. 根据 `--targets` 参数调用 Tauri bundler 打包：
   - `.deb`：Debian/Ubuntu 安装包
   - `.AppImage`：便携式 Linux 应用

参数来自顶层 `env`：

```yaml
env:
  APP_URL: ${{ github.event.inputs.url || 'https://pronovel.cc' }}
  APP_NAME: ${{ github.event.inputs.app_name || 'ProNovel' }}
  APP_WIDTH: ${{ github.event.inputs.width || '1200' }}
  APP_HEIGHT: ${{ github.event.inputs.height || '780' }}
```

- 手动触发时：用你在 Actions 页面填的值
- tag 推送时：`github.event.inputs.*` 全部为空，走 `||` 后面的默认值

`${APP_NAME,,}` 是 bash 语法，把应用名转成全小写（Linux 约定应用名为小写）。

#### 步骤 7：整理产物

```bash
mkdir -p output
for f in *.deb;      do [ -f "$f" ] && mv "$f" output/ProNovel_linux_amd64.deb;      done
for f in *.AppImage; do [ -f "$f" ] && mv "$f" output/ProNovel_linux_amd64.AppImage; done
```

Pake CLI 生成的文件名是动态的（包含版本号等），这里重命名为固定格式方便下载页识别。

#### 步骤 8：上传到 Artifacts

```yaml
- uses: actions/upload-artifact@v4
  with:
    name: pronovel-linux
    path: output/*
```

产物名 `pronovel-linux`。如果本次是 tag 推送，后续 release job 会通过这个名字下载并汇总。

#### 步骤 9：保存 Rust 缓存

```yaml
- uses: actions/cache/save@v4
  if: steps.cache.outputs.cache-hit != 'true'
  with:
    path: ...
    key: Linux-cargo-pronovel-<Cargo.lock 的 SHA256>
```

只有首次构建（缓存未命中）才执行。把这次编译好的东西存起来，下次构建直接恢复。

---

### 3.2 build-macos — macOS 构建

**运行环境**：`macos-latest`

**超时限制**：30 分钟（macOS 编译 Tauri 比 Linux 慢）

步骤 1-5、8-9 与 Linux 相同。区别在步骤 6 和 7。

#### 步骤 6：调用 Pake CLI 打包（macOS 版）

```bash
FLAGS="--width ${APP_WIDTH} --height ${APP_HEIGHT}"

# 根据手动触发时的选项拼接额外参数
if [ "true" = "true" ]; then FLAGS="${FLAGS} --hide-title-bar"; fi
if [ "true" = "true" ]; then FLAGS="${FLAGS} --multi-arch"; fi
if [ "true" = "true" ]; then FLAGS="${FLAGS} --debug"; fi

node dist/cli.js "${APP_URL}" \
  --name "${APP_NAME}" \
  ${FLAGS} \
  --targets universal
```

macOS 专有参数：

| 参数 | 效果 |
|------|------|
| `--targets universal` | 生成 Universal Binary（同时包含 x86_64 和 ARM64 代码） |
| `--hide-title-bar` | 隐藏 macOS 原生标题栏，webview 延伸到窗口顶部（沉浸式） |
| `--multi-arch` | 确保 Intel Mac 和 Apple Silicon Mac 都能运行 |

注意 `APP_NAME` 保持原始大小写（`ProNovel`），因为 macOS 文件名区分大小写。

#### 步骤 7：整理产物

```bash
mkdir -p output
for f in *.dmg; do [ -f "$f" ] && mv "$f" output/ProNovel_macos.dmg; done
```

---

### 3.3 build-windows — Windows 构建

**运行环境**：`windows-latest`

**超时限制**：25 分钟

步骤 1-5、8-9 与 Linux 相同。区别在步骤 6 和 7，shell 改用 PowerShell。

#### 步骤 6：调用 Pake CLI 打包（Windows 版）

```powershell
node dist/cli.js "$env:APP_URL" `
  --name "$env:APP_NAME" `
  --width "$env:APP_WIDTH" `
  --height "$env:APP_HEIGHT" `
  --targets x64
```

Windows 无 `--hide-title-bar` 和 `--multi-arch`（这些仅在 macOS 生效）。变量引用语法从 bash 的 `${VAR}` 变为 PowerShell 的 `$env:VAR`。

#### 步骤 7：整理产物

```powershell
New-Item -Path output -ItemType Directory -Force
$msi = Get-ChildItem -Name "*.msi" | Select-Object -First 1
if ($msi) { Move-Item $msi output\ProNovel_windows_x64.msi }
```

Windows 路径用反斜杠 `\`，其余逻辑与 Linux/macOS 相同。

---

### 3.4 release — 创建 GitHub Release

**触发条件**：

```yaml
if: startsWith(github.ref, 'refs/tags/')
needs: [build-linux, build-macos, build-windows]
```

只有推送 tag 时执行，且必须等三个 build job 全部完成。

#### 步骤 1：下载所有产物

```yaml
- uses: actions/download-artifact@v4
  with:
    pattern: pronovel-*
    merge-multiple: true
    path: output
```

- `pattern: pronovel-*`：匹配 `pronovel-linux`、`pronovel-macos`、`pronovel-windows`
- `merge-multiple: true`：合并到同一个 `output/` 目录
- 下载后 `output/` 里的文件：
  ```
  output/
  ├── ProNovel_linux_amd64.deb
  ├── ProNovel_linux_amd64.AppImage
  ├── ProNovel_macos.dmg
  └── ProNovel_windows_x64.msi
  ```

#### 步骤 2：创建 Release 并上传

```yaml
- uses: ncipollo/release-action@v1
  with:
    allowUpdates: true    # 允许更新已有的 Release（三个平台共享同一个 Release）
    omitBody: true        # 不写 Release 正文
    omitName: true        # Release 名 = tag 名
    artifacts: output/*   # 把 output/ 下所有文件上传为附件
    token: ${{ secrets.GITHUB_TOKEN }}
```

`ncipollo/release-action` 做的事：
1. 根据当前 tag（如 `v1.0.0`）在 GitHub 上创建 Release
2. 把 `output/*` 所有文件作为附件挂到 Release 上
3. `allowUpdates: true` 确保三个平台的产物能关联到同一个 Release（虽然 release job 只运行一次，但这是遗留保护）

---

## 四、下载页自动更新机制

`index.html` 是纯静态页面，部署在 GitHub Pages。

### 核心逻辑

```javascript
(async () => {
  // 1. 推断 GitHub API 地址
  const API = (() => {
    const host = window.location.hostname;   // canwhite.github.io
    const owner = host.split('.')[0];        // canwhite
    const repo  = window.location.pathname
                    .split('/')
                    .filter(Boolean)[0]
                    || 'ProNovel';           // ProNovel
    return `https://api.github.com/repos/${owner}/${repo}/releases/latest`;
  })();
  // → https://api.github.com/repos/canwhite/ProNovel/releases/latest

  // 2. 请求最新 Release 信息
  const release = await fetch(API).then(r => r.json());

  // 3. 显示版本号
  document.getElementById('version').textContent = release.tag_name;
  // → "v1.0.0"

  // 4. 遍历 Release 附件，匹配文件扩展名
  const extMap = {
    dmg:       { card: 'card-macos',          btn: 'dl-macos' },
    msi:       { card: 'card-windows',        btn: 'dl-windows' },
    deb:       { card: 'card-linux-deb',      btn: 'dl-linux-deb' },
    AppImage:  { card: 'card-linux-appimage', btn: 'dl-linux-appimage' },
  };

  for (const asset of release.assets) {
    const ext = asset.name.split('.').pop();               // "dmg"
    const ext2 = asset.name.split('.').slice(-2).join('.'); // "AppImage"
    const key = ext2 === 'AppImage' ? 'AppImage' : ext;

    const el = extMap[key];
    if (!el) continue;  // 不认识的格式跳过

    // 5. 显示对应的下载卡片 + 设置下载链接 + 显示文件大小
    card.style.display = '';                    // 从隐藏变成可见
    btn.href = asset.browser_download_url;      // 下载链接
    const mb = (asset.size / 1024 / 1024).toFixed(1);
    btn.textContent = `Download (${mb} MB)`;    // 文件大小
  }
})();
```

### 为什么下载页"自动更新"

- 页面每次打开都调 GitHub API 拿**最新** Release
- 你推送 `v1.1.0` → API 返回 `v1.1.0` 的 Release → 页面自动显示新版本
- **不需要改任何 HTML、不需要手动更新链接**

### 文件匹配逻辑

```
Release 附件名称                    → 匹配到哪个卡片
ProNovel_macos.dmg                  → macOS (.dmg)
ProNovel_windows_x64.msi            → Windows (.msi)
ProNovel_linux_amd64.deb            → Linux (.deb)
ProNovel_linux_amd64.AppImage       → Linux (.AppImage)
```

`.AppImage` 特殊处理：`split('.').slice(-2).join('.')` 拿到 `AppImage`，而不是取最后一个 `.` 后面的 `AppImage` 中的某个部分。

---

## 五、你只需做一件事

```bash
# 发布新版本 → 全自动完成
git tag v1.0.0
git push origin v1.0.0
```

之后发生的事（无需你做任何操作）：

| 时间 | 发生了什么 |
|------|-----------|
| 推送 tag 后 0 秒 | GitHub Actions 检测到 `v1.0.0` tag，启动 workflow |
| 0-15 分钟（首次）/ 0-5 分钟（后续） | Linux、macOS、Windows 三台虚拟机并行构建 |
| 构建完成后 | release job 汇总产物，创建 GitHub Release，上传安装包 |
| Release 发布后 | 下载页自动显示 v1.0.0 的下载链接 |
| 永久 | 安装包挂在 Release 上，不会过期 |

**不需要：**
- 本地安装 Rust / Node.js / pnpm
- 手动编译任何东西
- 手动上传文件到 Release
- 手动编辑下载页 HTML
- 维护 Pake 的 fork
