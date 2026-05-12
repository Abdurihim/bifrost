# Bifrost AI Research Radar 技术方案

## 功能模块说明

Bifrost AI Research Radar 是面向个人长期运行的 AI 技术情报采集、聚合、去重、趋势发现和学习辅助系统。它不是通用搜索引擎，也不以一次性问答为核心目标；它的核心价值是每天持续发现 AI 技术圈值得关注的项目、论文、观点、讨论和趋势，并沉淀成 Bifrost Agent 可查询、可解释、可复盘的本地知识库。

本方案只落技术设计，不实现代码。实施前必须经过详细评审，确认 V1 范围、外部 API 配额、数据保留策略、模型成本和安全边界。

## 目标与非目标

### 必须实现的长期目标

1. 采集：从 GitHub、Hacker News、RSS、论文源、Reddit、YouTube 等来源采集 AI 技术信号。
2. 清洗：将 HTML、RSS 条目、API JSON、README、论文摘要等清洗成可索引 Markdown 和纯文本。
3. 理解：生成摘要、技术价值、适合人群、和已有知识的关系、推荐动作。
4. 去重聚合：将同一事件在多个源上的传播合并为一个 Intelligence Event。
5. 趋势发现：发现项目、概念、人物、技术路线在 24h / 7d / 30d 窗口内的异常升温。
6. 学习输出：生成日报、周报、专题学习路线、代码实践建议和阅读队列。

### V1 明确范围

V1 必须覆盖关键数据来源，但采用分级接入、失败隔离和配置显式化来保证可落地。范围不是“只做三源”，而是“所有关键源都进入 V1 架构和验证闭环，按稳定程度选择不同采集策略”。

V1 必做来源：

- GitHub：项目发现、release、star velocity、issue/discussion 热点。
- Hacker News：高信号技术讨论。
- RSS：公司博客、个人博客、RSSHub 路由和站点 feed。
- 论文源：arXiv、OpenAlex、Semantic Scholar 的论文发现、摘要、引用/相关工作 metadata。
- Reddit：社区实践反馈、争议点、工具体验和失败案例。
- YouTube：channel RSS、视频 metadata、可选 transcript。
- X/Twitter：AI 圈首发观点、项目发布、人物关注方向；优先官方/第三方 API 或 RSSHub/Nitter fallback，必须支持来源可用性降级。
- 动态网页：对没有稳定 API/RSS 的高价值页面，通过 Crawl4AI / Spider / Playwright sidecar 采集 Markdown，必须隔离资源、超时和站点策略风险。

V1 必做能力：

- 存储：SQLite + FTS5；embedding/vector 进入 V1 可选能力，默认关闭但 schema、配置和降级路径必须完整。
- 输出：`radar init`、`radar run`、`radar digest`、`radar search`、`radar trends`，MCP Server 也进入 V1 交付范围。
- 去重：URL/canonical URL/content hash/title hash 必做，SimHash 必做，embedding 聚合作为可选增强但接口必须稳定。
- 模型：只用于摘要、标签、学习建议，不参与基础事实采集和硬去重判定。
- 最小管理面：CLI 必做；WebUI 不做复杂产品页，但需要最小状态/配置可视化设计，后续实现可拆 PR。

V1 保留边界：

- 不做无边界“全网采集”；只采集配置的 sources、watch topics 和 allowlist domain。
- 不自动发布到 IM 或外部平台；只生成本地 digest，IM/外部推送必须由用户显式开启。
- 不在未配置凭据时偷用外部 API；需要 API key 的 source 必须明确报 `source.auth_required` 或走公开 fallback。
- 动态爬虫不得跑在 Agent core 内，必须作为 Radar source adapter 或 sidecar 受 timeout、并发、robots/站点条款检查和 kill 语义约束。

## 当前 Bifrost 对齐点

本方案复用现有 Bifrost 能力，不另起炉灶：

- `crates/agent` 已有 `ToolHandler`、`ToolRegistry`、内置工具、MCP manager、turn-scoped `tool_search`。
- MCP manager 已支持 stdio 和 Streamable HTTP transport，可启动 server、初始化、列工具、调用工具和关闭 stdio 子进程。
- MCP deferred loading 已存在：工具数量达到阈值时通过 `tool_search` 发现并加载。
- `Cargo.toml` workspace 已包含 `crates/agent`、`crates/bifrost-storage`，并在 workspace 依赖中使用 `rusqlite`。
- `crates/agent` 已包含长期记忆、文件工具、PTY/shell、Goal/update_plan 等适合消费 Radar 输出的能力。

结论：Radar Core 应独立成 crate 生产知识，Agent 和 MCP 只消费知识。不要把采集、清洗、embedding、趋势检测直接塞进 `crates/agent`。

## 总体架构

```text
Bifrost Agent
  radar_digest / radar_search / radar_trends / radar_learn
        |
        v
Bifrost Radar MCP Server 或内置工具适配层
        |
        v
AI Research Radar Core
  Scheduler / Pipeline / Store / Intelligence
        |
        v
Source Adapters
  GitHub / Hacker News / RSS / Papers / Reddit / YouTube / X / Dynamic Web
        |
        v
Processing Pipeline
  Fetch -> Clean -> Dedup -> Score -> Summarize -> Embed -> Cluster
        |
        v
Local Knowledge DB
  SQLite + FTS5 + optional vector backend
```

