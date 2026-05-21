# ProNovel

[ProNovel](https://pronovel.cc) 的桌面客户端，基于 [Pake](https://github.com/tw93/Pake) 构建。

## 下载

前往 [Releases](../../releases) 页面下载最新版本。

| 平台 | 格式 |
|------|------|
| macOS | `.dmg`（Universal，支持 Intel + Apple Silicon） |
| Windows | `.msi`（64-bit） |
| Linux | `.deb` / `.AppImage` |

## 自动构建

本项目通过 GitHub Actions 实现**全自动构建 + 发布**，无需本地环境。

### 发布新版本（自动）

```bash
# 打完 tag 推送即可，构建、打包、发布全部自动完成
git tag v1.0.0
git push origin v1.0.0
```

推送 tag 后发生的事情：

1. GitHub Actions 自动启动三台虚拟机（Linux / macOS / Windows）并行构建
2. 每个平台打完包，自动创建 GitHub Release 并上传安装包
3. 构建完成，Release 页面自动出现下载按钮

### 查看构建状态

推送 tag 后，打开仓库的 [Actions](../../actions) 页面，左侧点击 **Build & Release** 即可看到实时进度。

### 手动触发（测试用）

1. 进入 [Actions](../../actions) → 左侧点击 **Build & Release**
2. 右侧点击 **Run workflow** 下拉菜单
3. 选择平台（默认全部），点击绿色 **Run workflow**
4. 构建完成后在对应 run 的 Artifacts 中下载产物

> 手动触发的产物存在 Artifacts 中（90天有效）。推送 tag 的产物挂在 Release 下（永久保存）。

## Supported Platforms

- macOS (Universal: Intel + Apple Silicon)
- Windows (64-bit)
- Linux (.deb + AppImage)

## How It's Built

- 检出 [tw93/Pake](https://github.com/tw93/Pake) 源码，编译 CLI，调用 Pake 打包
- 推送 `v*` tag → 自动三平台构建 → 自动 GitHub Release
- Rust 依赖缓存加速，首次约 15 分钟，后续约 5 分钟
