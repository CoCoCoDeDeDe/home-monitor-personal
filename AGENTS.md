# Codespace 约定

本目录是个人 IoT 工作区，包含两个独立 git 仓库：

- 根目录 → `CoCoCoDeDe/home-monitor-personal`：管理根目录文件和 `personal/`，忽略 `home-monitor*/`
- `home-monitor/` → `CoCoCoDeDe/home-monitor`：home-monitor 项目本体

## 文档归属规则

- **不分 ticket 的文档**：存 `personal/`（可在其下再组织文件夹）
- **分 ticket 的文档**：存 `personal/tickets/<ticket名>/`（目录内可再组织文件夹）
- **项目通用、长期、广度的文档**：才能存 `home-monitor`（如架构设计、硬件清单、部署手册）

个人上下文、临时记录、调研笔记一律不进 home-monitor。

## Ticket 命名规则

本项目规划为多人协作，ticket 命名与工作流约定以 `home-monitor/CONTRIBUTING.md` 为准。ticket 实体存于 `personal/tickets/`（本仓库）。

## 提交规则

- `personal/` 下的 ticket 记录随 ticket 进度**自动 commit + push**（本仓库），无需逐次确认；`personal/tmp-ai-chat/` 等临时目录不提交。
- `home-monitor/` 仓库的改动仍需逐次确认后才 commit。

## 环境备忘

- Windows 11 + Git Bash；ESP 固件开发在 Windows 侧用 VSCode + PlatformIO（Core 在 `C:\.platformio`，`PLATFORMIO_CORE_DIR=C:/.platformio`）
- 网络走本地代理 `http://127.0.0.1:17890`：git 已对 github.com 单独配代理；pio 等依赖 `HTTP_PROXY`/`HTTPS_PROXY` 环境变量（已 setx 持久化，新终端生效；当前会话需手动 export）
- git 凭据按仓库路径区分（`credential.useHttpPath=true`）：`CoCoCoDeDe/*` 用 CoCoCoDeDe 账号，其余用机器原账号