### 分层职责

| 层 | 职责 | 不负责 |
| --- | --- | --- |
| Agent 工具层 | 接收用户自然语言意图，调用 Radar 查询/报告/学习工具，组织最终回答 | 不直接抓网页，不直接写 Radar DB schema |
| MCP Server 层 | 将 Radar 能力暴露给 Bifrost Agent 和其他 Agent 运行时 | 不持有 Agent session 状态 |
| Radar Core | 调度采集、清洗、去重、评分、聚合、摘要、趋势计算 | 不管理 Bifrost 代理流量、不处理 IM 网关消息 |
| Source Adapter | 单一来源的鉴权、分页、rate limit、增量 checkpoint | 不做跨源聚合 |
| Store | SQLite schema、FTS、snapshot、查询、导入导出 | 不调用外部网络和模型 |
| Intelligence | scoring、dedup、clustering、trend、learning note | 不保存 secrets |

## 推荐落地形态

### 第一阶段：独立 crate + CLI

新增：

```text
crates/bifrost-radar/
  src/
    lib.rs
    config.rs
    scheduler.rs
    pipeline.rs
    types.rs
    store/
      mod.rs
      sqlite.rs
      migrations.rs
      fts.rs
    sources/
      mod.rs
      rss.rs
      github.rs
      hackernews.rs
    cleaner/
      mod.rs
      markdown.rs
      readability.rs
    intelligence/
      mod.rs
      dedup.rs
      scoring.rs
      trend.rs
      summarizer.rs
      learning.rs
crates/bifrost-cli/src/commands/radar.rs
```

CLI V1：

```bash
bifrost radar init
bifrost radar source list
bifrost radar source add rss --name "Simon Willison" --url "<feed-url>"
bifrost radar run --source rss --topic agent --max-items 100
bifrost radar digest --since 24h
bifrost radar search "browser agent" --since 30d --limit 20
```

第一阶段先不要求 Agent 内置工具或 MCP 工具可用，先证明本地知识库和 CLI 输出稳定。

### 第二阶段：MCP Server

新增：

```text
crates/bifrost-radar/src/mcp/server.rs
crates/bifrost-radar/src/mcp/tools.rs
```

Agent 配置示例：

```toml
[mcp_servers.ai_radar]
command = "bifrost"
args = ["radar", "serve-mcp", "--config", "~/.bifrost/radar/config.toml"]
enabled = true
startup_timeout_sec = 30
tool_timeout_sec = 600
```

MCP 工具：

- `radar_collect_now`
- `radar_digest`
- `radar_search`
- `radar_trends`
- `radar_explain`
- `radar_learn`
- `radar_watch`

优先 MCP 的原因：

- 与 Bifrost Agent 解耦，失败不会污染 Agent core。
- 可以被 Claude Code、Codex、Cursor 等其他 Agent 复用。
- 当前 Bifrost MCP manager 已能管理 stdio server 生命周期和 deferred tool discovery。
- 工具数量增加后可以自然进入 `tool_search`，不需要一次性暴露所有 source/tool。

### 第三阶段：Agent 内置工具

当 Radar Core 和 MCP 稳定后，再新增：

```text
crates/agent/src/tools/radar.rs
```

并在 `ToolRegistry::with_defaults(...)` 注册少量高价值内置工具：

- `radar_digest`
- `radar_search`
- `radar_trends`
- `radar_learn`

内置工具只作为高频消费入口，不替代 `bifrost-radar` crate。

## 数据模型

### Rust 类型草案

```rust
pub enum Platform {
    Github,
    HackerNews,
    Rss,
    Arxiv,
    OpenAlex,
    SemanticScholar,
    Reddit,
    Youtube,
    X,
}

pub struct RawItem {
    pub id: String,
    pub source_id: String,
    pub platform: Platform,
    pub url: String,
    pub canonical_url: Option<String>,
    pub title: String,
    pub author: Option<String>,
    pub content: Option<String>,
    pub published_at: Option<DateTime<Utc>>,
    pub fetched_at: DateTime<Utc>,
    pub raw_json: serde_json::Value,
}

pub struct CleanDocument {
    pub id: String,
    pub raw_item_id: String,
    pub title: String,
    pub markdown: String,
    pub text: String,
    pub lang: Option<String>,
    pub entities: Vec<Entity>,
    pub topics: Vec<String>,
    pub content_hash: String,
    pub simhash: Option<String>,
}

pub struct IntelligenceEvent {
    pub id: String,
    pub cluster_id: String,
    pub title: String,
    pub summary_short: String,
    pub summary_technical: String,
    pub why_it_matters: String,
    pub novelty_score: f32,
    pub importance_score: f32,
    pub learning_value_score: f32,
    pub confidence: f32,
    pub tags: Vec<String>,
    pub source_item_ids: Vec<String>,
}
```

### SQLite schema V1

Radar 使用独立数据库，不复用 traffic DB 或 memory DB：

