# secure-push-kit

> 一站式「安全凭据 + 安全发布」工具包：用 **secrets-link** 管 token，用 **打包 → 安检 → 推送** 一条链发布到 **GitHub · Gitee · AtomGit** 三平台。

---

## 项目内容（人话版）

这个仓库解决两个日常痛点：

1. **token 不想明文躺在磁盘上** —— `secrets-link` 是一个本地小面板：token 只存加密文件（AES-256-GCM 备份 + age 公钥加密 + `.zip` 防误提交封装），`secrets.cmd` 只引用路径，运行时才解密进环境变量。
2. **发布怕带出密钥、怕漏推平台** —— `打包（github-ready-packager）→ 安检（secret-scan）→ 推送（acp-studio / push-secure.cmd）` 固化成一条链，安检不通过就硬性停住；推送支持 GitHub、Gitee、AtomGit 三平台一次推完。

它不是某个框架的插件，而是四个独立 skill + 两个本地面板/脚本的**整合分发版**，任何 Agent（DSH / Codex / Claude Code / Cursor）或手工命令行都能用。

## 环境依赖

| 依赖 | 版本 | 必需性 | 用途 |
|---|---|---|---|
| 操作系统 | Windows / macOS / Linux | 必需 | 脚本均跨平台；`start.cmd` / `.cmd` 启动器为 Windows |
| Python | 3.9+（仅标准库） | 必需 | `secret-scan`、`github-ready-packager` 打包脚本、`acp-studio` 面板 |
| Node.js | 18+（仅标准库） | 必需 | `secrets-link` 面板 |
| git | 2.30+ | 必需 | 提交 / 推送 / 同步 |
| age | v1.2+ | 可选 | 三层加密中的 age 公钥加密（`age.exe` 亦可自行编译） |
| gh CLI | 任意近期版本 | 可选 | GitHub 推送的 token 兜底（`gh auth token`） |

## 目录结构

```
secure-push-kit/
├── README.md
├── LICENSE                     MIT
├── .gitignore                  排除本地凭据 / 生成物
├── secrets-link/               token 加密链 + 多工作区面板（Node，端口 7782）
│   ├── secrets-link-web.mjs
│   ├── launch-secrets-link.{cmd,mjs,sh}
│   ├── encrypt-token.py        AES 备份 + age + zip 三层封装
│   ├── decrypt-runtime.py      运行时解密（zip → age → 明文 stdout）
│   ├── decrypt-token.py        从 .backup.enc 恢复
│   └── links.json.example
├── acp-studio/                 提交 / 推送 / 双平台同步面板（Python 标准库，端口 8790）
│   ├── SKILL.md
│   ├── server.py
│   ├── web/                    零依赖前端（index.html / app.js / styles.css）
│   ├── start.cmd / start.sh
│   ├── config.example.json
│   ├── secrets.cmd.example
│   ├── scripts/push-secure.cmd 三平台安全推送（github | gitee | atomgit）
│   ├── scripts/push_all.py     三平台推送（Python，无需 shell helper，沙箱环境可用）
│   ├── scripts/sync_github_gitee.py
│   └── references/secrets-link-guide.md
└── skills/                     四个可被 Agent 直接调用的 skill
    ├── github-ready-packager/  SKILL.md + scripts/package_for_github.py
    ├── secret-scan/            SKILL.md + scripts/scan_secrets.py + references/
    ├── secure-push-workflow/   SKILL.md（编排层：打包 → 安检 → 推送）
    └── （acp-studio 的 SKILL.md 在 acp-studio/ 目录内）
```

## 工作流：打包 → 安检 → 推送

| 阶段 | 工具 | 可判定信号 | 说明 |
|---|---|---|---|
| 0 包装备 | `github-ready-packager` | zip 产出 + 语法/烟测通过 | 补齐 README/.gitignore/依赖清单/启动脚本，排除密钥与大文件 |
| 1 打包 | `scripts/package_for_github.py` | zip 路径 | 按排除规则逐文件写入，不依赖 `Compress-Archive` |
| 2 安检 | `secret-scan` | `findings=0` | 只读扫描；命中真实凭据即停，脱敏报告，**明确「未推送」** |
| 3 推送 | `acp-studio` / `push-secure.cmd` | commit hash + 远端分支确认 | token 只经环境变量 / credential helper，不进 argv、不落盘 |
| 4 同步 | `scripts/sync_github_gitee.py` | fast-forward 结果 | GitHub ↔ Gitee 单/双向；分叉分支只报告不强推 |

**硬门**：`secret-scan` 发现真实凭据 → 立即停止，绝不继续推送（见 `skills/secure-push-workflow/SKILL.md`）。

## 快速开始

### 1) 起 secrets-link 面板（可选，管 token 用）

```bash
node secrets-link/secrets-link-web.mjs      # 打开 http://127.0.0.1:7782
```

