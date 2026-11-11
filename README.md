# info-flow-builder

一个可直接给 AI 使用的 Skill：帮任何人从零搭建一套「私人信息流」系统。

## 它做什么

搭建完成后你会拥有三个东西：

1. **收集表单**——手机上点开，粘贴链接 + 写一句感悟，提交
2. **AI 自动整理**——每 2 小时抓取正文、生成摘要和标签、写一句价值点评
3. **信息流网站**——像刷朋友圈一样浏览自己筛选过的内容，手机可访问

所有资源建在**你自己的个人飞书账号**下，不依赖任何企业租户。

```
收集表单（飞书）→ AI 自动化（定时整理）→ 静态信息流网站（EdgeOne 托管）
```

## 安装

把下面这句话连仓库地址一起发给你的 AI Agent：

```
请从这个 GitHub 仓库安装 skill：
https://github.com/dancy3921-hui/info-flow-builder
clone 到 ~/.workbuddy/skills/info-flow-builder/，
装好后告诉我「帮我搭一个自己的信息流」怎么用。
```

Agent 手动操作的话就是两条命令：

```bash
git clone https://github.com/dancy3921-hui/info-flow-builder.git ~/.workbuddy/skills/info-flow-builder
```

Windows 下 `~` 即 `C:\Users\<你的用户名>`。

## 使用

装好后对 AI 说：

> 帮我搭一个自己的信息流

AI 会读取 `SKILL.md` 按 7 步执行，全程约 10-20 分钟，会向你索取飞书授权（扫一次码）。

## 前置条件

- 一个飞书账号（个人版即可，无需企业认证）
- 能对 AI 说上话，AI 有命令行权限

## 内容结构

```
SKILL.md                          主流程文档，7 步完整指南
references/site-template/
  feed.html                       信息流阅读页模板
  collect.html                    收集页模板（备用，主用飞书表单）
  data.json                       数据文件结构模板
```

## 这套流程踩过的坑（都已写进 SKILL.md）

- EdgeOne 云函数版需要 workspace 权限，纯静态部署不需要
- CloudStudio 的分享链接在手机浏览器打不开，用 EdgeOne
- EdgeOne 访问链接必须带 `?eo_token=...`，去掉就打不开
- 飞书 select 类型字段的值必须传数组，不能传字符串
- 个人飞书账号可以创建自建应用（2023 年已放开，免审发布）
- CLI 建字段时 url 类型会降级为 text，链接值传纯字符串

## License

MIT