```sql
CREATE TABLE radar_schema_meta (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL
);

CREATE TABLE radar_sources (
  id TEXT PRIMARY KEY,
  kind TEXT NOT NULL,
  name TEXT NOT NULL,
  url TEXT,
  config_json TEXT NOT NULL,
  enabled INTEGER NOT NULL DEFAULT 1,
  priority INTEGER NOT NULL DEFAULT 50,
  last_checked_at TEXT,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);

CREATE TABLE radar_source_checkpoints (
  source_id TEXT PRIMARY KEY,
  cursor_json TEXT NOT NULL,
  last_success_at TEXT,
  last_error TEXT,
  error_count INTEGER NOT NULL DEFAULT 0,
  updated_at TEXT NOT NULL
);

CREATE TABLE radar_raw_items (
  id TEXT PRIMARY KEY,
  source_id TEXT NOT NULL,
  platform TEXT NOT NULL,
  url TEXT NOT NULL,
  canonical_url TEXT,
  title TEXT NOT NULL,
  author TEXT,
  published_at TEXT,
  fetched_at TEXT NOT NULL,
  raw_json TEXT NOT NULL,
  status TEXT NOT NULL,
  UNIQUE(source_id, url)
);

CREATE TABLE radar_documents (
  id TEXT PRIMARY KEY,
  raw_item_id TEXT NOT NULL,
  title TEXT NOT NULL,
  markdown TEXT,
  text TEXT NOT NULL,
  content_hash TEXT NOT NULL,
  title_hash TEXT NOT NULL,
  simhash TEXT,
  lang TEXT,
  tokens INTEGER,
  created_at TEXT NOT NULL,
  UNIQUE(content_hash)
);

CREATE VIRTUAL TABLE radar_documents_fts
USING fts5(title, text, content='radar_documents', content_rowid='rowid');

CREATE TABLE radar_embeddings (
  document_id TEXT PRIMARY KEY,
  model TEXT NOT NULL,
  dim INTEGER NOT NULL,
  embedding BLOB NOT NULL,
  created_at TEXT NOT NULL
);

CREATE TABLE radar_entities (
  id TEXT PRIMARY KEY,
  kind TEXT NOT NULL,
  name TEXT NOT NULL,
  canonical_name TEXT,
  aliases_json TEXT NOT NULL DEFAULT '[]',
  created_at TEXT NOT NULL
);

CREATE TABLE radar_document_entities (
  document_id TEXT NOT NULL,
  entity_id TEXT NOT NULL,
  confidence REAL NOT NULL,
  PRIMARY KEY (document_id, entity_id)
);

CREATE TABLE radar_clusters (
  id TEXT PRIMARY KEY,
  title TEXT NOT NULL,
  topic TEXT,
  summary TEXT,
  first_seen_at TEXT NOT NULL,
  last_seen_at TEXT NOT NULL,
  heat_score REAL NOT NULL DEFAULT 0,
  novelty_score REAL NOT NULL DEFAULT 0
);

CREATE TABLE radar_cluster_items (
  cluster_id TEXT NOT NULL,
  raw_item_id TEXT NOT NULL,
  similarity REAL,
  PRIMARY KEY (cluster_id, raw_item_id)
);

CREATE TABLE radar_summaries (
  id TEXT PRIMARY KEY,
  target_type TEXT NOT NULL,
  target_id TEXT NOT NULL,
  summary_short TEXT NOT NULL,
  summary_technical TEXT,
  why_it_matters TEXT,
  learning_notes TEXT,
  model TEXT,
  created_at TEXT NOT NULL
);

CREATE TABLE radar_metrics_snapshots (
  id TEXT PRIMARY KEY,
  target_type TEXT NOT NULL,
  target_id TEXT NOT NULL,
  metric_name TEXT NOT NULL,
  metric_value REAL NOT NULL,
  observed_at TEXT NOT NULL,
  raw_json TEXT
);

CREATE TABLE radar_trends (
  id TEXT PRIMARY KEY,
  topic TEXT NOT NULL,
  window TEXT NOT NULL,
  current_count INTEGER NOT NULL,
  baseline_count REAL NOT NULL,
  z_score REAL NOT NULL,
  velocity REAL NOT NULL,
  explanation TEXT,
  created_at TEXT NOT NULL
);
```

迁移策略：

- 新 crate 内维护 `store/migrations.rs`，使用显式 schema version。
- Radar 是新增本地数据库，不做旧数据兼容。
- 每次 schema 改动必须同步更新 design、human_tests 和迁移测试。

## 配置设计

默认配置路径：

```text
~/.bifrost/radar/config.toml
```

测试必须使用临时 `BIFROST_DATA_DIR` 或显式 `--config`，避免污染用户真实 Radar 数据。

配置草案：

