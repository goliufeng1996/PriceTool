# 国内网络环境下向 GitHub 推送代码的方法

> 适用场景：网络能正常上网（如访问百度没问题），但**无法直连 github.com / api.github.com**（克隆、推送卡住或超时）。

---

## 1. 问题现象

- `git clone` / `git push` 卡住，最终报 `Connection timed out` / `Could not resolve host`
- `curl https://github.com` 超时返回 `HTTP 000`
- winget / choco 下载 GitHub 资源失败（如 `0x80072efd`）
- 浏览器却可能能打开 GitHub（因为走了不同的代理/VPN）

根因通常是：**DNS 能解析出正确 IP，但 TCP 到 GitHub 的 443 端口被网络层阻断**，且没有本地代理。

## 2. 快速诊断

```bash
# 1) DNS 是否正常（能解析出 IP 即为正常）
nslookup github.com

# 2) 直连是否可达（HTTP 000 = 不通）
curl -sS -m 10 -o /dev/null -w "HTTP %{http_code}\n" https://github.com

# 3) 是否有本地代理可用（常见代理端口）
netstat -ano | grep -E "127.0.0.1:(7890|7897|10809|10808|8888)"
```

若第 1 步能解析出 IP、第 2 步 `HTTP 000`、第 3 步无监听端口，即属于「DNS 正常 + TCP 阻断 + 无代理」，适用下文方案。

## 3. 解决方案：反向代理加速

使用 URL 前缀式第三方反向代理，把 GitHub 请求中转出去。常用两个（可用性会变化，**使用前先实测**）：

```bash
# 实测可达性
for h in ghfast.top gh-proxy.com; do
  curl -sS -m 8 -o /dev/null -w "$h -> HTTP %{http_code}\n" "https://$h"
done
```

用法：在原始 GitHub URL 前面拼上代理前缀即可：

```
原始:  https://github.com/<user>/<repo>.git
代理:  https://ghfast.top/https://github.com/<user>/<repo>.git
```

支持的场景：下载 release 文件、`git clone`、`git pull`、`git push` 均可。

> 注意：`ghfast.top` 主要中转 `github.com` 的 git/文件下载，**不中转 `api.github.com`**；如需中转 API 可用 `gh-proxy.com`。

## 4. 完整推送流程

```bash
# 1) 初始化并提交（本地，不需要网络）
git init
git add -A
git commit -m "Initial commit"

# 2) 通过代理推送（<token> 为 Personal Access Token）
git push "https://<user>:<token>@ghfast.top/https://github.com/<user>/<repo>.git" main
```

> 用「一次性 URL 带 token」的方式推送，token 只出现在命令行，不会写进 `.git/config`，避免凭据落盘。

## 5. Token 的生成与权限

1. 打开 `https://github.com/settings/tokens`
2. **Generate new token → Generate new token (classic)**
3. 权限勾选 **`repo`**（整个「Full control of private repositories」那一项）
4. 生成后复制 `ghp_...` 开头的字符串

⚠️ 常见报错对照：

| 报错 | 原因 |
|------|------|
| `Write access to repository not granted` | token 缺少写权限（没勾 `repo`，或 fine-grained token 的 Contents 是只读） |
| `Invalid username or password` / 401 | token 无效或已撤销 |
| `Repository not found` | 仓库不存在，或 token 无权访问该私有仓库 |

## 6. 配置 credential helper（免手输 token）

用 `store` 方式让 git 记住凭据（凭据按 host 区分，此处 host 是 `ghfast.top`）：

```bash
# 使用 store 作为凭据助手
git config --global credential.helper store

# 写入凭据（明文存储，注意文件权限）
printf 'https://<user>:<token>@ghfast.top\n' > ~/.git-credentials
```

配置后，push/pull 直接对 `ghfast.top` 生效，无需再带 token：

```bash
git remote add origin "https://ghfast.top/https://github.com/<user>/<repo>.git"
git push -u origin main   # 之后 git push 即可，不再提示输入
```

> 说明：`store` 是明文存储，安全级别低于 Git Credential Manager（加密存 Windows 凭据管理器）。但 GCM 的 OAuth 依赖直连 github.com（本场景不通），故 `store` 更实用。

### 6.1 两个容易踩的坑

**坑 1：凭据的 host 是代理域名，不是 github.com。** remote URL 是 `https://ghfast.top/https://github.com/...`，git 识别到的 host 是 `ghfast.top`，所以凭据要写 `...@ghfast.top`，而不是 `...@github.com`。

**坑 2：PortableGit / Git for Windows 自带的 `helper-selector` 会覆盖你的设置。** 有的发行版在系统级配置了 `credential.helper=helper-selector`，它会根据 `credential.helperselector.selected` 自动把 `credential.helper` 改回 `manager`（GCM），导致 `store` 不生效、push 时卡在弹窗。

排查与解决：

```bash
# 看所有来源的 credential 配置
git config --list --show-origin | grep -i credential
```

若看到系统级（`.../etc/gitconfig`）有 `credential.helper=helper-selector`，直接把它改掉（或删除该 `[credential]` 段），换成 `store`，再确认全局也是 `store` 即可。

**坑 3：`credential.useHttpPath` 默认 false。** git 查询凭据时**不带 path**，所以 store 条目写 `https://<user>:<token>@ghfast.top`（不带 path）就够了，能匹配该 host 下所有仓库。

## 7. 安全注意事项

1. **token 会经过第三方代理中转**，存在泄露风险，务必：
   - 用后及时在 `https://github.com/settings/tokens` 撤销；
   - 过期时间设短一点（如 7 天）。
2. 更安全可用 **fine-grained token**：只授权单个仓库 + `Contents: Read and write`，泄露影响面更小。
3. 代理域名（ghfast.top 等）可用性不稳定，长时间用不了时换一个镜像或改用真实代理/VPN。

## 8. 常见问题

- **Q：push 还是超时？** 先 `curl` 实测代理域名是否还可用，换一个镜像。
- **Q：`gh auth login` 卡住？** gh CLI 的认证走 `api.github.com`（被阻断），本场景改用 `git + HTTPS token`，不要用 `gh auth`。
- **Q：以后只想拉取不想推送？** 只读场景可用不带 token 的 `git clone https://ghfast.top/https://github.com/<user>/<repo>.git`（仅公开仓库）。
