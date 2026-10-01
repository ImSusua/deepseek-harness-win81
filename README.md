# DeepSeek Harness · Windows 8.1 移植补丁

以纯源码补丁方式把 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)
桌面端（v0.2.0-rc.2, Electron 44）移植到 **Windows 8.1 x64**。

**本仓库不含上游源码**：CI 自动拉取上游 master → 应用 `dsh-win81-port.patch` → 构建，
产物为 `deepseek-harness-win81-x64-portable.zip`（免安装，8.1 解压即用）。

## 原理（为什么换壳）

官方桌面端是 Electron 44（Chromium 13x），其主程序静态导入 Win10-only 的
`api-ms-win-shcore-scaling-l1-1-1.dll` 等 API——Chromium 110+ 整体放弃 Win7/8.1，
无法通过单点补丁修复。**支持 Win8.1 的 Electron 上限 = 22.x（Chromium 109）**。

补丁把桌面壳钉到 **Electron 22.3.27**，并适配所有“Electron 44 时代假设”（共 10 处文件）：

| 改动 | 原因 |
|---|---|
| electron `^44.0.0` → `22.3.27` | 8.1 支持上限 |
| 新增 `win81-compat.ts`（undici 5 垫 fetch 全局） | Electron 22 主进程为 Node 16 |
| `net.fetch` → 全局 fetch | net.fetch 是 Electron 25 API |
| 内嵌 Node `24.21.0` → `22.19.0`（lock.json 含 sha256） | 22.19.0 静态导入表实证 8.1 全过 |
| ConPTY 可用性预检 | 8.1 无 ConPTY，终端面板优雅降级；**agent 工具执行不受影响** |
| `DSH_WIN81=1` 打包开关 + rg.exe 导入重定向（shim 转发 RtlGenRandom） | Rust 1.78+ 的 ProcessPrng 在 8.1 缺失 |

## 使用

1. 打开本仓库 Actions → **build-win81** → Run workflow。
2. 构建完成后在 Artifacts 下载 `dsh-win81-portable`。
3. 目标 8.1 机器：安装 [VC++ 2015-2022 x64 运行库](https://aka.ms/vs/17/release/vc_redist.x64.exe)，
   解压 zip，运行 `DeepSeek Harness.exe`。

## 已知限制（8.1 行为差异）

1. 交互式终端面板不可用（无 ConPTY）；命令执行/文件搜索等 agent 工具正常。
2. 自动更新禁用（unsigned 构建），升级需重新构建。
3. web UI 由 Chromium 109 渲染，110+ 前端特性可能有细节差异。
4. rg.exe 补丁后签名失效（不影响加载）。
5. 补丁跟踪上游 master；若上游大幅变动导致 `git apply` 失败，请在本仓库提 issue。

## 手动构建（Windows 构建机）

```bash
git clone https://github.com/deepseek-ai/deepseek-harness.git dsh && cd dsh
git apply ../dsh-win81-port.patch
pnpm install --no-frozen-lockfile && pnpm run build
DSH_WIN81=1 pnpm package:desktop:win:x64:dir
node apps/desktop/scripts/win81-patch-rg.mjs <dist>/win-unpacked
```

## 许可

上游 MIT，本补丁同以 MIT 发布。姊妹项目：[FlClash-Win81](https://github.com/ImSusua/FlClash-Win81)。