```toml
[radar]
enabled = true
data_dir = "~/.bifrost/radar"
timezone = "Asia/Shanghai"
default_language = "zh-CN"
summary_language = "zh-CN"

[scheduler]
enabled = false
max_concurrent_sources = 8
default_interval_minutes = 60
per_source_timeout_sec = 120

[llm]
provider = "agent-default"
summary_model = "default"
embedding_model = "text-embedding-3-small"
max_daily_summary_items = 120
max_summary_input_chars = 12000

[store]
type = "sqlite"
path = "~/.bifrost/radar/radar.db"
enable_fts = true
enable_vector = false
vector_backend = "none"

[dedup]
content_hash = true
simhash = true
embedding_threshold = 0.88
cluster_time_window_hours = 72

[scoring]
freshness_half_life_hours = 36
semantic_duplicate_threshold = 0.88
trend_zscore_threshold = 2.0

[scoring.weights]
github_star_velocity = 1.4
hn_comments = 1.2
cross_source_mentions = 1.5
trusted_author = 1.3
official_release = 1.5
paper_with_code = 1.2
low_quality_media = -0.8
duplicate = -1.0

[output]
daily_digest_time = "09:00"
weekly_digest_day = "SUN"
```

Source 配置：

```toml
[[sources]]
id = "hn-ai"
kind = "hackernews"
name = "Hacker News AI"
enabled = true
interval_minutes = 30
lists = ["topstories", "beststories", "showstories"]
keywords = ["AI", "LLM", "agent", "MCP", "RAG", "OpenAI", "Claude", "local LLM"]

[[sources]]
id = "github-agent"
kind = "github"
name = "GitHub Agent Radar"
enabled = true
interval_minutes = 60
queries = [
  "topic:llm stars:>100 pushed:>2026-01-01",
  "topic:agent stars:>50 pushed:>2026-01-01",
  "mcp in:name,description stars:>20"
]
token_env = "GITHUB_TOKEN"

[[sources]]
id = "company-blogs"
kind = "rss"
name = "AI Company Blogs"
enabled = true
interval_minutes = 120
feeds = [
  "https://openai.com/blog/rss.xml",
  "https://www.anthropic.com/news/rss.xml"
]

[[sources]]
id = "papers-agent"
kind = "paper"
name = "Agent Papers"
enabled = true
interval_minutes = 240
providers = ["arxiv", "openalex", "semanticscholar"]
queries = [
  "large language model agent",
  "retrieval augmented generation evaluation",
  "code generation agent",
  "browser agent"
]

[[sources]]
id = "reddit-ai"
kind = "reddit"
name = "Reddit AI Practice"
enabled = true
interval_minutes = 120
subreddits = ["LocalLLaMA", "MachineLearning", "singularity", "OpenAI", "ClaudeAI"]
keywords = ["agent", "MCP", "RAG", "coding", "browser"]
token_env = "REDDIT_TOKEN"

[[sources]]
id = "youtube-ai"
kind = "youtube"
name = "YouTube AI Channels"
enabled = true
interval_minutes = 180
channel_feeds = [
  "https://www.youtube.com/feeds/videos.xml?channel_id=<channel-id>"
]
transcript = "optional"

[[sources]]
id = "x-ai"
kind = "x"
name = "X AI Radar"
enabled = true
interval_minutes = 60
strategy = "api_or_rss_fallback"
accounts = ["karpathy", "sama"]
keywords = ["agent", "MCP", "AI coding", "RAG eval"]
token_env = "X_BEARER_TOKEN"

[[sources]]
id = "dynamic-ai-pages"
kind = "dynamic_web"
name = "Dynamic AI Pages"
enabled = true
interval_minutes = 240
engine = "crawl4ai"
allow_domains = ["github.com", "docs.anthropic.com", "openai.com"]
urls = []
max_pages_per_run = 20
timeout_sec = 60
```

## Source Adapter 设计

### 通用 trait

```rust
#[async_trait]
pub trait SourceAdapter: Send + Sync {
    fn kind(&self) -> SourceKind;
    async fn fetch(&self, task: SourceTask, checkpoint: Option<Checkpoint>) -> RadarResult<FetchBatch>;
}
```

`FetchBatch` 必须包含：

- `items: Vec<RawItem>`
- `next_checkpoint: Option<Checkpoint>`
- `rate_limit: Option<RateLimitState>`
- `source_warnings: Vec<String>`

### GitHub

V1 使用 GitHub API 而不是爬页面：

- repo search：按 topic、stars、pushed、language 查询。
- release feed：跟踪已关注 repo 的 release。
- star snapshot：记录 stars/forks/open issues/watchers 的时间序列，用于 24h / 7d velocity。
- token 从 `token_env` 读取，不落盘。

去重 key：

- `repo:<owner>/<name>`
- `release:<owner>/<name>@<tag>`
- `discussion/issue` 后续再做。

### Hacker News

V1 使用官方结构化 API：

- `topstories`
- `beststories`
- `showstories`
- 可选 `newstories`

拉取流程：

1. 拉取 story id 列表。
2. 限制前 N 条，避免每轮请求过多。
3. 拉取 item detail。
4. 对 title/url/text 做关键词预筛。
5. 记录 score、descendants、time。

去重 key：

- `hn:<id>`
- canonical URL 与其他来源聚合。

### RSS

V1 使用 `feed-rs` 或同类库解析：

- 支持 Atom/RSS。
- 支持 per-feed ETag/Last-Modified checkpoint。
- 对 feed URL、entry id、link 做硬去重。
- HTML content 进入 cleaner。

RSS 不负责动态页面执行；遇到只有摘要的 feed，先保存摘要和 URL，由 dynamic_web adapter 按 allowlist 补抓正文。

