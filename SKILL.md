---
name: channels-video-publish
slug: channels-video-publish
displayName: 视频号自动发布
version: 1.0.0
description: 自动上传并定时发布视频到微信视频号（视频号助手网页版）。用原生 CDP 驱动已登录的 Chrome，完成上传视频、填短标题/描述/合集、声明原创、设定时发表（或即时发表）。当用户说「发视频号」「视频号发布」「上传视频号」「视频号定时发表」「补声明原创」「视频号排期」时使用。也用于核查已发布条目的原创声明状态。
agent_created: true
---

# 视频号自动发布

## Overview

用原生 CDP（不走 playwright）驱动一台带远程调试端口、已登录视频号助手的 Chrome，
把本地 mp4 上传到 `post/create`，填好短标题/描述/合集，勾原创，设定时后点发表。

核心价值：**避开新版「视频号助手」SPA 的 0×0 iframe 陷阱**。
发布表单藏在一个宽高为 0 的 iframe 里，鼠标坐标全失效，只能走合成 DOM 事件——
这些坑全在 `references/troubleshooting.md`，**改脚本前先读它**。

## When to use

- 「把这条视频发到视频号」「视频号定时发 10-15 07:00」
- 批量排期（配合 episodes.json 一条一条发）
- 「核查视频号哪些内容没声明原创」

## 前置条件

1. Chrome 带远程调试端口、用独立 profile 启动（别污染常用浏览器）：
   ```bash
   /Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
     --remote-debugging-port=9222 --user-data-dir=/Users/jx/ChromeAutomation &
   ```
   在该浏览器里**手动扫码登录**视频号助手。会话会过期，脚本退出码 3 会提示。
2. node 依赖 `ws`：`/Users/jx/.workbuddy/binaries/node/workspace/node_modules`
3. 跑之前必须清代理（否则 9222 流量被劫持到 80）：
   ```bash
   unset http_proxy https_proxy all_proxy HTTP_PROXY HTTPS_PROXY ALL_PROXY NODE_OPTIONS
   export NO_PROXY='127.0.0.1,localhost'
   ```
   Bash 工具里需要 `dangerouslyDisableSandbox: true`。

## Workflow

### Step 1 — 确认输入

- 视频文件（mp4）、**短标题 ≤16 字**、描述、合集名（默认「怎么活得更好」）
- 目标日期：**必须 ≤ 今天+14 天**（视频号硬限制，脚本会自动校验并中止）

### Step 2 — 先用 dry-run 跑一遍

强烈建议第一次或改过脚本后先：

```bash
node scripts/cdp-publish.js --file /path/a.mp4 --title "标题" \
  --desc "描述" --album "怎么活得更好" --duration 516 --day 15 --hour 07 --dry-run
```

dry-run 会做完上传+填字段+保存草稿，**不点发表**。确认草稿箱里内容无误再正式跑。

### Step 3 — 正式发布

- 定时：去掉 `--dry-run`，带 `--day/--hour`（可加 `--month`）
- 即时发表：加 `--immediate`（不定时，绕过日历面板）
- 批量：维护 episodes.json，用 `--episodes <文件> --ep <编号>`

### Step 4 — 核对

- 到草稿箱/发表记录核对日期时间与原创标记
- 核查原创：`node scripts/check-original.js`

## CLI

```
node scripts/cdp-publish.js --file <mp4> --title <短标题> [--desc <描述>]
       [--album <合集>] [--duration <秒>] [--day <日> --hour <时> [--minute <分>]]
       [--immediate] [--dry-run]
node scripts/cdp-publish.js --episodes episodes.json --ep 24 --day 15 --hour 07
node scripts/check-original.js [--all]
```

退出码：`0` 成功｜`1` 失败｜`3` 登录态失效需重扫｜`4` 定时日期超窗口（**未发布**）

## 安全规则（不要跳过）

- 目标日超 14 天窗口 → **中止，不发布**（宁可少发不错发）
- 日期/时间回填校验不符 → 中止
- **原创声明必须真实点击**，不要只改 DOM 属性
- 删除已发布条目不可逆且丢播放/赞数据，**必须用户明确确认**（本 skill 不含删除动作）
- 已发布条目**不能补声明原创**；>6 个月的连修改都不行，只能删旧重发

## 已知未打通

- ✅ 已验证：表单定位、文件上传、短标题、描述、合集
- ⚠️ 未验证：定时日历面板展开、原创弹窗闭环、最终发表的弹窗链
  （详见 `references/troubleshooting.md` 第 12 条）

## 文件

- `scripts/cdp-publish.js` — 主发布脚本
- `scripts/check-original.js` — 原创声明状态核查
- `references/troubleshooting.md` — **踩坑清单，改代码前必读**
- `assets/episodes.example.json` — 批量配置模板
