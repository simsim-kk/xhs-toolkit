# 灵感铺 · 小红书创作三件套

一站式小红书创作辅助工具：热点选题、智能排版、爆款率检测。

🔗 在线访问：[simsim-kk.github.io/xhs-toolkit](https://simsim-kk.github.io/xhs-toolkit/)

![status](https://img.shields.io/badge/status-live-brightgreen)
![stack](https://img.shields.io/badge/stack-GitHub%20Pages%20%2B%20Cloudflare%20Worker-blue)

---

## 这是什么

小红书创作者在发布一篇笔记前，通常要经历"找选题 → 写内容 → 调格式 → 判断效果"
这几步，过程分散在好几个工具和习惯里。灵感铺把这几步整合成一个网页，三个模块：

| 模块 | 功能 | 技术实现 |
|---|---|---|
| 🔥 热点追踪 | 按 10 个常见内容方向，给出当下适合切入的选题角度和话题标签 | Cloudflare Worker 定时任务 + KV 缓存 |
| 🎨 排版生成 | 把任意文字一键转换成小红书风格（标题/分段/emoji/话题标签） | 实时调用 Groq API |
| 📊 爆款检测 | 从标题吸引力、内容结构、话题契合、互动引导四个维度评分 | 实时调用 Groq API |

## 技术架构

```
访客浏览器
   │
   ├─ 静态页面（HTML/CSS/JS 单文件）─── 托管在 GitHub Pages
   │
   └─ fetch 请求 ──────────────────→ Cloudflare Worker（边缘函数）
                                          │
                                          ├─ GET /api/trends   → 读 KV 缓存
                                          ├─ POST /api/format  → 实时调 Groq
                                          └─ POST /api/score   → 实时调 Groq
                                          │
                                  Cron Trigger（每 4 小时）
                                          │
                                          ▼
                                  批量生成 10 个方向的热点
                                  写入 Workers KV
```

**为什么这么设计**

- **前后端分离**：前端是纯静态文件，部署在免费的 GitHub Pages；后端逻辑（API 调用、
  密钥管理、缓存）全部放在 Cloudflare Worker，密钥不会暴露在前端代码里。
- **热点用缓存而非实时调用**：热点选题不需要逐秒更新，定时任务每隔几小时批量生成
  一次存进 KV，用户访问时直接读缓存，响应速度从"等 AI 生成"变成"毫秒级返回"，
  也避免了高并发访问时重复消耗 API 额度。
- **代理调用 + 限流**：排版和检测功能允许访客直接使用而不需要自己的 API key，
  由 Worker 用我自己的 Groq key 代理转发；为防止被恶意刷量，加了基于访客 IP 的
  限流（每小时 20 次）和 CORS 白名单（只允许本项目域名调用）。

## 踩过的坑

最初尝试直接在 Worker 里抓取微博/抖音/知乎等公开热搜聚合 API，结果发现这类免费
接口普遍会拦截 Cloudflare 的数据中心出口 IP（返回 403/530），这是平台的反爬虫
机制，换了好几个数据源都遇到同样问题。后来改为用 AI 模型自身知识生成热点选题
建议，牺牲了"逐分钟更新的精确热搜排行榜"，换来了零依赖、零被拦截风险的稳定方案，
更适合"选题灵感激发"这个实际场景。

## 技术栈

- 前端：原生 HTML / CSS / JavaScript（无框架，单文件部署）
- 部署：GitHub Pages（前端）+ Cloudflare Workers（后端）
- 存储：Cloudflare Workers KV（热点数据缓存）
- AI：Groq API（llama-3.3-70b-versatile）
- 定时任务：Cloudflare Cron Triggers

## 本地开发 / 自行部署

如果想 fork 这个项目部署自己的版本：

1. **部署 Worker**：参考 [`xhs-worker/README.md`](./xhs-worker/README.md)，
   需要自己申请 Groq API key
2. **部署前端**：把 `index.html` 上传到任意支持静态托管的平台（GitHub Pages /
   Vercel / Netlify 均可），记得把代码里的 `WORKER_BASE` 改成你自己的 Worker 地址
3. **收紧跨域权限**：把 Worker 代码里的 `ALLOWED_ORIGIN` 改成你自己的前端域名

## 作者

[@simsim-kk](https://github.com/simsim-kk)