### Paper

V1 必须接入论文源：

- arXiv：按 query 拉取论文 metadata、摘要、分类、作者、发布时间。
- OpenAlex：补 works 搜索、引用/概念 metadata、open access URL。
- Semantic Scholar：补摘要、citation/reference count、recommendations。
- Paper adapter 只保存 metadata、abstract、公开 full text URL；PDF 下载和全文解析作为可配置开关，不默认执行。

去重 key：

- DOI。
- arXiv id。
- OpenAlex work id。
- Semantic Scholar paper id。
- title + first author + year fallback。

### Reddit

V1 必须接入 Reddit 作为实践反馈源：

- 优先官方 API；没有 token 时可使用公开 JSON/RSS fallback，但必须标记 `auth_mode = fallback`。
- 支持 subreddit allowlist、关键词过滤、score/comment metadata。
- 评论全文抓取默认关闭，V1 只抓 top-level post 和可配置 top comments。

去重 key：

- `reddit:<subreddit>:<post_id>`。
- canonical URL 与其他来源聚合。

### YouTube

V1 必须接入 YouTube：

- channel RSS 是默认稳定入口。
- metadata 包括标题、发布时间、频道、视频 URL、描述。
- transcript 是可选增强；未安装 transcript provider 时保存 metadata 并输出 `transcript_unavailable` warning。
- 视频内容摘要必须标注是否基于 transcript，不能把 metadata 摘要伪装成已看完整视频。

去重 key：

- `youtube:<video_id>`。
- canonical URL。

### X/Twitter

V1 必须接入 X/Twitter，因为 AI 项目和观点经常首发于 X。接入必须是多策略降级，而不是单点依赖：

1. 官方 API：用户提供 `X_BEARER_TOKEN` 时使用。
2. 第三方 API：用户显式配置 provider 和 token 时使用。
3. RSSHub/Nitter fallback：仅在用户显式配置 route/base URL 时使用。
4. 手工 watch URL：用户可以把单条 tweet/thread URL 加入 watch list，Radar 保存 URL 与可抓取 metadata。

限制：

- 不绕过登录墙或站点反爬策略。
- 不把不可用 fallback 当作错误中断全局采集。
- 必须记录 `source_warnings`，让 digest 知道 X source 是否部分失效。

去重 key：

- `x:<tweet_id>`。
- canonical URL。
- author + normalized text hash + time window fallback。

### Dynamic Web

V1 必须有动态网页 adapter，用来覆盖没有 API/RSS 的高价值页面。该 adapter 不是“全网爬虫”，而是受配置约束的补抓能力：

- 引擎：优先 Crawl4AI；Rust-native Spider 或 Playwright sidecar 可作为后续可选 engine。
- 输入：allowlist domain、显式 URL、从 RSS/HN/GitHub/X/Reddit 发现的 URL。
- 输出：Markdown、plain text、截图 metadata 可选。
- 限制：每轮最大页面数、单页 timeout、最大正文 bytes、最大重定向次数。
- 隔离：sidecar 进程必须可 kill；不在 Agent core 内执行浏览器；不共享用户浏览器 profile/cookies，除非用户明确配置。
- 合规：记录 robots/站点策略检查结果；对禁止抓取的页面保存 URL 和 metadata，不抓正文。

## Pipeline 设计

```text
SourceTask
  -> FetchBatch
  -> RawItem
  -> CleanDocument
  -> DedupCandidate
  -> ScoreInput
  -> IntelligenceEvent
  -> Cluster
  -> Summary
  -> Digest / TrendReport / LearningNote
```

### Fetch

- 每个 source 有独立 timeout、rate limit、checkpoint。
- 单轮失败只更新 `radar_source_checkpoints.last_error`，不阻断其他 sources。
- 网络请求必须设置 user agent，遵守 API rate limit。
- 默认并发由 `scheduler.max_concurrent_sources` 控制。

### Clean

输入来源：

- RSS content/summary。
- HN title/text/url metadata。
- GitHub repo description/README/release notes。
- Paper abstract / metadata / optional full text。
- Reddit post text / selected comments。
- YouTube metadata / optional transcript。
- X post/thread text。
- Dynamic Web Markdown。

清洗目标：

- `markdown`：保留标题、列表、代码块、链接。
- `text`：用于 FTS、hash、摘要输入。
- `content_hash`：归一化文本 hash。
- `title_hash`：归一化 title hash。
- `simhash`：可选。

V1 必须同时支持轻量 HTML to Markdown 和动态网页 Markdown 输出。浏览器执行只能通过 dynamic_web sidecar 进入，不能混入普通 cleaner 或 Agent tool runtime。

### Dedup

三层去重：

1. 硬去重：
   - `canonical_url`
   - normalized URL
   - `content_hash`
   - `title_hash`
2. 近似去重：
   - SimHash hamming distance
   - title similarity
   - author + time window
3. 语义聚合：
   - embedding cosine similarity
   - 同一 repo / paper / model / company
   - 同一实体 + 72h 时间窗口

V1 默认要求第 1 层和 SimHash 近似去重；embedding 语义聚合可默认关闭，但接口、阈值配置和降级路径必须在 V1 完成。

### Score

