---
name: github-ops
description: GitHub 操作技能。当用户需要连接 GitHub、创建仓库、推送代码、查询仓库列表、配置代理/VPN、或调试 GitHub 连接问题时使用此 skill。适用于 Windows 环境，内置中国区网络问题解决方案。
agent_created: true
---

# GitHub 操作专家

**触发时机：** 用户要求连接 GitHub、推送代码到 GitHub、查询仓库、创建仓库、配置代理/VPN。

## 本 Skill 文件清单

| 文件 | 用途 |
|------|------|
| `SKILL.md` | 主流程控制（本文件） |
| `references/github-api.md` | REST API 端点、请求格式、常见错误 |

## 外部依赖

无外部 skill 依赖。

## 核心原则

1. **Python 优先：** Windows 环境下 curl 传中文或复杂 JSON 容易失败，API 操作用 Python。
2. **Token-in-URL 兜底：** Windows schannel SSL/TLS 经常连接失败，HTTPS URL 中嵌入 token 兜底。
3. **直推 GitHub：** origin 直接指向 GitHub，不经 Gitea 跳板。
4. **VPN 检查优先：** 操作前先检测网络连通性。

## 流程

### 第1步：检查网络连通性

```bash
curl -s -o /dev/null -w "%{http_code}" --connect-timeout 5 https://api.github.com
```

- **200** → 网络正常，继续
- **超时/连接失败** → 需要配置代理，进入"关键技术 → VPN/代理配置"

### 第2步：读取 Token

```bash
cat ~/.workbuddy/github_token
```

或

```bash
python -c "
with open('C:/Users/ZhuanZ（无密码）/.workbuddy/github_token', 'r') as f:
    print(f.read().strip())
"
```

当前用户：`cat0141`
Token 存储路径：`~/.workbuddy/github_token`

### 第3步：根据操作类型执行

#### 操作A：查询仓库列表

```bash
python -c "
import urllib.request, json
token = open('C:/Users/ZhuanZ（无密码）/.workbuddy/github_token').read().strip()
req = urllib.request.Request('https://api.github.com/user/repos?per_page=100', headers={
    'Authorization': 'token ' + token,
    'Accept': 'application/vnd.github.v3+json'
})
repos = json.loads(urllib.request.urlopen(req).read())
for r in repos:
    print(r['full_name'], r['html_url'])
"
```

#### 操作B：创建仓库

```bash
python -c "
import urllib.request, json
token = open('C:/Users/ZhuanZ（无密码）/.workbuddy/github_token').read().strip()
data = json.dumps({'name': '仓库名', 'description': '描述', 'private': False}).encode('utf-8')
req = urllib.request.Request('https://api.github.com/user/repos', data=data, headers={
    'Authorization': 'token ' + token,
    'Accept': 'application/vnd.github.v3+json',
    'Content-Type': 'application/json'
})
r = json.loads(urllib.request.urlopen(req).read())
print('URL:', r['html_url'])
print('Clone:', r['clone_url'])
"
```

#### 操作C：初始化本地仓库并推送到 GitHub

1. 进入目标目录
2. 初始化并提交：
   ```bash
   git init
   git add -A
   git commit -m "初始提交"
   ```
3. 添加远程并推送：
   ```bash
   # 添加 origin 指向 GitHub（带 token）
   git remote add origin https://cat0141:$(python -c "print(open(r'C:/Users/ZhuanZ（无密码）/.workbuddy/github_token').read().strip())")@github.com/cat0141/仓库名.git
   git push -u origin master
   ```
4. 如果已有远程但连接失败（schannel 错误）：
   ```bash
   # 改为带 token 的 URL
   git remote set-url origin https://cat0141:TOKEN@github.com/cat0141/仓库名.git
   git push origin master
   ```

#### 操作D：日常推送

```bash
git add -A
git commit -m "提交信息"
git push origin main
```

### 第4步：验证结果

- 创建/推送后访问对应 GitHub URL 确认成功
- 返回给用户结果汇总

## 关键技术

### VPN/代理配置（中国区）

GitHub 在中国大陆可能需要代理。检测步骤：

1. **测试连通性：**
   ```bash
   curl -s -o /dev/null -w "%{http_code}" --connect-timeout 5 https://api.github.com
   ```

2. **如果连接失败，检查系统代理：**
   ```bash
   echo $env:HTTP_PROXY
   echo $env:HTTPS_PROXY
   ```

