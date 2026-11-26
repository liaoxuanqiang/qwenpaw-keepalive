# qwenpaw-keepalive

通过 GitHub Actions 定时向 QwenPaw 平台发送沙箱心跳，减少沙箱因长期未使用而停止的情况。脚本会登录账号并自动枚举该账号下的全部沙箱，因此新增沙箱后通常不需要修改脚本。

## 工作方式

`keepalive.py` 使用 `QP_ACCOUNT` 和 `QP_PASSWORD` 登录平台，获取账号下的沙箱列表，并逐个调用官方心跳接口。如果官方心跳未返回 HTTP 200，脚本会尝试获取沙箱的 `console_token`，再调用备用的 `push-messages` 接口。

工作流位于 `.github/workflows/qwenpaw-keepalive.yml`：

- 默认每 6 小时运行一次（UTC 时间每逢第 23 分钟），也支持在 GitHub Actions 页面手动触发。
- 默认配置了两个账号任务。每个任务分别运行脚本，并处理对应账号下的所有沙箱。
- `acc1` 任务成功执行后会把 UTC 运行时间写入 `.last-run` 并推送到仓库；该提交也用于保持仓库活跃。

## 配置 GitHub Secrets

在仓库的 **Settings → Secrets and variables → Actions** 中添加以下 Repository secrets：

| Secret 名称 | 用途 |
| --- | --- |
| `QP1_ACCOUNT` | 第一个 QwenPaw 账号 |
| `QP1_PASSWORD` | 第一个账号的密码 |
| `QP2_ACCOUNT` | 第二个 QwenPaw 账号 |
| `QP2_PASSWORD` | 第二个账号的密码 |

工作流中的 `acc1` 使用 `QP1_ACCOUNT` / `QP1_PASSWORD`，`acc2` 使用 `QP2_ACCOUNT` / `QP2_PASSWORD`。如只需运行一个账号，可在 `.github/workflows/qwenpaw-keepalive.yml` 的 `strategy.matrix.include` 中移除不需要的任务，并相应调整该任务的 secrets 映射。不要把账号或密码直接写入仓库文件。

## 启用与手动运行

1. 将仓库添加到自己的 GitHub 账号，并在仓库设置中配置所需的 Actions secrets。
2. 确认仓库允许 GitHub Actions 运行；工作流需要 `contents: write` 权限，才能提交 `.last-run`。
3. 打开仓库的 **Actions → qwenpaw-keepalive**，选择 **Run workflow** 可手动触发。
4. 在该工作流的运行记录中查看日志。脚本会输出各沙箱心跳及备用接口的 HTTP 状态；缺少凭据、登录失败或未找到沙箱时，任务会失败。

## 本地运行

脚本仅使用 Python 标准库，无需安装第三方依赖。设置环境变量后运行：

```powershell
$env:QP_ACCOUNT = "你的账号"
$env:QP_PASSWORD = "你的密码"
python keepalive.py
```

使用 bash 时：

```bash
export QP_ACCOUNT="你的账号"
export QP_PASSWORD="你的密码"
python3 keepalive.py
```

退出码 `0` 表示保活函数最终返回成功，`1` 表示最终返回失败，`2` 表示缺少必需的环境变量。
