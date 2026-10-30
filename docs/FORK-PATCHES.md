# Fork 补丁：Linux x64 桌面构建

本仓库是 [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness) 的 fork，只维护一个补丁。

## 为什么需要它

上游 Desktop 的打包链路只声明了两个发布目标：`mac-arm64`、`mac-x64`、`win-x64`，并且都在自托管发布机上完成签名/公证。Linux 在运行时层是支持的（`scripts/primary-runtime/lock.json` 的 `targets` 已包含 `linux-x64`、`linux-arm64`，`electron-builder.config.mjs` 里也留了 `linux: { target: ['AppImage'] }` 分支），但**目标白名单没有接上**，所以任何 Linux 产物都打不出来。

本补丁把这层接上，并提供一条 GitHub 托管 runner 上的未签名 AppImage 构建。

## 补丁内容

| 层 | 改动 |
| --- | --- |
| 目标白名单 | `desktop-build-paths.mjs` / `.d.mts`：`SUPPORTED_TARGETS`、`desktopTargetPlatform()` 增加 `linux-x64` |
| 目标定义 | `package-target.ts`：`DesktopPackageTargetName`、`TARGETS`、`--linux` 构造器参数、Linux 构建主机校验 |
| 构建环境 | `desktop-package-environment.mjs` / `.d.mts`：新增 `localLinuxPackageEnvironment()`，未签名构建不读 dotenv、不校验更新策略 |
| 运行时准备 | `prepare-runtime.ts` / `prepare-dsh.ts`：Electron 可执行文件路径按平台解析（Linux 取解包根目录的 `electron`），并补 `chmod 0755` |
| CLI 启动器 | `prepare-cli.ts`：非 Windows 平台都补可执行位 |
| 打包后校验 | `smoke-packaged-runtime.ts`：`linux-unpacked/` 布局与应用名 |
| 发布策略 | `electron-builder-config.mjs`：未签名且非 Windows 时不再解析更新策略来源（Linux 没有签名链路） |
| 类型声明 | `desktop-auto-update-environment.d.mts`、`desktop-release-store.ts`、`desktop-upload-plan.ts`：目标枚举纳入 `linux-x64` |
| 脚本入口 | 根 `package.json` 与 `apps/desktop/package.json` 增加 `package:desktop:linux:x64` / `package:linux:x64` |

新增（fork 自有）文件：

- `.github/workflows/linux-package.yml` —— `workflow_dispatch`，ubuntu-latest，未签名 AppImage
- `.github/fork-patches.json`、`.github/fork-paths.txt` —— 补丁清单
- `docs/FORK-PATCHES.md` —— 本文件

## 怎么构建

本地（Linux x64 主机）：

```bash
pnpm install --frozen-lockfile
cd apps/desktop
pnpm run package:linux:x64 --check --unsigned   # 预检：配置 + 工具链
pnpm run package:linux:x64 --unsigned           # 产物在 .desktop-build/targets/linux-x64/unsigned-artifacts/
```

CI：Actions → **Package Desktop · Linux · x64** → Run workflow。产物 `linux-x64-unsigned-<run>-<attempt>` 里是 `.AppImage`，同时上传打包诊断日志。

## 边界与已知限制

- **只有未签名构建**。产物仅供本机使用，不进入上游的发布/自动更新链路（`DSH_DESKTOP_UNSIGNED=1` 会跳过签名、公证和更新策略校验）。
- 只覆盖 `linux-x64`。`linux-arm64` 的运行时构建脚本同样是现成的，但 `electron-builder` 目标与 CI runner 尚未接入。
- 上游 `pnpm run build` 的 `host aggregate check` 在 8 GB 机器上会被 OOM kill；GitHub 托管 runner 是 16 GB，实测可过。本地构建请确保内存或加 swap。

## 退役条件

上游若在 `apps/desktop/scripts/package-target.ts` 的 `TARGETS` 中自己声明了 `linux-x64`，且提供 Linux 打包工作流，本补丁即可退役：先 `git branch -f fork-archive/linux-x64-desktop-<日期> HEAD` 归档，再 reset 到上游。

当前未安装每日同步 workflow 与探测器；补丁刚落地，先以手工 rebase 维护。
