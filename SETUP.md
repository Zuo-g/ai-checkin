# AI 五平台自动签到 · GitHub Actions 部署指南

**平台**：Trae · WorkBuddy · Qoder · ZCode · MiniMax
**推送**：企业微信群机器人 + PushPlus
**成本**：公开仓库完全免费

---

## 为什么不建议用青龙？

这台轻量服务器的容器出网 TCP 被拦（`Errno 113 Host is unreachable`），而所有签到脚本都靠 HTTPS 调平台 API，**在青格里一个都跑不起来**。GitHub 的 runner 网络完好，不存在这个问题。

---

## 部署步骤（10 分钟）

### 1. 新建仓库

GitHub → **New repository**
- 名称随意，比如 `ai-checkin`
- **选 Public**（私有仓库每月有 2000 分钟免费额度，这个任务约 60 分钟/月，也够用；但 Public 最省心）
- 不要勾选 README

### 2. 上传 workflow

仓库里新建目录结构：

```
.github/workflows/checkin.yml
```

把本包里的 `checkin.yml` 内容粘进去保存。

GitHub 网页新建文件时，如果不给 `.yml` 后缀会自动加 `.txt`，**注意确认文件名是 `checkin.yml`**。

### 3. 配置 Secrets

**Settings → Secrets and variables → Actions → New repository secret**

逐条添加：

| Secret | 值 | 必填 |
|---|---|:-:|
| `TRAE_REFRESH_TOKEN` | Trae refreshToken（单账号） | ⭕ |
| `TRAE_ACCOUNTS` | accounts 数组整行 JSON（多账号，优先于上一条） | ⭕ |
| `WORKBUDDY_REFRESH_TOKEN` | 每行一个 `手机号:AT:RT` | ⭕ |
| `QODER_TOKENS` | 国内版，多账号换行或 `&` | ⭕ |
| `QODER_GLOBAL_TOKENS` | 国际版（每天 100 Credits） | ⭕ |
| `ZCODE_CN_TOKENS` | 国内版 | ⭕ |
| `ZCODE_INTL_TOKENS` | 国际版 | ⭕ |
| `MINIMAX_TOKEN` | MiniMax Code token | ⭕ |
| `MINIMAX_USER_ID` | storage.json 里的 `realUserID` | ⭕ |
| `MINIMAX_UUID` | 可选 | ➖ |
| `MINIMAX_DEVICE_ID` | 可选 | ➖ |
| `WECOM_WEBHOOK` | `https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxx` | ⭕ |
| `QYWX_TOKEN` | 企业微信 key（备用，与 WECOM_WEBHOOK 二选一即可） | ⭕ |
| `PUSHPLUS_TOKEN` | PushPlus token（个人微信） | ⭕ |
| `PLUSPLUS_TOKEN` | PushPlus token（备用命名） | ⭕ |

> **推送至少填一个**，否则手机收不到任何通知。
> 不用的平台留空即可，脚本会自动跳过并在结果表里标注「⏭️ 跳过」。

### 4. 启用 workflow

**Actions** 标签页 → 左侧找到 `AI 五平台每日签到` → 点 **Enable workflow**

### 5. 手动跑一次验证

Actions → 选中该 workflow → 右上角 **Run workflow** → 等 1~2 分钟

看两处：
- **Summary** 区域：五个平台的成功/失败表格
- **手机**：应该收到企业微信或 PushPlus 推送

---

## 怎么拿各平台的 Token

| 平台 | 命令 | 说明 |
|---|---|---|
| Trae | `python trae_get_token.py --accounts` | 在装有 Trae 客户端的 Windows 上，自动解密 |
| WorkBuddy | `python workbuddy_daily.py --export-env --save` | 桌面端 v5.3.8+ 登录后运行 |
| Qoder | `python qoder_login.py [--region global]` | 浏览器授权一次，输出 `access\|refresh` |
| ZCode | `python zcode_login.py [--region intl]` | 输出 `jwt\|device_mid` |
| MiniMax | `python minimax_checkin.py --export-env --save` | 客户端重新登录后运行 |

登录辅助脚本都在对应仓库里，先 `git clone` 到本地跑。

---

## 三个必须知道的点

### 1. 缓存机制不能删

`refreshToken` 每次续期都会轮换，旧值立刻作废。而 Actions 的工作目录跑完就丢，**必须靠 `actions/cache` 把新 token 带到下一次运行**。

workflow 里那段 `actions/cache` 一行都不能删。删了的表现是：第一次正常，第二次开始全部 401。

### 2. 缓存 7 天会被清

如果 workflow 连续超过一周没成功运行，恢复不到缓存，就会退回用 Secret 里的旧 token，需要重新跑一次 `*_login.py` 或 `--export-env --save` 更新 Secret。

正常每天定时跑就不会触发这个。

### 3. 定时不是准点的

GitHub 的 schedule 在高峰期可能延迟几分钟到十几分钟。另外 GitHub 免费账户的定时任务有排队延迟，繁忙时可能更久。

workflow 里的 cron 已经错开了整点（13 分），避免和全网其他定时任务撞在一起。

---

## 常见问题

| 现象 | 原因 | 处理 |
|---|---|---|
| 全部 401 | Secret 值带了引号 | 去掉引号，Secret 是纯文本 |
| 第二次运行开始全 401 | 缓存被清了 | 重新获取 token 更新 Secret |
| Trae 报 `9074` | 设备号不对 | 从 `storage.json` 键名 `iCubeAuthInfo://icube-dc:` 后取真实设备号 |
| Trae 报 `9095` | 一个设备号签了多个号 | 一个设备号只能配一个账号 |
| Qoder 一直 pending | 缺 UMID 设备身份 | 在装有 Qoder 客户端的机器上跑一次取 token 流程 |
| 手机收不到推送 | Secret 没填或填错 | 检查 `WECOM_WEBHOOK` / `PUSHPLUS_TOKEN` |
| MiniMax 报 401 | `MINIMAX_USER_ID` 填错 | 必须是 storage.json 的 `realUserID`，不是 JWT 的 `user.id` |
| 脚本报 ModuleNotFoundError | 依赖没装全 | workflow 里已装 `requests` + `python-dotenv`，如还缺可在该 step 追加 |

---

## 关于安全性

- token 只存在 GitHub Secrets 里，不进仓库、不进日志
- workflow 用 `permissions: contents: read`，只读权限
- 脚本从各项目 `main` 分支拉取。如果你介意供应链风险，可以改成下载后本地审计，或把文件 fork 到自己仓库固定版本

---

## 合规

全部为第三方逆向实现，与各平台官方无关。批量代签在用户协议里属灰色地带，建议只用自己的账号、单账号规模使用。