基础公式：

```text
importance =
  source_weight
  + freshness_score
  + author_weight
  + cross_source_mentions
  + engagement_score
  + repo_growth_score
  + novelty_score
  + learning_value_score
  - duplicate_penalty
```

GitHub repo score：

```text
repo_score =
  log(stars + 1)
  + 24h_star_delta * 2
  + 7d_star_delta
  + release_recent_bonus
  + issue_activity
  + maintainer_reputation
  + topic_match_score
```

原则：

- LLM 不能作为唯一重要性裁判。
- 结构化信号优先：star velocity、HN comments、cross-source mentions、official release。
- LLM 只补充 `why_it_matters`、`learning_value_score` 的解释和标签建议。

### Summarize

摘要输入必须带来源和证据：

- 原始标题。
- URL。
- source kind。
- 关键 metadata。
- 清洗文本截断片段。
- 去重/聚合依据。

摘要输出必须结构化：

```json
{
  "summary_short": "一句话",
  "summary_technical": "技术摘要",
  "why_it_matters": "为什么值得关注",
  "suitable_for": ["Agent", "Infra"],
  "recommended_action": "read|try|track|ignore",
  "tags": ["mcp", "ai-coding"],
  "confidence": 0.82
}
```

LLM 失败时：

- 保留 raw/document。
- digest 中可显示未摘要项。
- 不因摘要失败回滚采集。

### Trend

V1 趋势按 topic/entity/repo 统计：

- `current_count`
- `baseline_count`
- `z_score`
- `velocity`
- `cross_source_mentions`

窗口：

- 当前：24h / 7d。
- baseline：7d / 30d。

趋势输出不能只说“变热”，必须包含证据：

- 哪些来源增长。
- 代表性项目/文章。
- 与前窗口的倍数或 z-score。
- 置信度和可能误判原因。

## Agent 和 MCP 工具协议

### `radar_collect_now`

手动触发采集。

```json
{
  "sources": ["github", "hackernews", "rss"],
  "topics": ["agent", "mcp", "ai coding"],
  "max_items": 200
}
```

输出：

```json
{
  "fetched": 120,
  "new_items": 34,
  "deduplicated": 16,
  "summarized": 20,
  "warnings": []
}
```

### `radar_digest`

```json
{
  "scope": "daily",
  "topics": ["ai-coding", "agent", "mcp"],
  "since": "24h",
  "format": "brief"
}
```

### `radar_search`

混合搜索：FTS keyword + optional vector。

```json
{
  "query": "browser agent 最近有什么新项目",
  "since": "30d",
  "limit": 20,
  "include_sources": true
}
```

### `radar_trends`

```json
{
  "window": "7d",
  "baseline": "30d",
  "topics": ["mcp", "rag", "browser-use", "ai-coding"]
}
```

### `radar_explain`

```json
{
  "event_id": "evt_xxx",
  "level": "technical"
}
```

### `radar_learn`

```json
{
  "topic": "MCP ecosystem",
  "level": "intermediate",
  "duration": "7d",
  "include_projects": true
}
```

### `radar_watch`

```json
{
  "topic": "AI Coding Agent",
  "keywords": ["codex", "devin", "openhands", "claude code"],
  "sources": ["github", "hn", "rss"],
  "frequency": "daily"
}
```

## 输出模板

### Daily Digest

```markdown
# AI Research Radar Daily

## 1. 今天最重要的 5 件事

### 1. <title>
- 一句话：...
- 技术价值：...
- 为什么值得关注：...
- 适合谁看：Agent / Infra / RAG / AI Coding
- 推荐动作：阅读 / 试用 / 跟踪 / 忽略
- 来源：GitHub + HN + Blog

## 2. 新项目

| 项目 | 方向 | 24h 增长 | 价值判断 |
| --- | --- | ---: | --- |

## 3. 趋势信号

- MCP 工具生态讨论增加 2.8x。
- Browser Agent 新项目数量上升。
- RAG 讨论从向量库转向 evaluation / memory。

## 4. 今天值得深入学习

主题：Browser-use / Computer-use Agent

建议阅读顺序：

1. 基础文章
2. GitHub 项目
3. HN 讨论
4. 实践任务

## 5. 给我的实践任务

- 用 Bifrost Agent 接一个简单 RSS Radar source。
- 对 20 条 HN item 做摘要和聚类。
```

### Learning Route

```markdown
# MCP 生态 7 天学习路线

## 你需要先理解

- MCP 的 resources / tools / prompts 差异。
- stdio / Streamable HTTP transport。
- tool approval / permission model。

## 推荐阅读

1. 官方规范。
2. 高质量实现。
3. 社区争议讨论。
4. 真实项目代码。

## 实践任务

Day 1: 写一个 stdio MCP server
Day 2: 接入 Bifrost Agent
Day 3: 增加 resources
Day 4: 做 deferred tool discovery
Day 5: 做权限控制
Day 6: 写 human test
Day 7: 总结设计文档
```

## 调度与长期运行

### 运行模式

1. 手动 CLI：
   - `bifrost radar run`
   - 适合开发和调试。
2. 本地 scheduler：
   - Bifrost 启动时可选启动 Radar scheduler。
   - 必须默认关闭，避免用户未配置时产生外部请求。
