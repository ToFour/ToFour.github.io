# AGENTS.md

## GitHub 连接

本项目推送 GitHub 必须走 Clash 代理，直连 443 不通。

**必读**：[docs/github-connection.md](docs/github-connection.md)

快速三条命令：

```powershell
$port = (Select-String "$env:USERPROFILE\.config\clash\config.yaml" -Pattern "mixed-port").Line.Split(':')[1].Trim()
$env:GH_TOKEN=(echo "protocol=https`nhost=github.com" | git credential fill | Select-String "password=").ToString().Split('=')[1]; $env:HTTPS_PROXY="http://127.0.0.1:$port"; & "C:\Program Files\GitHub CLI\gh.exe" api user --jq .login
git -c http.version=HTTP/1.1 -c http.proxy="http://127.0.0.1:$port" -c https.proxy="http://127.0.0.1:$port" push
```

不要轻易 `gh auth login`，token 已存在 Windows 凭据管理器中。
