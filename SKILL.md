---
name: ai-daily-digest
description: "从 90 个顶级技术博客 RSS 获取最新文章 → 编号展示（中文标题+摘要）→ 用户选号 → 抓取全文 → 高记忆度总结。触发命令：/digest。"
---

# AI Daily Digest v2

极简交互流程：RSS 拉列表 → 用户选文章 → 高记忆度总结。

## 前置依赖

- `digest.ts` 脚本（本 skill 自带，负责 RSS 抓取 + AI 评分 + 中文标题翻译）
- `baoyu-url-to-markdown`（抓取文章全文）
- `high-memory-summary-skill`（总结规则）
- `bun` 运行时（通过 `npx -y bun` 自动安装）

## 流程

### 第一步：获取文章列表

运行：
```bash
npx -y bun ${SKILL_DIR}/scripts/digest.ts --hours 48 --top-n 30 --lang zh --json --output /tmp/digest-articles.json
```

参数说明：
- `--hours 48`：抓取最近 48 小时的文章（可选 24/48/72/168）
- `--top-n 30`：保留 30 篇（给用户更多选择）
- `--json`：输出 JSON 格式

JSON 中每篇文章包含：
- `titleZh`：中文标题
- `title`：原始英文标题
- `description`：RSS 摘要（通常 1-3 句话）
- `link`：文章链接
- `sourceName`：博客名称
- `pubDate`：发布时间

### 第二步：编号展示

读取 JSON，按编号输出给用户：

```
📋 技术博客精选（共 N 篇，来自 90 个 RSS 源）

1. 中文标题 — 摘要前 80 字 [来源博客]
2. 中文标题 — 摘要前 80 字 [来源博客]
3. 中文标题 — 摘要前 80 字 [来源博客]
...
```

- 标题用 `titleZh`，摘要从 `description` 截取
- 每条附带来源博客名
- 不要自动选中任何条目，等用户指定

### 第三步：用户指定编号 → 串行抓取 + 并行总结

用户会给出编号，格式可能是："1、3、5" 或 "2-6" 或 "7"。

**抓取阶段（串行）：**

对每篇选中的文章，使用 `baoyu-url-to-markdown` 抓取全文：
```bash
npx -y bun ~/.agents/skills/baoyu-url-to-markdown/scripts/main.ts "<article_url>"
```
抓取完成后读取输出的 markdown 文件。

如果抓取失败（付费墙、403 等），回退到 RSS 中的 `description`，并标注"仅基于摘要总结"。

**总结阶段（并行）：**

全部抓取完成后，使用 `delegate_task` 并行总结：
- 将文章按批次分配给子 agent（每批 1-3 篇）
- 每个子 agent 拿到文章全文（或摘要），按 `high-memory-summary-skill` 规则输出总结
- 主 agent 收集所有总结，按编号顺序合并输出

**并行批次建议：**
- 3-5 篇：每篇一个子 agent
- 6-10 篇：每 2 篇一个子 agent
- 10+ 篇：每 3 篇一个子 agent
- 同时最多 3-4 个子 agent 并行

### 第四步：高记忆度总结输出

对每篇文章，严格按 `high-memory-summary-skill` 的规则输出：

核心原则：
- 找到单条主线（governing thread）
- 只保留 3-5 个最值得记住的支持点
- 每个支持点带一个记忆锚点（数据、金句、反转、场景）
- 砍掉信息性细节，保留记忆性细节
- 语气像已经消化过内容的人在转述，不像笔记机器

输出结构：

```
## [编号]. 中文标题

🔗 原文链接
📝 来源博客
📅 发布时间（如有）

### 值得记住的部分

（用自然段落写出主线 + 3-5 个支持点，每点带记忆锚点）

### 多想一层（仅在内容有观点/判断/预测时才加）

（简短一段，指出最容易误解的地方，或条件限制）
```

### 多篇处理

- 每篇独立一个 `##` 小节
- 各篇之间不合并、不交叉
- 按用户指定的编号顺序输出
- 如果某篇抓取失败但有 RSS 摘要，基于摘要做简短总结并标注
- 如果某篇完全无内容，明确标注"无法总结"并跳过

## 输出保存

全部总结完成后，同时保存一份到：
```
~/outputs/ai-daily-digest-skill/YYYY-MM-DD-技术博客精选总结.md
```

## 注意事项

- RSS 抓取走公网，无需浏览器
- 全文抓取（baoyu-url-to-markdown）需要网络访问，部分站点可能有付费墙
- 抓取阶段串行执行，避免并发请求被限流
- 总结阶段可并行处理（纯 LLM 工作，无需网络）
- 英文内容由子 agent 翻译为简体中文后再总结
- 如果 RSS 中没有 description，只显示中文标题和来源

## 旧版兼容

如需使用旧版 AI 评分+HTML 报告模式，参考原 SKILL.md 中的 `Agent 深度解读` 或 `Lite 极速模式` 流程。本版本（v2）为默认入口。
