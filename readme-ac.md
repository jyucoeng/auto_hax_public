# Auto HUX 使用说明（给朋友 / acquaintances）

自动在 [hus.co.id](https://hus.co.id) 抢 / 创建 VPS 的工具。你**不需要改代码**，只用自己的 GitHub Actions 跑作者维护的私有代码库即可。

---

## 一、整体架构

| 组件 | 说明 | 你是否需要 |
|------|------|-----------|
| 私有代码库 `jyucoeng/auto_hax` | `hus_create_vps.py` 等核心脚本 | 只申请**只读**权限 |
| 你自己的公开库（工作流） | 只放 `.github/workflows/hax_renew.yml` | 从 `jyucoeng/auto_hax_public` 复刻一份 |
| 你的 GitHub Secrets | 你自己的账号 / 代理等凭据 | 必需，**仅在你自己库里，别人看不到** |
| cron-job.org | 按你北京时间抢机点准点触发工作流 | 推荐（比 GitHub 原生定时更准时，不被排队） |

> 核心原则：**代码是作者的，配置/凭据是你自己的**。工作流里的配置（env）优先级高于私有库的 `hus_config.txt`，所以你完全不用碰私有库里的文件。

---

## 二、前置准备

1. **hus.co.id 账号**：手机号（格式 `+国家码-号码`，如 `+44-7386113027`）。
2. **Telegram Bot**：用 [@BotFather](https://t.me/BotFather) 创建一个，拿到 `Bot Token` 和你的 `chat_id`（给自己发 `/start` 后可用 [@userinfobot](https://t.me/userinfobot) 查 chat_id）。
3. **Telegram 用户会话**（用于 HAX 登录确认）：去 <https://my.telegram.org> 申请 `API ID` / `API Hash`，然后用本机生成一次 session 字符串：
   ```bash
   python3 -c "from telethon.sync import TelegramClient; c=TelelegramClient('tmp',你的API_ID,你的API_HASH); c.start(); print(c.session.save()))"
   ```
   把打印出的长字符串保存好——它和上面的 API ID / Hash 一起，作为 `HUS_BATCH` 每账号的**第 4~6 项**填入（不再有单独的全局 TG Secret）。生成后删除本机产生的 `tmp.session` 文件。
4. **（可选）代理**：只配一个 `HAX_HY2_PROXY_URL`（sing-box 代理源，如 hy2/vmess 订阅或配置文本）。工作流会据此生成 sing-box 配置并启动本地 SOCKS5，浏览器(http)与 TG 自动共用该出口，无需再单独配 TG 代理。hus.co.id 若在你网络下被拦，建议配置。

---

## 三、步骤

### 1. 准备你的公开库
- 直接用 `jyucoeng/auto_hax_public`，或 **复刻（fork）一份到你名下**（推荐各自独立，方便改自己的抢机时间点）。
- 确认工作流文件位于 `.github/workflows/hax_renew.yml`，默认分支为 `main`，且 **Actions 已启用**（仓库 Settings → Actions → 允许）。

### 2. 申请私有库只读权限
- 让作者把你的 GitHub 账号加为 `jyucoeng/auto_hax` 的**只读协作者**；或作者给你一个对该库有读权限的 PAT（作为下面的 `PRIVATE_REPO_TOKEN`）。

### 3. 配置 Secrets（你公开库 → Settings → Secrets and variables → Actions）
把下面这些**你自己的**值填进去（名称必须完全一致）：

| Secret 名 | 内容 | 必填 |
|-----------|------|------|
| `PRIVATE_REPO_TOKEN` | 对 `jyucoeng/auto_hax` 有读权限的 PAT（classic，需 `repo` 或 fine-grained `Contents:read`） | ✅ |
| `HUS_BATCH` | 账号串：`phone,tg_bot_token,tg_chat_id,tg_api_id,tg_api_hash,tg_session`；多账号用 `;` 分隔。**必须填全 6 项**（后三项即 TG 用户会话，用于登录确认），不再有单独的全局 TG Secret | ✅ |
| `HUS_VPS_SPECS` | 可选，`phone,datacenter,os`；多账号 `;` 分隔 | ⬜ |
| `HAX_HY2_PROXY_URL` | 可选，sing-box 代理源（hy2/vmess/trojan 等订阅或配置文本）。**设了它，浏览器(http)和 TG 会共用其生成的本地 SOCKS5 出口**，无需再单独配 TG 代理 | ⬜ |
| `SOCKS_PORT` | 可选，本地 SOCKS5 端口，默认 `10808`，一般不用改 | ⬜ |
| `HUS_STATE_SALT` | 可选，reg_state.json key 的加盐因子；设置后即使状态文件泄露也无法反解手机号。本地用 `decode_state.py` 查看时需提供**同一盐**。日志中只显示「已加盐+盐长度」，绝不打印盐原文 | ⬜ |

> `HUS_BATCH` 若只填前 3 项（phone,token,chat），则 TG 用户会话改用上面单独的 `HUS_TG_*` 三项。

### 4. 触发工作流

#### A. 手动试跑（先验证链路）
公开库 → **Actions** → 选 `hax_renew` → **Run workflow**。首次建议手动跑一次，确认私有库能检出、依赖能装、到点能创建。运行日志和截图在 **Artifacts**（`hax-screenshots-*` / `hax-log-*`）。

#### B. cron-job.org 准点拉起（正式用法）
1. 注册 <https://console.cron-job.org>。
2. **Create Job**，填写：
   - **URL**：`https://api.github.com/repos/<你的公开库全名>/actions/workflows/hax_renew.yml/dispatches`
     - 例：`https://api.github.com/repos/jyucoeng/auto_hax_public/actions/workflows/hax_renew.yml/dispatches`
   - **Method**：`POST`
   - **Body**（Content-Type 选 `application/json`）：`{"ref":"main"}`
   - **Headers**（添加 3 条）：
     - `Accept: application/vnd.github+json`
     - `Authorization: Bearer <你的GitHub PAT，需 workflow scope>`
     - `X-GitHub-Api-Version: 2022-11-28`
   - **Schedule**：
     - **Timezone**：`Asia/Shanghai`
     - 时间点填你配置里的 `HUS_CRON_TIMES` 各**提前 5 分钟**（因为 runner 装依赖要几分钟，需提前拉起才能准点就绪）。例如配置是 `01:00,03:00,04:00,05:00,08:00,18:00,19:00,20:29`，则任务时间设为 `00:55,02:55,03:55,04:55,07:55,17:55,18:55,20:24`。
3. 保存并启用。到点会自动触发，无需人工干预。

---

## 四、重要说明

- **配置优先级**：工作流 `env` 区 > 私有库 `hus_config.txt`。你改自己公开库 yml 里的 `env` 即可调整抢机点 / 区域 / 系统，**不要去改私有库文件**。
- **安全性**：Secrets 仅存在于你自己的公开库，经 `${{ secrets.* }}` 加密注入，他人（含作者）看不到值；也不要把任何 PAT / session 写进代码或 Issues。
- **私有库 `hus_config.txt` 已清空真实凭据**，随只读分享不会泄露。
- **时区无关**：脚本内部统一按**北京时间（UTC+8）**判断抢机点，GitHub runner 的时区不影响触发。
- **时长**：每次运行约 17 分钟（覆盖预登录 + 抢机窗口），由工作流自动结束；GitHub Actions 免费额度通常够用。

---

## 五、排错

- **创建没发生 / 失败**：下载 Artifacts 里的 `hax-log-*` 和 `hax-screenshots-*` 看详情，多半是登录验证码、账号被限或代理问题。
- **检出私有库 401/403**：检查 `PRIVATE_REPO_TOKEN` 是否对 `jyucoeng/auto_hax` 有读权限、是否过期。
- **dispatch 403**：触发用的 GitHub PAT 需 **`workflow` scope**（classic）或 fine-grained **Actions: Read and write**；公开库也要对该 PAT 所属账号可见/可写。
- **代理无效**：确认 `HAX_HY2_PROXY_URL` 内容格式正确，且 `SOCKS_PORT` 与代理一致。

---

## 六、自己改抢机时间 / 区域（可选）

编辑你公开库里的 `hax_renew.yml` → `env` 区：
- `HUS_CRON_TIMES`：你的北京时间点（逗号分隔）
- `VPS_REGIONS_WANT`：想抢的区域白名单（如 `US-OpenVZ-3,US-OpenVZ-2,EU-1`）
- `HUS_OS`：默认系统（如 `Ubuntu 24`）

改完推到 `main` 即生效；cron-job.org 任务时间记得同步提前 5 分钟。
