# channels-video-publish · 视频号自动发布

用原生 CDP 驱动已登录的 Chrome，把本地视频自动上传到**微信视频号助手**，
填好短标题 / 描述 / 合集，声明原创，设定时发表（或即时发表）。

支持 WorkBuddy / QClaw / ima / Claude Code / Cursor。

## 为什么需要它

新版「视频号助手」改成了 SPA hub，**真正的发布表单藏在一个宽高为 0×0 的 iframe 里**：

- 鼠标坐标点击全部失效 → 只能用合成 DOM 事件
- 表单 iframe 的 URL 是 `empty.html` → 靠 URL 匹配定位的老脚本全废
- file chooser 在 0×0 iframe 里根本不触发 → 必须走 `objectId` + `DOM.setFileInputFiles`

这些坑每一个都能耗掉一小时。本 skill 把它们全部固化在 `references/troubleshooting.md`。

## 安装

把整个目录放到 skills 目录下即可：

```
~/.workbuddy/skills/channels-video-publish/     # WorkBuddy
~/.claude/skills/channels-video-publish/        # Claude Code
```

依赖：`ws`（`npm i ws`）。

## 前置条件

1. 带远程调试端口启动 Chrome（用独立 profile，别污染常用浏览器）：

```bash
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --remote-debugging-port=9222 --user-data-dir=/Users/jx/ChromeAutomation &
```

在该浏览器里**手动扫码登录**视频号助手。会话会过期，脚本会以退出码 3 提示。

2. 跑之前清代理（否则 9222 的流量会被劫持到 80）：

```bash
unset http_proxy https_proxy all_proxy HTTP_PROXY HTTPS_PROXY ALL_PROXY NODE_OPTIONS
export NO_PROXY='127.0.0.1,localhost'
```

## 用法

```bash
# 定时发表（10-15 07:00）
node scripts/cdp-publish.js --file video.mp4 --title "短标题" \
  --desc "描述" --album "合集名" --duration 516 --day 15 --hour 07

# 即时发表（绕过日历面板）
node scripts/cdp-publish.js --file video.mp4 --title "短标题" --immediate

# 演练：上传+填字段+存草稿，但不点发表
node scripts/cdp-publish.js --file video.mp4 --title "短标题" --dry-run

# 批量排期（episodes.json）
node scripts/cdp-publish.js --episodes episodes.json --ep 24 --day 15 --hour 07

# 核查哪些已发布内容没声明原创
node scripts/check-original.js
```

### 退出码

| 码 | 含义 |
|---|---|
| 0 | 成功 |
| 1 | 失败 |
| 3 | 登录态失效，需重新扫码 |
| 4 | 定时日期超出 14 天窗口（**未发布**） |

## 安全设计

- 视频号定时**最远只能选未来 14 天**。脚本会提前校验，超窗直接中止，绝不误发
- 日期 / 时间回填会二次校验，不符预期就停
- 原创声明必须真实点击，不允许只改 DOM 属性
- 本 skill **不含任何删除动作** —— 删除已发布内容不可逆且会丢播放/赞数据

## 已知限制

- ✅ 已验证：表单定位、文件上传、短标题、描述、合集
- ⚠️ 未验证：定时日历面板展开、原创弹窗闭环、最终发表的弹窗链
  （日历面板在 0×0 iframe 里不吃合成 click；用 `--immediate` 可绕过）

详见 `references/troubleshooting.md`。
