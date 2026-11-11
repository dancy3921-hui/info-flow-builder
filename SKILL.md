---
name: info-flow-builder
description: 搭建「私人信息流」系统：飞书多维表格当选题库后端 + 收集表单 + AI 自动整理 + 可刷读的静态信息流网站。适用于用户想复刻松月式信息流（收集要轻、AI 整理、沉浸刷读）的需求。触发词：信息流、选题库、收集工具、松月、私人信息流。
---

# 私人信息流搭建指南

帮用户从零搭建一套完整的私人信息流系统。整体链路：

**收集表单（飞书）→ AI 自动化（定时整理）→ 静态信息流网站（EdgeOne 托管）**

所有资源建在用户的**个人飞书账号**下，不依赖任何企业租户。

## 第一步：确认飞书 CLI 可用

```bash
lark-cli auth status --json
```

- 若已登录且是用户想要的账号，直接进入第二步
- 若未登录或需要换账号：

```bash
# 创建/绑定应用（个人飞书账号也可创建自建应用，免审发布）
lark-cli config init --new --force-init
# Agent 环境必须加 --force-init；生成 open.feishu.cn/page/cli?user_code=... 链接给用户，用其飞书账号确认

# 用户确认后，发起授权（base 权限域）
LARKSUITE_CLI_NO_UPDATE_NOTIFIER=1 lark-cli auth login --domain base --no-wait --json
# 把返回的 verification_url 给用户扫码，确认后用 auth login --device-code <device_code> 完成登录
```

坑：device code 有效期约 10 分钟，过期需重新 `--no-wait` 取新链接；`auth login --device-code` 过期时静默退出无输出。

## 第二步：创建选题库多维表格

```bash
lark-cli base +base-create --name "<用户的>信息流选题库" --table-name "信息流" --fields '[
  {"name":"链接","type":"text"},
  {"name":"类型","type":"select","multiple":false,"options":[{"name":"公众号文章"},{"name":"小红书笔记"},{"name":"即刻动态"},{"name":"知识星球"},{"name":"网页"},{"name":"视频"}]},
  {"name":"感悟","type":"text"},
  {"name":"标题","type":"text"},
  {"name":"摘要","type":"text"},
  {"name":"标签","type":"select","multiple":true,"options":[{"name":"信息管理"},{"name":"AI工作流"},{"name":"选题库"},{"name":"案例"},{"name":"工具"},{"name":"方法论"},{"name":"文案"},{"name":"运营"},{"name":"产品"},{"name":"其他"}]},
  {"name":"AI备注","type":"text"},
  {"name":"状态","type":"select","multiple":false,"default_value":["待整理"],"options":[{"name":"待整理"},{"name":"已整理"}]},
  {"name":"收集时间","type":"datetime"}
]' --as user
```

记录返回的 `base_token` 和 URL。再用 `+table-list` 拿 `table_id`。

注意：标签字段必须在创建时就带 options，否则写记录会失败。

## 第三步：创建收集表单

```bash
# 创建表单
lark-cli base +form-create --base-token <base_token> --table-id <table_id> --name "随手收集" --description "看到好东西就丢进来，顺手写一句感悟" --as user
# 查字段列表拿 field_id，设置表单只显示：链接 / 类型 / 感悟 三个题
lark-cli base +view-set-visible-fields --base-token <base_token> --table-id <table_id> --view-id <form_id> --json '{"visible_fields":["<链接field_id>","<类型field_id>","<感悟field_id>"]}' --as user
# 开启分享
lark-cli base +form-share-update --base-token <base_token> --table-id <table_id> --form-id <form_id> --enabled=true --as user
# 获取分享链接
lark-cli base +form-share-get --base-token <base_token> --table-id <table_id> --form-id <form_id> --as user
```

设计原则（来自松月）：收集动作要极轻——只留 链接/类型/感悟 三项，**感悟是灵魂**，收藏那一刻的价值判断不记下来一个月后就忘了。

## 第四步：搭信息流网站（纯静态）

模板文件在 `references/site-template/`：

- `feed.html`（信息流刷读页，同时也是 index 首页）：卡片式布局，显示标题/AI摘要/感悟（橙色标注块）/标签/类型筛选，读同目录 `data.json`
- `collect.html`：备用收集页（链接+类型+感悟，提交接口版）
- `data.json`：数据文件，格式 `{"updated":"YYYY-MM-DD","items":[{"id","url","type","note","ts","status","title","summary","tags","source"}]}`，按收集时间倒序

操作：
1. 复制三个文件到部署目录，把 `feed.html` 里的收集表单链接占位符替换为第三步拿到的真实链接
2. `feed.html` 同时复制一份命名为 `index.html`（作为网站首页）

## 第五步：部署到 EdgeOne Makers（纯静态，免费）

```bash
cd <部署目录>
PAGES_SOURCE=skills edgeone makers deploy -n <项目名> --json
```

坑（重要）：
- **必须纯静态**：项目里不能带 cloud functions / @edgeone/pages-blob 依赖，否则报 "Please upgrade your EdgeOne Makers to use Blob"
- **产物 URL 必须完整保留**：`https://<子域名>.edgeone.cool?eo_token=...&eo_time=...`，去掉 token 参数直接 401。给用户的链接不能截断
- **令牌跨部署仍有效**：重新部署后旧 token 依然 200，用户书签不会失效
- 连续两次部署可能报 Windows DeleteDirectory 异常（临时目录占用），等 3 秒重试即可
- 不要用 CloudStudio 托管给终端用户：其 shareLink 缺访问参数，手机浏览器打开报 403「链接不完整」

## 第六步：创建 AI 整理自动化

用 automation_update 创建（scheduleType=recurring，rrule=`FREQ=HOURLY;INTERVAL=2`），prompt 模板：

```
读取飞书选题库（lark-cli base +record-list --base-token <base_token> --table-id <table_id> --as user），
筛选「状态」≠「已整理」的记录。无待整理记录则结束。
对每条：WebFetch 抓取链接正文（失败则基于 URL 和感悟概括），生成：标题（原文标题）、
摘要（80字内）、标签（从预设选项选3-5个）、AI备注（一句话点评对内容创作的利用价值）。
用 record-batch-update 写回（select 字段值必须传数组，如 ["已整理"]），收集时间为空则填当前时间。
全部完成后读取所有已整理记录，生成 data.json 写入部署目录（含 ts 毫秒时间戳，倒序），
再用 edgeone makers deploy 重新部署，把完整带 token 的新链接发给用户。
```

注意：select 字段写值必须传数组（如 `["已整理"]`），传字符串会失败。

## 第七步：交付与使用指导

给用户交代清楚：
1. 收集表单链接 → 手机「添加到主屏幕」，命名「收集」
2. 信息流网站完整链接（含 token）→ 同样加到主屏幕，命名「信息流」
3. 日常使用：刷到好内容 → 复制链接 → 打开收集 → 粘贴+写一句感悟 → 提交；最长 2 小时后信息流里可刷到
4. 选题时打开飞书表格按标签筛选

## 可选升级

- 标签选项按用户领域定制
- 频率调整（INTERVAL=1 更快 / 4 更省）
- 若用户 EdgeOne 账号开通了 workspace，可升级为云函数实时写入版（收集页直接提交入库，不再依赖表单）
