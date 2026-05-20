---
name: ai-daily
description: 自动爬取当天AI新闻并精炼总结，提供原文地址
---

# AI Daily News

自动爬取当天AI领域的新闻，精炼总结并提供原文地址。

## Important Notes

- **WebFetch 不可用**：必须使用 `curl` 命令行工具抓取，不要用 WebFetch
- **SPA 网站无法抓取**：36氪、虎嗅、新浪、搜狐、雷锋网、新智元、极客公园等都是 JavaScript 渲染的 SPA，curl 拿不到新闻内容，**不要尝试抓取这些源**
- **只抓取 SSR 渲染的网站**：以下源经过验证可以用 curl 抓取到实际内容
- **RSS 是可靠的备用方案**：机器之心、量子位等站点提供 RSS/Feed，可以用 curl 获取结构化数据

## Instructions

### Step 1: Fetch news from verified SSR sources

```bash
export TEMP_DIR="$(pwd)/ai_daily_$(date +%Y%m%d)"
mkdir -p "$TEMP_DIR"

# 量子位（SSR渲染，可直接抓取）
curl -s -m 15 "https://www.qbitai.com/" \
  -H "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36" \
  -o "$TEMP_DIR/qbitai.html" &

# 量子位 RSS（备用）
curl -s -m 15 "https://www.qbitai.com/feed" \
  -H "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36" \
  -o "$TEMP_DIR/qbitai_feed.xml" &

# 机器之心（SSR渲染）
curl -s -m 15 "https://www.jiqizhixin.com/" \
  -H "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36" \
  -o "$TEMP_DIR/jiqizhixin.html" &

# 机器之心 RSS（备用）
curl -s -m 15 "https://www.jiqizhixin.com/rss" \
  -H "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36" \
  -o "$TEMP_DIR/jiqizhixin_feed.xml" &

# 36氪 AI 频道 RSS
curl -s -m 15 "https://36kr.com/information/AI/feed" \
  -H "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36" \
  -o "$TEMP_DIR/36kr_ai_feed.xml" &

# TechCrunch AI/Tech RSS
curl -s -m 15 "https://techcrunch.com/feed/" \
  -H "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36" \
  -o "$TEMP_DIR/techcrunch_feed.xml" &

# Ars Technica Tech RSS
curl -s -m 15 "https://arstechnica.com/feed/" \
  -H "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36" \
  -o "$TEMP_DIR/arstechnica_feed.xml" &

# AI News RSS
curl -s -m 15 "https://www.artificialintelligence-news.com/feed/" \
  -H "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36" \
  -o "$TEMP_DIR/ai_news_feed.xml" &

# MIT Technology Review RSS
curl -s -m 15 "https://www.technologyreview.com/feed/" \
  -H "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36" \
  -o "$TEMP_DIR/mit_tr_feed.xml" &

# VentureBeat AI RSS
curl -s -L -m 15 "https://venturebeat.com/feed/" \
  -H "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36" \
  -o "$TEMP_DIR/venturebeat_feed.xml" &

# Product Hunt RSS
curl -s -m 15 "https://www.producthunt.com/feed" \
  -H "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36" \
  -o "$TEMP_DIR/producthunt_feed.xml" &

wait
```

### Step 2: Parse and extract news

用 python 解析 HTML 提取新闻标题和链接：

```bash
python "$TEMP_DIR/parse.py" 2>&1 | tee "$TEMP_DIR/output.txt"
```

parse.py 内容（写入到临时文件后执行）：

