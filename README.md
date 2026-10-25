# xwork — 更新源仓库

Xwork 本地工作台的**更新发布仓库**。只做两件事：托管版本清单（本仓库文件）、托管安装包（Releases 附件）。

> 客户端只读一个入口：`update/stable.json`。其余文件都是给人看和维护用的。

---

## 1. 目录结构（本仓库要维护的文件）

```
xwork/
├─ README.md                          本文件：协议与发布流程
├─ schema/
│  └─ update-manifest.schema.json     清单字段定义（JSON Schema，客户端可校验）
├─ update/
│  ├─ stable.json                     稳定渠道清单 ← 客户端唯一入口
│  ├─ beta.json                       内测渠道清单（可选，不存在 = 无内测版）
│  ├─ stable.example.json             清单模板（首个正式发布前保留，勿删）
│  └─ notes/
│     ├─ v1.12.0.md                   该版本的更新说明（Markdown）
│     └─ v1.11.0.md
└─ （代码不放这里，代码在本地工作副本，本仓库只存发布物料）
```

| 文件 | 谁读 | 变更频率 |
|---|---|---|
| `update/stable.json` | **客户端每次检查更新** | 每个正式版本改一次 |
| `update/beta.json` | 选择内测渠道的客户端 | 每个内测版本 |
| `update/notes/v*.md` | 客户端（点"查看详情"时）与人 | 每版本新增一个 |
| `schema/update-manifest.schema.json` | 客户端校验、维护者 | 极少改（改要升 `schema` 号） |

**安装包本体不放这里**，放在 Releases 附件里，原因见第 6 节。

---

## 2. 客户端如何使用（三步）

```
① 取清单        GET update/stable.json（依次试三条路径，见 §4）
② 比版本        remote.version vs 本地 VERSION → 新 / 同 / 旧
③ 取包并安装    按 platforms.<plat> 的 url → 失败换 mirrors → 校验 sha256 → 解压 → 替换程序部件
```

### 检查时机（客户端侧约定）

- 启动后 **10 秒**异步检查一次，不阻塞界面，超时 8 秒，失败静默只写日志。
- 距上次自动检查不足 **24 小时**则跳过；手动"检查更新"不受限制。
- 全程只发 1 个 HTTP 请求（清单），不预取包。

### 版本比对规则

1. 去掉前缀 `v`，得到 `main` 或 `main-pre` 形式。
2. 拆 `major.minor.patch`，缺位补 0，**逐段按整数比较**（不能按字符串比，否则 `1.10.0 < 1.9.0`）。
3. 三段相同再看预发布后缀：都没有 → 相等；只有一个有 → **有后缀的更小**；都有 → 按标识符逐段比，
   数字段按数值、字母段按字典序，数字段小于字母段，标识符少的小。
4. 决策：

   | 结果 | 动作 |
   |---|---|
   | `remote > local` | 提示可更新（`mandatory: true` 时不可跳过） |
   | `remote == local` | 无操作 |
   | `remote < local` | 不提示，只记日志（本地是开发版或回滚版） |

5. **防回滚**：客户端本地记 `highest_seen`。远端版本低于它时，必须用户显式确认"降级安装"，
   否则拒绝——防止清单被篡改成旧版（旧版可能带已知漏洞）。

---

## 3. 清单字段

```json
{
  "schema": 1,
  "channel": "stable",
  "app": "xwork",
  "published_at": "2026-10-25T20:30:00+08:00",
  "latest": {
    "version": "1.12.0",
    "released_at": "2026-10-25",
    "min_version": "1.9.0",
    "mandatory": false,
    "notes_url": "https://raw.githubusercontent.com/gaio-x/xwork/main/update/notes/v1.12.0.md",
    "notes": "修复压缩包预览内存泄漏；连通性巡检支持按目标配速。",
    "platforms": {
      "win-x64": {
        "kind": "zip-onedir",
        "file": "xwork-1.12.0-win64.zip",
        "size": 232783872,
        "sha256": "0000000000000000000000000000000000000000000000000000000000000000",
        "url": "https://github.com/gaio-x/xwork/releases/download/v1.12.0/xwork-1.12.0-win64.zip",
        "mirrors": [
          "https://gh-proxy.com/https://github.com/gaio-x/xwork/releases/download/v1.12.0/xwork-1.12.0-win64.zip",
          "https://ghproxy.net/https://github.com/gaio-x/xwork/releases/download/v1.12.0/xwork-1.12.0-win64.zip"
        ]
      }
    }
  },
  "history": [
    { "version": "1.11.0", "released_at": "2026-10-25", "tag": "v1.11.0" }
  ]
}
```

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `schema` | int | ✅ | 清单格式版本，**不兼容变更才升**；客户端只认已知值，不认识就拒更新 |
| `channel` | string | ✅ | `stable` / `beta`，必须与文件所在渠道一致 |
| `app` | string | ✅ | 固定 `xwork`，防止清单张冠李戴 |
| `published_at` | string(ISO8601) | ✅ | 清单发布时间，含时区 |
| `latest.version` | string | ✅ | 新版本号，与 `core/version.py` 的 `VERSION` 完全一致 |
| `latest.released_at` | string | ✅ | 发布日期 `YYYY-MM-DD` |
| `latest.min_version` | string | ✅ | 低于它不保证平滑升级：走"整包覆盖 + 提示先备份" |
| `latest.mandatory` | bool | ✅ | `true` = 强制更新（仅安全修复用），UI 不可跳过 |
| `latest.notes_url` | string | ⬜ | 更新说明的 Markdown 地址 |
| `latest.notes` | string | ✅ | 一句话摘要（≤200 字），**离线可读**，不依赖 notes_url |
| `latest.platforms` | object | ✅ | 键为 `win-x64`（将来可加 `win-arm64`、`portable`） |
| `platforms.*.kind` | string | ✅ | 目前只有 `zip-onedir` |
| `platforms.*.file` | string | ✅ | 附件文件名，须与 Release 附件**完全一致** |
| `platforms.*.size` | int | ✅ | 精确字节数：算进度、校验续传点 |
| `platforms.*.sha256` | string(64 hex) | ✅ | 整包哈希，下载后必须校验 |
| `platforms.*.url` | string | ✅ | 规范地址（GitHub 原始 Release 地址） |
| `platforms.*.mirrors` | string[] | ⬜ | 有序兜底镜像，客户端**按顺序**试 |
| `history` | array | ⬜ | 最近若干版本，供"回滚到旧版"列出候选 |
| `signature` | object | ⬜ | `{alg, key_id, value}`，Ed25519 签名（默认关闭） |