3. MCP on demand：
   - Agent 调用 `radar_collect_now` 触发短采集。
4. 后续 automation：
   - 可接入 Bifrost/Agent 任务系统生成日报。

### 并发与取消

- 每个 source task 必须有 timeout。
- `radar_collect_now` 必须可取消。
- scheduler 不允许同时对同一个 source 运行多个 fetch。
- 长摘要任务必须可记录 partial failure。

## 安全、隐私与合规边界

### Secrets

- API token 只从环境变量或现有安全配置读取。
- 不写入 SQLite。
- 不写入 summary。
- MCP tool result 不回显 token。

### 网络行为

- 默认不启用系统代理，不依赖 Bifrost proxy 修改系统网络。
- 采集请求必须有可识别 user agent。
- 尊重 API rate limit。
- X/Twitter、Reddit、YouTube transcript、论文 API 和动态网页都必须支持 `auth_required` / `fallback_unavailable` / `rate_limited` 等可诊断状态，不能静默失败。
- 动态页面爬虫必须独立隔离资源、超时、robots/站点条款风险；禁止无 allowlist 的全网扩散。

### 本地数据

- 默认数据目录：`~/.bifrost/radar/`。
- 测试使用临时目录。
- 用户可执行 `bifrost radar export` 和 `bifrost radar clear`，后续 PR 再设计。
- 摘要和学习路线必须保留来源 URL，避免知识无出处。

### Agent 边界

- Agent 可以查询 Radar 输出，但不应在无用户请求时主动外发本地 Radar DB。
- `radar_watch` 只新增本地关注主题，不自动订阅外部账号或发布消息。
- MCP 工具必须按 Bifrost 现有 approval/visibility 策略接入，不绕过工具权限层。

## 实施路线

### PR 1：`feat(radar): add local AI research radar core with multi-source collectors`

范围：

- 新增 `crates/bifrost-radar`。
- SQLite schema + migrations。
- config loader。
- RSS collector。
- HN collector。
- GitHub collector。
- Paper collector：arXiv / OpenAlex / Semantic Scholar。
- Reddit collector。
- YouTube collector：channel RSS + optional transcript。
- X/Twitter collector：API or configured fallback。
- Dynamic Web adapter：Crawl4AI sidecar + allowlist URL fetch。
- URL/hash dedup。
- SimHash near-duplicate dedup。
- FTS keyword search。
- digest generator。
- CLI：`init`、`run`、`digest`、`search`。
- `design/ai-research-radar.md` 按实现更新。
- `human_tests/ai-research-radar.md` 创建并执行。

不包含：

- MCP server。
- Agent 内置工具。
- 复杂 WebUI。
- 自动 IM/外部平台推送。
- 未配置 source 的全网爬取。

### PR 2：`feat(agent): expose AI radar through MCP tools`

范围：

- `bifrost radar serve-mcp`。
- MCP tools：`radar_digest`、`radar_search`、`radar_collect_now`。
- example config。
- Agent 配置示例。
- deferred tool discovery 验证。
- human test 覆盖真实 Bifrost + MCP 调用。

### PR 3：`feat(radar): add trend detection and learning route`

范围：

- GitHub/HN metric snapshots。
- topic/entity extraction。
- trend z-score。
- `radar_trends`。
- `radar_learn`。
- weekly review 输出。

### PR 4：`feat(radar): harden source quality and optional vector search`

范围：

- source health dashboard。
- X fallback 可用性监控。
- dynamic_web engine 切换。
- transcript provider 扩展。
- optional sqlite-vec or LanceDB backend。
- WebUI 最小管理面。

## 测试方案

### 单元测试

PR 1 必须至少覆盖：

- `config_loads_default_radar_paths`：默认路径、source 配置、token env 字段解析。
- `rss_source_parses_atom_and_rss_entries`：Atom/RSS 条目解析。
- `hackernews_filters_keywords_and_preserves_scores`：HN item 关键词过滤和 score/comment metadata。
- `github_source_builds_repo_raw_items`：GitHub repo/release metadata 转 RawItem。
- `paper_source_deduplicates_by_external_ids`：arXiv/OpenAlex/Semantic Scholar 外部 ID 去重。
- `reddit_source_uses_fallback_status_when_unauthenticated`：未配置 token 时明确 fallback 或 auth_required。
- `youtube_source_preserves_transcript_availability`：视频 metadata 与 transcript 可用性分离。
- `x_source_reports_fallback_unavailable_without_blocking`：X fallback 不可用时记录 source warning 且不阻断其他源。
- `dynamic_web_respects_allowlist_and_timeout`：动态网页 adapter 只抓 allowlist URL 且 timeout 可控。
- `dedup_uses_canonical_url_and_content_hash`：URL/hash 硬去重。
- `dedup_uses_simhash_for_near_duplicates`：标题/正文轻微变化时聚合为近似重复。
- `fts_search_returns_ranked_documents`：FTS 搜索排序。
- `digest_groups_items_by_importance`：digest 输出按 score 和来源证据排序。
- `migrations_initialize_schema_version`：schema version 初始化和重复运行幂等。

### E2E 测试

PR 1 新增 shell E2E：

