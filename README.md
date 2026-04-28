# Arc Bot

Arc Network 每日积分任务自动化脚本（多账号版）。脚本使用 Playwright 启动 Chromium，登录 `https://community.arc.network` 后依次完成内容阅读、视频观看、活动注册、论坛发帖和评论，并在本地保存账号状态与浏览器 session。

> 使用前请确认你的操作符合 Arc Network、Gmail、代理服务等相关平台规则。本项目更适合作为 Playwright 自动化学习和个人任务管理示例，不建议用于滥用、多账号刷量或垃圾内容发布。

## 功能概览

- 多账号串行执行，避免同时打开多个浏览器上下文。
- 通过 Gmail IMAP 自动读取 Arc/Circle magic link 完成登录。
- 登录成功后将 session 保存到 `sessions/`，后续运行优先复用。
- 支持 HTTP、HTTPS、SOCKS5 代理；SOCKS5 带认证会自动转成本地 HTTP 代理供 Chromium 使用。
- 自动执行每日任务：
  - 阅读 5 篇 Content 文章。
  - 观看 1 个 Content 视频。
  - 注册可用 Events。
  - 在 Discussions/Forum 发 1 篇帖子。
  - 评论 2 条帖子。
- 记录已读文章、已注册活动和最近运行时间到 `arc_state.json`。
- 将日志和失败截图输出到上级目录的 `security-reports/`。

## 文件说明

```text
arc_daily.py        主脚本
requirements.txt   Python 依赖
accounts.txt       Arc 登录邮箱列表，每行一个
gmail_passes.txt   Gmail 应用专用密码，每行一个
proxies.txt        代理列表，每行一个，可写 none
sessions/          自动生成，保存各账号浏览器 session
arc_state.json     自动生成，保存任务状态
```

注意：`accounts.txt`、`gmail_passes.txt`、`proxies.txt` 按行一一对应。例如第 1 行邮箱使用第 1 行 Gmail 应用密码和第 1 行代理。

## 环境要求

- Python 3.10 或更新版本。
- 可访问 Gmail IMAP。
- Chromium 浏览器由 Playwright 安装。
- 如果使用 Gmail magic link 登录，每个 Gmail 账号需要：
  - 开启两步验证。
  - 生成应用专用密码。
  - 在 Gmail 设置中启用 IMAP。

## 安装

建议先创建虚拟环境：

```bash
python -m venv .venv
```

Windows PowerShell：

```powershell
.\.venv\Scripts\Activate.ps1
```

macOS / Linux：

```bash
source .venv/bin/activate
```

安装依赖和 Chromium：

```bash
pip install -r requirements.txt
playwright install chromium
```

Linux 服务器如缺少浏览器系统依赖，可再执行：

```bash
playwright install-deps chromium
```

## 配置

### `accounts.txt`

每行填写一个 Arc 登录邮箱：

```text
alice@gmail.com
bob@gmail.com
```

### `gmail_passes.txt`

每行填写一个 Gmail 应用专用密码，与 `accounts.txt` 同行对应：

```text
abcd efgh ijkl mnop
wxyz abcd efgh ijkl
```

### `proxies.txt`

每行填写一个代理，与 `accounts.txt` 同行对应。支持：

```text
http://user:pass@host:port
https://user:pass@host:port
socks5://user:pass@host:port
http://host:port
none
```

如果 `proxies.txt` 不存在，脚本会全部直连；如果代理行数少于账号数，缺少的账号也会直连。直连可能增加账号风控风险。

## 运行

首次部署检查：

```bash
python arc_daily.py --setup
```

直接启动守护进程：

```bash
python arc_daily.py
```

当前脚本不是“只跑一次就退出”，而是启动后立即执行一轮任务，然后每隔 24 小时继续执行下一轮。需要停止时，在终端按 `Ctrl+C`。

## 执行流程

1. 读取 `accounts.txt`、`gmail_passes.txt`、`proxies.txt`。
2. 启动一个无头 Chromium 浏览器。
3. 对每个账号依次执行：
   - 尝试加载 `sessions/<邮箱前缀>.json`。
   - 如果 session 失效，进入 Arc 登录页，提交邮箱。
   - 通过 Gmail IMAP 查找 magic link 并打开。
   - 登录成功后保存 session。
   - 读取任务前积分。
   - 完成阅读、视频、活动注册、发帖、评论。
   - 读取任务后积分。
   - 更新 `arc_state.json` 和 session。
4. 所有账号完成后打印积分汇总。
5. 等待 24 小时后自动开始下一轮。

账号之间会随机等待 30-90 秒，单个任务内也有随机滚动和等待，用来模拟更自然的浏览行为。

## 日志与截图

脚本会在项目上级目录创建 `security-reports/`，常见输出包括：

```text
security-reports/arc_daily_YYYY-MM-DD.log
security-reports/profile_<email-prefix>.png
security-reports/login_failed_<email-prefix>.png
security-reports/post_failed_<email-prefix>.png
security-reports/error_<email-prefix>.png
```

如果积分识别、登录、发帖或任务执行失败，可以优先查看当天日志和相关截图。

另外，登录成功截图会保存到项目目录：

```text
login_result_<email-prefix>.png
```

## 定时运行

### Linux / macOS

`python arc_daily.py --setup` 会尝试添加 cron：

```cron
0 9 * * * python /path/to/arc_daily.py >> /path/to/security-reports/arc_cron.log 2>&1
```

如果系统没有 `crontab`，请手动配置。由于脚本自身已经包含 24 小时循环，通常更推荐二选一：

- 使用脚本内置守护进程：长期运行 `python arc_daily.py`。
- 使用系统定时任务：改造脚本为单轮运行后退出，或确保定时任务不会重复启动多个进程。

### Windows

可以使用“任务计划程序”每日启动：

```text
python D:\path\to\arc_daily.py
```

同样需要注意不要重复启动多个长期运行的脚本进程。

## 常见问题

### 提示 `accounts.txt 中没有有效邮箱`

检查 `accounts.txt` 是否只有注释或空行。有效行不能以 `#` 开头。

### 提示 `gmail_passes.txt 只有 N 条`

`gmail_passes.txt` 的有效行数少于账号数。请按账号顺序补齐 Gmail 应用专用密码。

### 收不到 magic link

检查：

- Gmail 是否开启 IMAP。
- 应用专用密码是否正确，不是普通登录密码。
- 邮件是否进入垃圾邮件或其他分类。
- Arc/Circle 邮件是否延迟到达。
- 代理网络是否影响登录页面提交。

### SOCKS5 代理不能用

脚本会把带认证的 `socks5://user:pass@host:port` 转成本地 HTTP 代理。请确认已安装 `python-socks`：

```bash
pip install -r requirements.txt
```

### 积分显示为 `N/A`

脚本通过页面选择器猜测积分位置。如果 Arc 页面结构变化，积分读取可能失败，但任务仍可能已执行。查看 `security-reports/profile_<email-prefix>.png` 确认页面状态。

## 数据与安全

- `gmail_passes.txt` 存放敏感应用密码，不要提交到公开仓库。
- `sessions/` 内保存登录态，也不要公开。
- `arc_state.json` 记录账号任务状态，通常也不应公开。
- 建议把以下内容加入 `.gitignore`：

```gitignore
gmail_passes.txt
accounts.txt
proxies.txt
sessions/
arc_state.json
login_result_*.png
../security-reports/
```

## 依赖

```text
playwright>=1.40.0
python-socks>=2.0.0
```
