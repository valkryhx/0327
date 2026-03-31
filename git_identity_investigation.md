# Git 提交身份与推送账号排查记录（脱敏）

> 本文档已做隐私脱敏处理，不包含真实邮箱全文、密码、Token 或其他敏感凭证。

## 1. 当时使用过的排查命令

### 1.1 查看提交中记录的作者/提交者信息

```bash
git log -1 --format='Author: %an <%ae>%nCommitter: %cn <%ce>'
```

用途：
- 查看最近一次提交里实际写入的 `author` / `committer`
- 用来判断提交里记录的邮箱是否正确

### 1.2 查看本地 Git 身份配置

```bash
git config --get user.name; git config --get user.email; git config --get credential.helper
```

用途：
- 查看当前仓库/当前环境正在使用的 Git 用户名与邮箱
- 同时确认凭证助手类型

### 1.3 尝试查看 GitHub CLI 登录状态

```bash
gh auth status -h github.com
```

用途：
- 如果本机安装了 `gh`，可用于查看 GitHub CLI 当前登录账号
- 本次环境中 `gh` 不存在，因此这条路不可用

### 1.4 查看远端与凭证相关配置

```bash
git remote get-url origin && git config --show-origin --get-all credential.helper && git config --show-origin --get credential.username && git config --show-origin --get credential.https://github.com.username
```

用途：
- 确认远端仓库地址
- 查看凭证助手从哪里生效
- 查看是否配置了用户名级别的凭证偏好

### 1.5 通过 Git 凭证系统解析 GitHub 用户名（不读取密钥内容）

```bash
python - <<'PY'
import subprocess
p = subprocess.run(
    ['git', 'credential', 'fill'],
    input='protocol=https\nhost=github.com\n\n',
    text=True,
    capture_output=True,
)
out = p.stdout.splitlines()
vals = {}
for line in out:
    if '=' in line:
        k, v = line.split('=', 1)
        vals[k] = v
print('credential_fill_exit=', p.returncode)
print('credential_username=', vals.get('username', ''))
print('credential_has_password=', 'yes' if 'password' in vals else 'no')
PY
```

用途：
- 让 Git 自己解析 `github.com` 的已保存凭证
- 仅确认是否存在 `username`，以及是否存在 `password/token`
- **不会在文档中记录真实 token/password 内容**

---

## 2. 排查思路

本次排查分成两个层面：

### 2.1 提交里写入的身份是什么

先确认最近一次提交中的：
- `Author`
- `Committer`

因为 GitHub 页面展示的提交身份，首先取决于提交对象里写入的姓名和邮箱。

如果这里的邮箱写错了，那么即使 push 成功，提交页展示出来的身份也可能不符合预期。

### 2.2 实际 push 使用的是哪个 GitHub 账号

提交邮箱不等于 push 认证账号。

因此需要继续确认：
- 远端是哪个 GitHub 仓库
- 当前 Git 使用什么凭证助手
- Git 为 `github.com` 解析出来的用户名是谁

这一步的目标不是读取密钥，而是确认：
- push 时走的是哪套本机凭证
- 对应的是哪个 GitHub 用户名

---

## 3. 当时得到的结论（已脱敏）

### 3.1 提交邮箱配置存在格式错误

排查发现：
- 本地 `git config user.email` 配置值格式异常
- 最近一次提交中的 `Author` / `Committer` 邮箱也继承了这个错误配置

文档中不保留错误邮箱全文，只保留结论：
- **邮箱字符串格式有误，需要修正后重写提交**

### 3.2 push 使用的是已保存的 GitHub 凭证

排查发现：
- Git 远端指向 GitHub 仓库
- 凭证助手为 Git Credential Manager 类方案
- Git 为 `github.com` 解析到了已保存的用户名
- 且存在对应密钥（仅确认存在，不展示内容）

结论：
- **本次 push 使用的是本机已保存的 GitHub 凭证完成认证**
- **可确认 push 账号与仓库所属用户名一致**

---

## 4. 处置过程

### 4.1 修正后续提交使用的本地邮箱

执行思路：
- 更新本地 Git `user.email`
- 保持 `user.name` 不变

对应命令模式：

```bash
git config user.name "<your-github-username>"
git config user.email "<correct-email>"
```

### 4.2 重写最近一次提交的作者与提交者信息

由于错误邮箱已经写入最近一次提交，因此只改配置还不够，必须重写该提交。

执行思路：
- 读取原提交 message
- 用正确邮箱 amend 最近一次提交
- 同时更新 author / committer

对应命令模式：

```bash
msg=$(git log -1 --format=%B)
GIT_COMMITTER_NAME="<your-github-username>" \
GIT_COMMITTER_EMAIL="<correct-email@example.com>" \
git commit --amend --author="<your-github-username> <correct-email@example.com>" -m "$msg"
```

> 实际执行时，author 参数的格式应为：
> `Name <email@example.com>`

### 4.3 将重写后的提交强制推送到远端

因为 amend 会生成新的 commit hash，远端历史需要同步更新，所以必须推送覆盖远端对应分支。

对应命令：

```bash
git push --force-with-lease origin master
```

之所以使用 `--force-with-lease`：
- 比 `--force` 更安全
- 可避免误覆盖他人刚刚推送的新提交

---

## 5. 复核命令

修正完成后，可用以下命令再次确认：

```bash
git log -1 --format='Commit: %H%nAuthor: %an <%ae>%nCommitter: %cn <%ce>'
```

用途：
- 核对最新提交的 author / committer 是否已变为正确邮箱

---

## 6. 隐私处理说明

本记录刻意不保留以下内容：
- 真实邮箱全文
- 错误邮箱全文
- 凭证中的 password / token
- 本机凭证缓存的原始返回内容

只保留：
- 排查命令
- 排查思路
- 风险判断
- 处置步骤

这样可以保证后续复盘可用，同时避免敏感信息泄露。