```text
e2e-tests/tests/test_radar_cli_core.sh
```

验证点：

1. 使用临时 `BIFROST_DATA_DIR`。
2. `bifrost radar init` 创建 config 和 SQLite。
3. 使用本地 HTTP fixture 提供 RSS feed 和 HN/GitHub mock API。
4. 使用本地 fixture 提供 arXiv/OpenAlex/Semantic Scholar、Reddit、YouTube、X fallback 和 dynamic_web 页面。
5. `bifrost radar run` 采集固定 fixtures。
6. `bifrost radar search "browser agent"` 返回跨源 fixture 条目。
7. `bifrost radar digest --since 24h` 输出来源、摘要、推荐动作。
8. 重复执行 `radar run` 不产生重复 raw item/document。
9. X fallback 不可用、YouTube transcript 不可用或 dynamic_web 超时时，其他 source 仍然成功，digest 显示 warnings。

PR 2 新增 shell E2E：

```text
e2e-tests/tests/test_radar_mcp_agent.sh
```

验证点：

1. 启动真实 Bifrost，使用临时数据目录和 `--no-system-proxy`。
2. 配置 `mcp_servers.ai_radar` 指向 `bifrost radar serve-mcp`。
3. 通过 `/agent/chat` 调用 `radar_search` 或 `radar_digest`。
4. 当 MCP tool 数量后续扩展到阈值时，验证 `tool_search` 能发现 Radar 工具。

### 真实场景测试

新增：

```text
human_tests/ai-research-radar.md
```

必须覆盖：

- 设计文档完整性评审。
- `radar init/run/search/digest` CLI 主链路。
- 本地 fixture 去重回归。
- MCP Server 接入 Agent 的真实链路。
- scheduler 默认关闭和 `--no-system-proxy` 边界。
- 数据目录和 token 不落盘边界。

### 工作区验证

代码实施 PR 最终必须执行：

- `cargo fmt --all -- --check`
- `cargo clippy --workspace --all-targets --all-features -- -D warnings`
- `cargo test -p bifrost-radar --all-features`
- `cargo test -p bifrost-cli radar --all-features`
- `cargo test --workspace --all-features`
- 相关 E2E 脚本。
- `human_tests/ai-research-radar.md` 全部用例。
- 按修改范围决定是否执行 `scripts/ci/local-ci.sh`。

本设计文档变更本身不修改 Rust/WebUI/脚本/配置运行行为，单元测试、E2E、workspace all-features、local-ci 可标记为不适用；但必须执行文档型 human test 和两轮 Review/Fix/Test。

## Review/Fix/Test 闭环方案

### 第 1 轮

复核：

- 用户目标是否都落入设计文档。
- 是否明确“先评审后实施”。
- 是否控制 V1 范围。
- 是否覆盖 Bifrost 当前架构接入点。
- 是否更新 `human_tests/readme.md`。

命令：

```bash
source ~/.zshrc
git status --short
git diff -- design/ai-research-radar.md human_tests/ai-research-radar.md human_tests/readme.md
rg -n "AI Research Radar|radar_" design/ai-research-radar.md human_tests/ai-research-radar.md human_tests/readme.md
```

### 第 2 轮

复核：

- 第 1 轮发现的问题是否修复。
- 文档是否没有承诺已实现能力。
- 测试矩阵是否区分设计文档变更和后续实现 PR。
- git diff 是否只包含本次文档文件。

命令：

```bash
source ~/.zshrc
git status --short
git diff --stat
git diff --check
```

## 文档更新要求

本次设计文档变更：

- 新增 `design/ai-research-radar.md`。
- 新增 `human_tests/ai-research-radar.md`。
- 更新 `human_tests/readme.md` 索引和总计。

后续实现 PR 还必须按实际变更同步：

- CLI docs / README：新增 `bifrost radar` 命令时更新。
- Agent/MCP docs：新增 MCP server 和工具时更新。
- human_tests：每个 PR 更新并执行。

## 风险与待评审问题

1. 模型成本：摘要、学习路线、embedding 可能带来稳定成本，V1 必须支持只采集和 FTS 搜索。
2. API 配额：GitHub/HN/RSS/Reddit/YouTube/X/论文源都需要 per-source checkpoint 和退避，不可让单源失败拖垮全局。
3. 事实可信度：摘要必须保留来源，不允许生成无来源结论。
4. 重复信息：V1 做硬去重和 SimHash，语义聚合默认可关闭；多源事件合并质量需要真实数据集继续校准。
5. 向量后端：sqlite-vec/LanceDB 不应在 V1 强制引入，避免安装复杂度。
6. 动态爬虫：Crawl4AI/Spider/Playwright 必须作为隔离 sidecar，不能进入 Agent core；需要评审资源限制、cookie/profile 禁用默认值和 kill 语义。
7. Agent 自主性：scheduler 默认关闭；日报自动生成或 IM 推送必须由用户显式开启。
8. 数据保留：需要评审默认保留天数、最大 DB 大小、附件/正文是否保存全文。
9. Source 可用性：X/Nitter/RSSHub/YouTube transcript 等来源可能不稳定，V1 必须把 source health 和 partial failure 作为一等输出。