```python
import os, re, sys, xml.etree.ElementTree as ET

TEMP_DIR = os.path.dirname(os.path.abspath(__file__))

all_items = []

# 解析 HTML 源
html_sources = {
    '量子位': os.path.join(TEMP_DIR, 'qbitai.html'),
    '机器之心': os.path.join(TEMP_DIR, 'jiqizhixin.html'),
}

for source, path in html_sources.items():
    if not os.path.exists(path):
        continue
    with open(path, 'r', encoding='utf-8') as f:
        content = f.read()

    # 提取 h2/h3/h4 中包含 a 标签的标题和链接
    pattern = r'<(h[234])[^>]*>(.*?)</\1>'
    matches = re.findall(pattern, content, re.IGNORECASE | re.DOTALL)

    for tag, text in matches:
        links = re.findall(r'href="([^">]+)"', text)
        titles = re.findall(r'>([^<]+)<', text)
        for link, title in zip(links, titles):
            title = title.strip()
            if len(title) > 6:
                if link.startswith('/'):
                    link = 'https:' + link
                all_items.append((title, link, source))

# 解析 RSS/Feed 源
feed_sources = {
    '量子位': os.path.join(TEMP_DIR, 'qbitai_feed.xml'),
    '机器之心': os.path.join(TEMP_DIR, 'jiqizhixin_feed.xml'),
    '36氪AI': os.path.join(TEMP_DIR, '36kr_ai_feed.xml'),
    'TechCrunch': os.path.join(TEMP_DIR, 'techcrunch_feed.xml'),
    'ArsTechnica': os.path.join(TEMP_DIR, 'arstechnica_feed.xml'),
    'AI News': os.path.join(TEMP_DIR, 'ai_news_feed.xml'),
    'MIT Tech Review': os.path.join(TEMP_DIR, 'mit_tr_feed.xml'),
    'VentureBeat': os.path.join(TEMP_DIR, 'venturebeat_feed.xml'),
    'Product Hunt': os.path.join(TEMP_DIR, 'producthunt_feed.xml'),
}

for source, path in feed_sources.items():
    if not os.path.exists(path):
        continue
    try:
        tree = ET.parse(path)
        root = tree.getroot()
        # RSS 2.0
        for item in root.iter('item'):
            title = item.findtext('title', '').strip()
            link = item.findtext('link', '').strip()
            if title and len(title) > 6:
                all_items.append((title, link, source))
        # Atom
        for item in root.findall('.//{http://www.w3.org/2005/Atom}entry'):
            title_el = item.find('{http://www.w3.org/2005/Atom}title')
            link_el = item.find('{http://www.w3.org/2005/Atom}link')
            title = title_el.text.strip() if title_el is not None else ''
            link = link_el.get('href', '').strip() if link_el is not None else ''
            if title and len(title) > 6:
                all_items.append((title, link, source))
    except Exception:
        pass

# 去重
seen = set()
unique_items = []
for title, link, source in all_items:
    if title not in seen:
        seen.add(title)
        unique_items.append((title, link, source))

for title, link, source in unique_items:
    print(f'{source}|||{title}|||{link}')
```

### Step 3: Select the most important news

从提取的新闻中筛选 5-10 条最重要的：

1. **大模型/LLM** 相关发布、评测、对比
2. **AI 产品** 新发布或重大更新
3. **行业大事件**：融资、收购、政策监管、高管变动
4. **学术研究**：顶会论文突破、重要开源项目
5. **自动驾驶/机器人** 等具身智能领域进展

### Step 4: Compile the report

Format the report as:

```markdown
# AI日报 - {YYYY年MM月DD日}

## 重要新闻

### 1. [新闻标题]
**摘要**: 精炼的新闻摘要（100-200字）
**来源**: 来源媒体/网站
**原文**: [原文链接](url)

### 2. [新闻标题]
**摘要**: ...
**来源**: ...
**原文**: [原文链接](url)

---

## 行业动态

- [简短的行业动态列表]

---

## 最新发布/产品

- [新发布的AI产品或工具]

---

*本报告由 AI Daily Skill 自动生成*
```

## Verified Working Sources

以下源经过验证可以用 curl 抓取到实际新闻内容（SSR 渲染或 RSS）：

以下源经过验证可以用 curl 抓取到实际新闻内容（SSR 渲染或 RSS）：

| 来源 | 类型 | 说明 |
|------|------|------|
| 量子位 | SSR + RSS | 中文 AI 媒体 |
| 机器之心 | SSR + RSS | 中文 AI 媒体 |
| 36氪AI | RSS | 中文 AI 频道 |
| TechCrunch | RSS | 英文科技/AI 新闻 |
| Ars Technica | RSS | 英文科技新闻 |
| AI News | RSS | 英文 AI 垂直媒体 |
| MIT Tech Review | RSS | 英文科技媒体 |
| VentureBeat | RSS | 英文 AI/科技媒体 |
| Product Hunt | Atom | 英文新产品发布 |

以下源 **不可用**（SPA 渲染，curl 无法获取新闻内容）：

| 来源 | 状态 | 原因 |
|------|------|------|
| 虎嗅 | ❌ 不可用 | Nuxt SPA，HTML 无内容 |
| 雷锋网 | ❌ 不可用 | SPA，HTML 无内容 |
| 新智元 | ❌ 不可用 | SPA，HTML 无内容 |
| 极客公园 | ❌ 不可用 | SPA，HTML 无内容 |
| Synced | ❌ 不可用 | SPA，重定向到 /lander |
| 新浪科技 | ❌ 不可用 | SPA |
| 搜狐科技 | ❌ 不可用 | SPA |
| TopHub | ❌ 不可用 | SPA |

## Guidelines

- 精选 5-10 条最重要的新闻
- 摘要要精炼，突出关键信息（谁、做了什么、为什么重要）
- 每条新闻必须提供可点击的原文链接
- 如果无法获取任何新闻，输出："今日无法获取AI新闻，请稍后重试。"

## Error Handling

- 如果某个 curl 请求超时或失败，立即跳过，尝试下一个
- 不要重试失败的请求
- 不要无限循环
- 每个源最多等待 15 秒（curl -m 15）
- 如果所有源都失败，报告失败并结束
