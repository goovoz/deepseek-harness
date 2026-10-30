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
| 可执行名 | `electron-builder-config.mjs`：`linux.executableName` 固定为 `deepseek-harness`——包名带 scope（`@deepseek-ai/dsh-desktop`），electron-builder 推导出的名字含 `@`，AppImage 组装会直接拒绝 |
| Office 断言 | `smoke-runtime.ts`：没有原生 LibreOfficeKit 引擎的平台跳过 DOCX/XLSX/PPTX→PDF 的冒烟断言（见下方限制） |
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
- **Linux 上 Office→PDF 不可用**。`@deepseek-ai/libreoffice-kit` 的 optionalDependencies 只有 `darwin-arm64`、`darwin-x64`、`win32-x64`、`win32-arm64` 和 `wasm`，上游没有发布 Linux 原生引擎；打包后的运行时会报 `Installed LibreOfficeKit package is incomplete: @deepseek-ai/libreoffice-kit-linux-x64-glibc`。这不是本补丁引入的问题，是上游缺件。若上游补上 Linux 引擎，`smoke-runtime.ts` 的门控会自动恢复断言。
- 只覆盖 `linux-x64`。`linux-arm64` 的运行时构建脚本同样是现成的，但 `electron-builder` 目标与 CI runner 尚未接入。
- 上游 `pnpm run build` 的 `host aggregate check` 在 8 GB 机器上会被 OOM kill；GitHub 托管 runner 是 16 GB，实测可过。本地构建请确保内存或加 swap。
- 打包日志里的 `[WARN] Unsupported platform` 是 pnpm 按 `os`/`cpu` 过滤平台专属包，属正常输出；只要 `linux-x64` 不在被跳过之列即可。

## 本 fork 的 Actions 配置

上游共有 33 个 workflow，其中 `ci-master.yml`、`sandbox.yml`、`e2e.yml` 等在 push 到 master 时会跑 macOS/Windows 矩阵，对只出 Linux 包的 fork 是纯浪费。当前仓库设置：

- **仅 `linux-package.yml` 处于 active**，其余 32 个通过 `PUT /repos/{owner}/{repo}/actions/workflows/{id}/disable` 置为 `disabled_manually`；
- 恢复某个 workflow 只需 `PUT .../enable`，不需要改代码；
- Actions 缓存已清空（`DELETE /repos/{owner}/{repo}/actions/caches/{id}` 逐个删除，批量接口在无 `key` 参数时会 422）。

本地 `git push` 从某些网络环境会被 TLS 中断（`send-msg: unexpected disconnect`），补丁提交可用 Git Data API（blob → tree → commit → 更新 ref）发布，避开 smart-HTTP 通道。

## 退役条件

上游若在 `apps/desktop/scripts/package-target.ts` 的 `TARGETS` 中自己声明了 `linux-x64`，且提供 Linux 打包工作流，本补丁即可退役：先 `git branch -f fork-archive/linux-x64-desktop-<日期> HEAD` 归档，再 reset 到上游。

当前未安装每日同步 workflow 与探测器；补丁刚落地，先以手工 rebase 维护。
