# Xwork 本地工作台

**本仓库是 Xwork 项目的开源仓库**：源码、协议文档、构建脚本与发布物料都归属这里。

Xwork 是一个单机运行的 Windows 桌面工作台 —— 一个进程里同时装着文件管理、文档预览、
应用启动器、本地知识库、AI 对话和插件扩展，数据只落本机，不依赖云端账号。

```
绿色版 · onedir · 解压即用 · 无安装器 · 无云端依赖
```

---

## 1. 能做什么

| 模块 | 能力 |
|---|---|
| 文件管理 | 目录浏览、挂载根白名单、新建/改名/移动/复制、批量改名、按类型或日期归档、回收站与撤销 |
| 预览 | 图片、音视频、CSV、PDF、Word/Excel/PPT（版式保真）、压缩包（zip/7z/rar，内置 7-Zip 免安装） |
| 应用启动器 | 七种启动类型、参数模板、图标提取、使用计数、开始菜单扫描与导入、批量删除可撤销 |
| 知识库 | 本地文件建索引、分块与向量检索、问答、笔记、OCR、索引增量更新与坏文件隔离 |
| AI 对话 | 多轮对话、知识库引用、密钥经 DPAPI 加密存储、模型可换 |
| 运维 | 网段计算、连通性巡检、日志与抓包入口等日常动作（按需扩展） |
| 插件系统 | 插件契约 v2：命令、文件动作、卡片、页面、面板、菜单、导航、设置、hooks、拦截器 |
| 界面与体验 | 原生 ES Module 前端、命令面板、全局热键、系统托盘、深/浅色主题 |

一切能力都注册成 `Command`，界面、命令面板、插件走同一套入口 —— 加功能不等于加按钮。

---

## 2. 技术栈

| 层 | 选型 |
|---|---|
| 后端 | Python 3.13 · FastAPI · uvicorn |
| 前端 | 原生 ES Module + 手写 CSS（无框架、无构建步骤） |
| 外壳 | pywebview（WebView2） |
| 存储 | SQLite（WAL） |
| 托盘/热键 | pystray · Win32 |
| 打包 | PyInstaller（onedir） |

分层约束：`core/` 不 import Web 框架，能力通过命令注册表暴露；路径常量只从
`core/paths.py` 取，区分可写目录与只读资源目录（打包后资源在 `_internal/`）。

---

## 3. 仓库状态

| 内容 | 状态 |
|---|---|
| 发布与更新协议（`docs/`、`schema/`、`update/`） | ✅ 已就位 |
| 源码（`core/` `api/` `web/` `plugins/` `tools/`） | ⏳ 随首个正式版迁入 |
| `LICENSE` | ⏳ 定稿前补 |

源码迁入前，构建与发布仍在本机工作副本中完成；本文档描述的是迁入后的目标布局。

目标布局：

```
xwork/
├─ run.py                入口（开发 / 打包同一套启动路径）
├─ core/                 纯逻辑层（不含 Web 框架）
├─ api/                  FastAPI 路由
├─ web/                  前端（原生 ESM + CSS）
├─ plugins/<id>/         插件（manifest.json + backend.py）
├─ tools/                构建、验收、发布脚本
├─ docs/                 文档（含 release-protocol.md、incremental-update.md）
├─ schema/               清单 JSON Schema
└─ update/               版本清单与更新说明
```

---

## 4. 运行

**发布包**：从 Releases 下载 `xwork-<version>-win64.zip`，解压到任意目录，双击 `Xwork.exe`。
首次启动会在同级生成 `config/`、`data/`、`plugins/`，三者跨版本保留，升级不覆盖。

**从源码**（迁入后）：

```bash
python -m venv .venv
.venv/Scripts/pip install -r requirements.txt
.venv/Scripts/python run.py --dev            # 开发：热加载 + 打开窗口
.venv/Scripts/python run.py --port 8765      # 只起服务
```

**要求**：Windows 10 1809+ / Windows 11（x64），需 WebView2 运行时（Win11 自带）。
Word 版式保真额外需要本机安装 Microsoft Word。

---

## 5. 构建与发布

```bash
python tools/build_exe.py --zip --with-model     # → dist/xwork-win64.zip
tools/verify_all.py                              # 全量验收
```

发布流程、版本清单字段、客户端检查与安装逻辑，见
[docs/release-protocol.md](docs/release-protocol.md)；更新体积如何进一步压缩，见
[docs/incremental-update.md](docs/incremental-update.md)。

> 更新协议是发布环节的工程约定，不是本仓库的用途定义。

---

## 6. 文档

| 文档 | 内容 |
|---|---|
| [docs/release-protocol.md](docs/release-protocol.md) | 发布与更新协议：清单字段、拉取降级链、安装与回滚 |
| [docs/incremental-update.md](docs/incremental-update.md) | 增量更新可行性、收益测算与前置条件 |
| [schema/update-manifest.schema.json](schema/update-manifest.schema.json) | 清单 JSON Schema |

---

## 7. 参与

- **Issue**：功能建议、缺陷、使用问题。
- **PR**：先开 Issue 对齐方向，再提改动；一个 PR 只做一件事。
- **约定**：`core/` 保持无 Web 框架依赖；新增能力注册为 `Command`；改动插件契约需同时更新协议文档。
- 提交信息用中文或英文均可，写清"为什么改"，不要只写"优化"。

---

## 8. 许可

许可证尚未选定，定稿前不视为已授权。选定后会在本文件与 `LICENSE` 中同时声明。
