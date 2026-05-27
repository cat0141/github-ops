# GitHub REST API 参考

**用途：** GitHub REST API 端点、请求格式、常见错误。

## 基础信息

- **API 地址：** `https://api.github.com`
- **认证方式：** `Authorization: token <token>`
- **请求头：** `Accept: application/vnd.github.v3+json`
- **分页：** `per_page=100`，`page=N`

## 常用端点

### 仓库管理

| 操作 | 方法 | 端点 | 说明 |
|------|------|------|------|
| 列出所有仓库 | GET | `/user/repos?per_page=100&sort=updated` | 返回最近更新的仓库 |
| 获取单个仓库 | GET | `/repos/{owner}/{repo}` | 仓库详情 |
| 创建仓库 | POST | `/user/repos` | Body: `{"name": "xxx", "description": "xxx", "private": false}` |
| 删除仓库 | DELETE | `/repos/{owner}/{repo}` | 谨慎使用 |
| 更新仓库 | PATCH | `/repos/{owner}/{repo}` | Body: 需更新的字段 |

### 提交与推送

| 操作 | 方法 | 端点 | 说明 |
|------|------|------|------|
| 列出提交 | GET | `/repos/{owner}/{repo}/commits` | 提交历史 |
| 创建分支 | POST | `/repos/{owner}/{repo}/git/refs` | Body: `{"ref": "refs/heads/branch", "sha": "xxx"}` |
| 创建文件 | PUT | `/repos/{owner}/{repo}/contents/{path}` | Body: `{"message": "xxx", "content": "base64内容", "branch": "main"}` |

### 其他

| 操作 | 方法 | 端点 | 说明 |
|------|------|------|------|
| 获取当前用户 | GET | `/user` | 用户信息 |
| 列出用户 | GET | `/users` | 所有用户 |

## Python 请求模板

### GET 请求

```python
import urllib.request, json

token = open('C:/Users/ZhuanZ（无密码）/.workbuddy/github_token').read().strip()
req = urllib.request.Request(
    'https://api.github.com/端点',
    headers={
        'Authorization': 'token ' + token,
        'Accept': 'application/vnd.github.v3+json'
    }
)
result = json.loads(urllib.request.urlopen(req).read())
print(result)
```

### POST 请求

```python
import urllib.request, json

token = open('C:/Users/ZhuanZ（无密码）/.workbuddy/github_token').read().strip()
data = json.dumps({
    'name': '仓库名',
    'description': '描述',
    'private': False
}).encode('utf-8')

req = urllib.request.Request(
    'https://api.github.com/user/repos',
    data=data,
    headers={
        'Authorization': 'token ' + token,
        'Accept': 'application/vnd.github.v3+json',
        'Content-Type': 'application/json'
    }
)
result = json.loads(urllib.request.urlopen(req).read())
print(result)
```

### DELETE 请求

```python
import urllib.request, json

token = open('C:/Users/ZhuanZ（无密码）/.workbuddy/github_token').read().strip()
req = urllib.request.Request(
    'https://api.github.com/repos/用户名/仓库名',
    method='DELETE',
    headers={
        'Authorization': 'token ' + token,
        'Accept': 'application/vnd.github.v3+json'
    }
)
resp = urllib.request.urlopen(req)
print(resp.status)  # 204 表示删除成功
```

## 常见错误

| 错误 | 状态码 | 原因 | 解决 |
|------|--------|------|------|
| `Problems parsing JSON` | 400 | Windows shell 引号转义问题 | 改用 Python |
| `Not Found` | 404 | 仓库不存在或权限不足 | 检查仓库名和 token 权限 |
| `Requires authentication` | 401 | Token 无效或过期 | 刷新 token |
| `Already exists` | 422 | 仓库名已存在 | 换一个仓库名 |
| `Bad credentials` | 401 | Token 错误或格式不对 | 检查 token 文件 |
| 连接超时 | - | 网络不通 | 配置代理/VPN |

## Token 信息

- **存储路径：** `~/.workbuddy/github_token`
- **当前用户：** `cat0141`
- **类型：** Personal Access Token (PAT)
- **权限：** repo, workflow, admin:org

## 版本记录

| 版本 | 日期 | 说明 |
|------|------|------|
| v1.0 | 2026-05-27 | 初始版本 |