3. **配置代理（如果有代理工具）：**
   ```bash
   git config --global http.proxy http://127.0.0.1:7890
   git config --global https.proxy http://127.0.0.1:7890
   ```

4. **取消代理（如果不需要了）：**
   ```bash
   git config --global --unset http.proxy
   git config --global --unset https.proxy
   ```

5. **curl 使用代理：**
   ```bash
   curl -x http://127.0.0.1:7890 https://api.github.com
   ```

### Windows schannel SSL/TLS 连接失败

**现象：** `schannel: failed to receive handshake, SSL/TLS connection failed`

**原因：** Windows Git 默认使用 schannel 作为 SSL 后端，在某些网络环境下握手失败。

**解决方案（按优先级）：**

1. **方案A（最快）：** URL 中嵌入 token
   ```bash
   git remote set-url origin https://用户名:TOKEN@github.com/用户名/仓库名.git
   ```

2. **方案B：** 切换 SSL 后端为 OpenSSL
   ```bash
   git config --global http.sslBackend openssl
   ```

3. **方案C：** 跳过 SSL 验证（不推荐，仅调试用）
   ```bash
   git config --global http.sslVerify false
   ```

### Windows curl JSON 解析失败

**现象：** GitHub API 返回 `"message": "Problems parsing JSON"`

**原因：** Windows shell 对 JSON 字符串中的引号转义处理不可靠。

**解决方案：** 用 Python 代替 curl，通过 `json.dumps()` 和 `encode('utf-8')` 构造请求体。

### 远程仓库配置

```bash
# 查看当前远程
git remote -v

# origin 直接指向 GitHub
# push URL 带 token（避免 schannel 连接失败）
# fetch URL 用干净形式（避免泄露 token）

# 设置方式
git remote set-url origin https://github.com/cat0141/仓库名.git
git remote set-url --push origin https://cat0141:TOKEN@github.com/cat0141/仓库名.git

# 日常操作
git push origin main
```

### Token 安全

- Token 存储在 `~/.workbuddy/github_token`，不要在对话中明文展示
- push URL 中的 token 仅用于操作执行，完成后建议清理
- 不要在 commit message 或日志中包含完整 token

## 规则

| # | 规则 | 违反后果 |
|---|------|---------|
| R1 | API 操作用 Python，不要用 curl | JSON 解析失败 |
| R2 | 不要在对话中明文展示 token | 安全风险 |
| R3 | 推送前检查远程仓库地址是否正确 | 推到错误仓库 |
| R4 | 操作前检测网络连通性 | 白白等待超时 |

## 反模式

| 反模式 | 为什么不好 | 正确做法 |
|--------|-----------|---------|
| 用 curl 传复杂 JSON | Windows shell 引号转义不可靠 | 用 Python urllib |
| 直接在对话中打印完整 token | 泄露到上下文 | 用 `$()` 引用变量 |
| 跳过连通性检测直接推送 | 连接失败后排查困难 | 先 ping GitHub API |
| 用 `--force` 推送 | 可能丢失远程提交 | 先 pull 再 push |
| 把所有远程都叫 origin | 分不清哪个是 GitHub | 远程命名要有意义 |

## 质量自检清单

操作完成后，逐项检查：

- [ ] 网络连通性已检测？
- [ ] Token 已正确读取且未明文暴露？
- [ ] 远程仓库地址正确（直接指向 GitHub）？
- [ ] 操作成功已验证（API 返回 200 或 git push 成功）？

## Skill 演进原则

| 变更类型 | 改哪里 |
|----------|--------|
| 新增操作类型 | `SKILL.md` |
| 新增 API 端点/格式 | `references/github-api.md` |
| VPN/代理方案更新 | `SKILL.md` 关键技术章节 |
| 新增错误模式/解决方案 | `references/github-api.md` |

## 版本历史

| 版本 | 日期 | 说明 |
|------|------|------|
| **v1.1** | 2026-05-27 | 移除 Gitea 跳板。origin 直接指向 GitHub，简化推送流程为单远程仓库。 |
| **v1.0** | 2026-05-27 | 初始版本。基于 douyin-script-expert 推送实战经验创建。内置：网络检测、token 管理、仓库 CRUD、推送流程、VPN/代理配置、Windows schannel 解决方案、双远程仓库最佳实践。 |