- 首次使用：复制 `secrets-link/links.json.example` → `links.json`，填入你自己的 token 文件路径（`links.json` 已 gitignore）。
- 面板显示「变量 → 文件路径 → token 预览（脱敏）」；选中 token 可一键封装成 `.zip`（内含 age 密文）+ `.backup.enc`（AES-256-GCM）。
- 点「生成 secrets.cmd」，得到只含路径引用的批处理配置。

### 2) 提交并推送（三平台）

```bat
rem 单仓库、单平台（github|gitee|atomgit）
acp-studio\scripts\push-secure.cmd  G:\path\to\repo  repo-name  atomgit
acp-studio\scripts\push-secure.cmd  G:\path\to\repo  repo-name  github
acp-studio\scripts\push-secure.cmd  G:\path\to\repo  repo-name  gitee
```

或起可视化面板（支持 `target=all` 一次推三个平台，附 AtomGit 远端自动推导）：

```bash
cd acp-studio
python server.py            # http://127.0.0.1:8790
```

也可以直接用 Python 一次推三个平台（不依赖 `sh` 子进程，受限/沙箱环境更稳）：

```bash
python acp-studio/scripts/push_all.py <repo路径> repo-name \
    --targets github,gitee,atomgit --atomgit-user <你的AtomGit用户名> --proxy http://127.0.0.1:7897
```

> 为什么有两套：`push-secure.cmd` 走 git 的 `credential.helper=!f(){...}`（不把 token 放进 argv，但需要能起 POSIX shell）；`push_all.py` 走 `http.extraheader=Authorization: Basic ...`（不需要 shell，token 只经环境变量传入并在输出里脱敏）。按环境选一个即可。

面板 JSON API：

```http
GET  /api/repo?repo=<绝对路径>              # 分支 / 远端 / 变更 / 最近提交
POST /api/commit  {"repo_path":"...","message":"✨ feat: ...","push":true,"target":"all"}
POST /api/push    {"repo_path":"...","target":"all"}
POST /api/sync    {"repos":[],"dry_run":false,"direction":"github-to-gitee"}
```

### 3) 安检（推送前必跑）

```bash
python skills/secret-scan/scripts/scan_secrets.py --local <打包产物目录>
```

## 安全模型（诚实说明）

- **AES-256-GCM `.backup.enc`**：密码 = age 私钥派生（PBKDF2-SHA256 600k 次），用 `cryptography` 库实现，是真加密、可恢复。
- **age 密文**：X25519 + ChaCha20-Poly1305，公钥加密、私钥解密。
- **`.zip` 封装**：zip 密码 = 公钥 —— 这是「**防误提交封装**」而不是加密（公钥公开、无保密性），只用于给「已加密」做格式标记，并让面板只接受 `.zip` 输入。
- **token 不进 argv、不落盘、不进仓库配置**：`secrets.cmd` 只引用路径；git 侧用 credential helper / 运行时解密的环境变量。
- 任何 `links.json`、`secrets.cmd`、`config.json`、`state.json`、`*.backup.enc`、`*.zip`、`.env`、私钥都在 `.gitignore` 内，**不要手动提交**。

## 四个 skill 各自负责什么

| skill | 角色 | 入口 |
|---|---|---|
| `secure-push-workflow` | 编排层：顺序调用下面三个，安检不过硬停 | `skills/secure-push-workflow/SKILL.md` |
| `github-ready-packager` | 整理成可提交 GitHub 的独立项目 + zip | `skills/github-ready-packager/` |
| `secret-scan` | 只读凭据扫描（本地 + GitHub/Gitee 远端），脱敏报告 | `skills/secret-scan/` |
| `acp-studio` | 提交 / 推送 / GitHub↔Gitee 同步，三平台目标 | `acp-studio/SKILL.md` |

## 与上游的关系

本仓库是**整合分发版**，不是唯一来源：

- `secrets-link/` 来自 DSH 框架的 `INTRO/secrets-link`（运行中的 live 副本）。
- `skills/` 四个 skill 来自 `agent-deploy/skills/`。
- `acp-studio/` 面板来自原 `acp-studio` 仓库（该仓库已标注废弃、内容并入 `agent-deploy`）。

如果发现 bug，优先在对应上游修，再同步回本仓库；本仓库新增了 `push_all.py`（三平台、无 shell helper），并把 `push-secure.cmd` 扩展为支持 `atomgit` 目标与 `SECRETS_CMD` 覆盖。

## 免责声明

- 仅用于**你自己的凭据与自有仓库**；不要用它托管或分发他人 token。
- 面板只绑定 `127.0.0.1`；请勿把 7782 / 8790 暴露到公网。
- `secret-scan` 是启发式扫描，不保证 100% 覆盖；发布前请自行复核。

## License

[MIT](LICENSE) © 2026 hpsks416
