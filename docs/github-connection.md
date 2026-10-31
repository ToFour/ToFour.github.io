# GitHub 连接规范（Windows + Clash 代理环境）

> 适用：ToFour.github.io 及其他需推送 GitHub 的项目。本环境 **直连 github.com:443 不通**，必须走代理。

---

## 1. 前置依赖

| 项 | 值 |
|----|----|
| gh CLI 版本 | 2.103.3 |
| gh 安装路径 | `C:\Program Files\GitHub CLI\gh.exe` |
| 代理工具 | Clash（mixed-port 每次重启会变） |
| 当前代理端口 | **51133**（以 `~/.config/clash/config.yaml` 的 `mixed-port` 为准） |
| 直连 github.com:443 | ❌ 不通，必须走代理 |

---

## 2. 凭证来源（已确认可用）

- **gh CLI**：`gh auth status` 显示未登录，`hosts.yml` 不存在，环境变量无 token
- **实际凭证**：存储在 **Windows 凭据管理器** 中（git 默认 credential helper）
  - 用户名：`ToFour`
  - token：运行时通过 `git credential fill` 获取（见下方命令），**不要硬编码到仓库**
- **不要轻易重新授权**（`gh auth login`），优先复用已有 token

---

## 3. 代理设置

### 当前会话临时生效（推荐）

```powershell
$port = (Select-String "$env:USERPROFILE\.config\clash\config.yaml" -Pattern "mixed-port").Line.Split(':')[1].Trim()
$env:HTTPS_PROXY = "http://127.0.0.1:$port"
$env:HTTP_PROXY  = "http://127.0.0.1:$port"
```

### 持久化写入用户环境变量（可选）

```powershell
[Environment]::SetEnvironmentVariable("HTTPS_PROXY", "http://127.0.0.1:51133", "User")
```

> 注意：端口会变，持久化后需在 Clash 重启后同步更新。一般推荐临时设置。

---

## 4. 校验命令（三条）

```powershell
# ① 确认代理端口与连通性（200/301 即通）
$port = (Select-String "$env:USERPROFILE\.config\clash\config.yaml" -Pattern "mixed-port").Line.Split(':')[1].Trim()
Invoke-WebRequest -Uri "https://github.com" -Proxy "http://127.0.0.1:$port" -UseBasicParsing -TimeoutSec 10 -MaximumRedirection 0

# ② 确认 gh 账号身份（token 从凭据管理器取，不硬编码）
$env:GH_TOKEN = (echo "protocol=https`nhost=github.com" | git credential fill | Select-String "password=").ToString().Split('=')[1]
$env:HTTPS_PROXY = "http://127.0.0.1:$port"
& "C:\Program Files\GitHub CLI\gh.exe" api user --jq .login
# 期望输出: ToFour

# ③ 确认仓库可达
& "C:\Program Files\GitHub CLI\gh.exe" repo view ToFour/ToFour.github.io --json name,visibility
```

### git 推送命令（带代理 + HTTP/1.1）

```powershell
git -c http.version=HTTP/1.1 `
    -c http.proxy="http://127.0.0.1:$port" `
    -c https.proxy="http://127.0.0.1:$port" `
    push
```

> `http.version=HTTP/1.1` 是必须的，否则会报 `TLS connect error: unexpected eof while reading`。

---

## 5. 故障排查清单（按顺序判断）

| 步骤 | 检查项 | 命令 | 失败处理 |
|------|--------|------|---------|
| 1 | 代理端口是否监听 | `Get-NetTCPConnection -State Listen -LocalAddress 127.0.0.1 \| Where-Object LocalPort -eq $port` | 端口变了，重读 config.yaml |
| 2 | 代理能否访问 github | `Invoke-WebRequest ... -Proxy http://127.0.0.1:$port https://github.com` | Clash 未运行或节点失效 |
| 3 | gh 认证是否有效 | `gh api user --jq .login` | token 过期 → 用 git credential fill 取新 token |
| 4 | git ls-remote 是否成功 | `git -c http.version=HTTP/1.1 -c http.proxy=... ls-remote origin` | 检查 remote URL 与分支 |
| 5 | 最后才考虑重新登录 | — | 确认以上都正常仍失败，再执行 `gh auth login` |

### 取已有 token（无需重新登录）

```powershell
echo "protocol=https`nhost=github.com" | git credential fill
# 输出中的 password 字段就是可用 token
```

---

## 6. 本项目仓库信息

- remote：`https://github.com/ToFour/ToFour.github.io.git`
- 默认分支：`main`
- 最近 HEAD：`f9fb8a8`（2026-10-21）
- 推送验证：`git ls-remote origin` 成功

---

## 7. 三条可直接照抄的命令

```powershell
# 取端口
$port = (Select-String "$env:USERPROFILE\.config\clash\config.yaml" -Pattern "mixed-port").Line.Split(':')[1].Trim()

# gh 操作（token 从凭据管理器取）
$env:GH_TOKEN=(echo "protocol=https`nhost=github.com" | git credential fill | Select-String "password=").ToString().Split('=')[1]; $env:HTTPS_PROXY="http://127.0.0.1:$port"; & "C:\Program Files\GitHub CLI\gh.exe" api user --jq .login

# git 推送
git -c http.version=HTTP/1.1 -c http.proxy="http://127.0.0.1:$port" -c https.proxy="http://127.0.0.1:$port" push
```
