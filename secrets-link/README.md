# secrets-link

> token 文件 → 三层加密封装 → `secrets.cmd` 只引用路径 → 运行时解密进环境变量。目的：**token 不再明文躺在磁盘上**。

零依赖 Node 面板（端口 `7782`），支持多工作区（DSH / Codex / Claude / 多台机器各一套 `secrets.cmd`）。

## 快速开始

```bash
# 1. 复制配置样例并填入你自己的 token 文件路径
cp links.json.example links.json

# 2. 起面板
node secrets-link-web.mjs          # 或 launch-secrets-link.cmd / .sh
# 打开 http://127.0.0.1:7782
```

面板上：

- 显示「变量 → 文件路径 → token 预览（脱敏）」；
- 勾选 token →「🔒 封装选中」→ 生成 `<file>.zip`（内含 age 密文）+ `<file>.backup.enc`（AES-256-GCM 备份）；
- 点「生成 secrets.cmd」→ 得到只含路径引用的 `secrets.cmd`（`for /f` 运行时调用 `decrypt-runtime.py` 解密）。

## 三层加密链

| 层 | 产物 | 算法 | 用途 |
|---|---|---|---|
| 1 | `<file>.backup.enc` | AES-256-GCM（PBKDF2-SHA256 600k，密码 = age 私钥） | 真加密备份，可恢复 |
| 2 | age 密文 | X25519 + ChaCha20-Poly1305 | 公钥加密，私钥解密 |
| 3 | `<file>.zip` | zip 密码 = 公钥 | **防误提交封装**（非加密，公钥公开），面板只认 `.zip` |

恢复：`python decrypt-token.py <file>.backup.enc <identity> <out>`
运行时：`python decrypt-runtime.py <file>.zip <age_bin> <identity>` → 明文到 stdout。

## 文件说明

| 文件 | 作用 |
|---|---|
| `secrets-link-web.mjs` | 面板（多工作区 / 一键封装 / 预览解密 / 生成 secrets.cmd） |
| `launch-secrets-link.*` | 跨平台启动器（cmd / mjs / sh） |
| `encrypt-token.py` | 三层封装（AES 备份 + age + zip），封装后删明文 |
| `decrypt-runtime.py` | 运行时解密（解 zip → age 解密 → stdout） |
| `decrypt-token.py` | 从 `.backup.enc` 恢复明文 |
| `links.json.example` | 配置样例；真实 `links.json` 含本机路径，**已 gitignore** |

## 已知坑（来自实战）

1. `for /f` 的命令路径**不能含空格**——`GITHUB token` 这类目录用 8.3 短路径（`GITHUB~1`）。
2. 沙箱内 Node 子进程会因命名管道被拒（`EPERM`）；面板要在普通桌面会话跑。
3. `decrypt-runtime.py` 的临时文件写系统 temp，不写只读的密钥盘。
4. `cmd /c "call secrets.cmd && set X=%VAR%"` 里 `%VAR%` 会在调用前展开 → 拿空值；用 `.cmd` 文件或 `if defined X` 判断。
5. age 私钥 / identity 路径同样不能含空格。

详见仓库 `acp-studio/references/secrets-link-guide.md`。