**校验纪律（客户端侧）**：`schema` / `app` / `channel` 不匹配、`platforms[当前平台]` 缺失、
`sha256` 非 64 位十六进制、`size <= 0`、`version` 非法 —— 任一项不过**直接拒绝本次更新**，
不降级容错。清单是更新链的根，容错等于给投毒留口子。

---

## 4. 拉取路径与代理（实测，2026-10-25）

清单和包都走同一套降级链，顺序必须一致——否则会出现"清单从一个源拿、包从另一个源拿"，
一半通一半不通：

| 顺序 | 地址写法 | 实测 |
|---|---|---|
| 1 | 直连 `https://raw.githubusercontent.com/...` / `https://github.com/.../releases/download/...` | **经代理 1.05–1.18 MB/s**；不走代理则握手失败 |
| 2 | `https://gh-proxy.com/<原始地址>` | 200，45 KB/s |
| 3 | `https://ghproxy.net/<原始地址>` | 200，124 KB/s |

要点：

- **必须显式配代理**。只依赖环境变量不可靠：当前 `HTTP_PROXY` 指向的端口会把 `github.com`
  路由成直连从而握手失败，且端口会随客户端重启变化。客户端应支持"自动探测候选端口 → 手动指定 → 关闭"。
- 镜像不需要代理，是代理不可用时的兜底；速度慢一个数量级，222 MB 需 30–80 分钟。
- **分段并发无收益**：实测 4 段并发 128 KB/s，与单连接 124 KB/s 基本一致，瓶颈在端到端带宽。
  所以客户端做**断点续传**（`Range`，镜像实测支持 `206`），不做多线程分片。

---

## 5. Release 约定

| 项 | 约定 |
|---|---|
| tag | `v<version>`，如 `v1.12.0`（与清单 `version` 一一对应） |
| 标题 | `Xwork v1.12.0` |
| 附件 | `xwork-<version>-win64.zip`（onedir 整包，解压即用） |
| 附加 | `xwork-<version>-win64.zip.sha256`（纯文本，`<hash>  <文件名>`） |
| 说明 | Release 正文与 `update/notes/v<version>.md` 内容一致 |
| draft | 先 draft 上传完附件再发布，避免半成品被客户端拉到 |

---

## 6. 为什么安装包不放在仓库文件里

- GitHub **单文件硬上限 100 MB**，当前整包 222 MB，推不上去（`git push` 会被服务端直接拒绝）。
- Release 附件上限 **2 GB**，带 CDN、支持 `Range`，适合大包。
- 因此：**清单走仓库文件（可 diff、可回滚、体积极小），二进制走 Release 附件**。

体量构成（为什么包这么大）：解包约 382 MB，主要是 `cv2` 113 MB、`onnxruntime` 64 MB、
`pymupdf` 45 MB、`jieba` 37 MB，另加应用数据里的向量模型。
**后续若要做增量更新，拆包比做二进制 diff 更划算。**

---

## 7. 渠道

| 渠道 | 文件 | 客户端行为 |
|---|---|---|
| `stable` | `update/stable.json` | 默认。只放验证过的正式版 |
| `beta` | `update/beta.json` | 需手动开启。文件不存在即视为无内测版 |

两个渠道各自独立，不做跨渠道版本号比较（避免 `1.12.0-beta` 与 `1.12.0` 打架）。

---

## 8. 发布流程（维护者）

1. 升级 `core/version.py` 的 `VERSION`，跑通全量验收。
2. 打包：`python tools/build_exe.py --zip --with-model` → `dist/xwork-win64.zip`。
3. 计算 `size` 与 `sha256`。
4. 建 draft Release（tag `v<version>`），上传 zip 与 `.sha256`。
5. 写 `update/notes/v<version>.md`。
6. 更新 `update/stable.json`：`version` / `size` / `sha256` / `url` / `mirrors` / `notes` / `history`。
7. 提交清单 → Release 转正式。
8. **发布后自检**：`curl` 清单比对本地 `VERSION`；`curl -r 0-1023` 附件确认 `206`；
   抽样下载全包校验 `sha256` 与清单一致。

> 2–8 由 `tools/release_app.py` 一条命令完成（待实现）。人只负责写更新说明和按下确认。

---

## 9. 权限

| 动作 | 需要什么 |
|---|---|
| 检查更新 / 下载安装包 | **无需任何凭据**（公开仓库） |
| 发布清单 / 上传附件 | GitHub PAT，scope `repo` |

公开仓库是刻意的：一旦私有，客户端就得内嵌只读 token，反编译即泄漏，且"一键更新"变成"先填 token"。
