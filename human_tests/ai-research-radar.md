# AI Research Radar 真实场景测试用例

## 功能模块说明

验证 Bifrost AI Research Radar 的设计文档、后续 CLI/MCP/Agent 接入计划和安全边界是否可评审、可实施、可验证。本文件当前用于设计评审阶段；后续实现 PR 必须把未执行的实现型用例补充为真实执行结果。

## 前置条件

- 工作目录为仓库根目录。
- 执行任何 shell 命令前先运行 `source ~/.zshrc`。
- 本轮仅评审设计文档，不修改 Rust/WebUI/脚本实现。
- 不启动正式 9900 服务。
- 不修改系统代理。
- 不调用外部 GitHub/HN/RSS/Reddit/YouTube/X/论文 API；所有实现型真实链路留到后续 PR 使用本地 fixture 或显式用户配置。

## 测试用例列表

### TC-RADAR-01 设计文档覆盖用户提出的六个核心闭环

操作步骤：

1. 执行 `source ~/.zshrc && rg -n "采集|清洗|理解|去重|趋势|学习输出" design/ai-research-radar.md`。
2. 打开 `design/ai-research-radar.md` 的“目标与非目标”“Pipeline 设计”“输出模板”章节。
3. 确认文档分别覆盖采集、清洗、理解、去重聚合、趋势发现、学习输出。

预期结果：

- 六个核心闭环均可在设计文档中直接定位。
- 文档说明 Radar 不是搜索引擎，而是长期运行的技术情报系统。

本次执行结果：

- 2026-05-12 设计阶段执行通过。`rg` 能定位六个闭环，文档已分别写入目标、pipeline 和输出模板。

### TC-RADAR-02 Bifrost 架构接入路径清晰且不污染 Agent core

操作步骤：

1. 执行 `source ~/.zshrc && rg -n "ToolRegistry|MCP|tool_search|bifrost-radar|crates/agent|serve-mcp" design/ai-research-radar.md`。
2. 确认文档将 Radar Core 放入独立 `crates/bifrost-radar`。
3. 确认文档规划先 CLI/Core，再 MCP Server，最后少量 Agent 内置工具。

预期结果：

- 文档明确 Agent 只消费能力，Radar Core 生产知识。
- 文档没有要求把采集、清洗、embedding、趋势检测直接塞入 `crates/agent`。
- MCP 方案和 Agent 内置工具方案有清晰先后顺序。

本次执行结果：

- 2026-05-12 设计阶段执行通过。文档明确独立 crate、MCP 第二阶段、Agent 内置第三阶段。

### TC-RADAR-03 V1 范围覆盖所有关键数据来源并保持可实施边界

操作步骤：

1. 执行 `source ~/.zshrc && rg -n "V1|GitHub|Hacker News|RSS|论文源|Reddit|YouTube|X/Twitter|动态网页|Crawl4AI|SQLite|FTS|SimHash" design/ai-research-radar.md`。
2. 打开“V1 明确范围”“Source Adapter 设计”“实施路线”章节。
3. 确认 V1 必须覆盖 GitHub、HN、RSS、论文源、Reddit、YouTube、X/Twitter 和动态网页。
4. 确认文档通过 source allowlist、fallback、timeout、sidecar、source_warnings 和 scheduler 默认关闭控制风险。

预期结果：

- 文档不再把 X/Twitter、动态爬虫、Reddit、YouTube 或论文源列为 V1 不做。
- 文档明确这些关键来源进入 V1，但不做无边界全网采集、不绕过登录墙、不自动外发、不偷用未配置 API key。
- 第一批实现可以拆 PR，但 V1 完成定义必须包含这些关键 source 的采集、降级、去重和验证闭环。

本次执行结果：

- 2026-05-12 设计阶段执行通过。V1 多源必做、分级接入和风险边界已写入文档。

### TC-RADAR-04 数据、安全和隐私边界可审查

操作步骤：

1. 执行 `source ~/.zshrc && rg -n "Secrets|token|不落盘|rate limit|user agent|系统代理|数据目录|来源 URL|scheduler 默认关闭" design/ai-research-radar.md`。
2. 打开“配置设计”“安全、隐私与合规边界”“调度与长期运行”章节。
3. 确认 API token、网络行为、本地数据、Agent 边界均有约束。

预期结果：

- token 只从环境变量或安全配置读取，不写入 SQLite 或 summary。
- scheduler 默认关闭。
- 测试必须使用临时数据目录。
- 默认不修改系统代理。
- 摘要和学习路线必须保留来源 URL。

本次执行结果：

- 2026-05-12 设计阶段执行通过。安全边界已在文档中分节描述。

### TC-RADAR-05 后续实现验证矩阵完整

操作步骤：

1. 执行 `source ~/.zshrc && rg -n "单元测试|E2E|真实场景测试|cargo test --workspace --all-features|local-ci|test_radar_cli_core|test_radar_mcp_agent" design/ai-research-radar.md`。
2. 打开“测试方案”“Review/Fix/Test 闭环方案”“文档更新要求”章节。
3. 确认后续实现 PR 的单元测试、E2E、human_tests、workspace 验证、local-ci 策略都已规划。

预期结果：

- 文档包含 PR 1/PR 2 的 E2E 脚本建议。
- 文档区分本次设计文档变更的不适用测试和后续实现 PR 必跑测试。
- 文档包含两轮 Review/Fix/Test 闭环要求。

本次执行结果：

- 2026-05-12 设计阶段执行通过。测试矩阵和两轮闭环已写入设计文档。

### TC-RADAR-06 human_tests 索引同步

操作步骤：

1. 执行 `source ~/.zshrc && rg -n "ai-research-radar.md|AI Research Radar|总计" human_tests/readme.md`。
2. 确认 `human_tests/readme.md` 中存在 `ai-research-radar.md` 条目。
3. 确认总测试文件数和总用例数已随本文件增加。

预期结果：

- `human_tests/readme.md` 索引包含 AI Research Radar。
- 总计从 95 个测试文件、1624 个测试用例更新为 96 个测试文件、1630 个测试用例。

本次执行结果：

- 2026-05-12 设计阶段执行通过。索引和总计已同步。

## 后续实现 PR 必须补充执行的真实链路用例

以下用例当前不在本设计文档变更中执行，后续实现对应能力时必须补充为正式用例并记录真实结果：

- `TC-RADAR-CLI-01`：`bifrost radar init` 使用临时数据目录创建 config 和 SQLite。
- `TC-RADAR-CLI-02`：本地 RSS/HN/GitHub/论文源/Reddit/YouTube/X/dynamic_web fixture 通过 `bifrost radar run` 采集入库。
- `TC-RADAR-CLI-03`：重复 `radar run` 不产生重复 raw item/document。
- `TC-RADAR-CLI-04`：`radar search` 通过 FTS 返回排序结果和来源 URL。
- `TC-RADAR-CLI-05`：`radar digest --since 24h` 输出摘要、技术价值、推荐动作和来源。
- `TC-RADAR-CLI-06`：X fallback、YouTube transcript、dynamic_web 任一 source 失败时不阻断其他 source，并在 digest/source health 中展示 warning。
- `TC-RADAR-MCP-01`：真实 Bifrost + `mcp_servers.ai_radar` 调用 `radar_search`。
- `TC-RADAR-MCP-02`：Radar MCP 工具数量扩展后通过 `tool_search` 发现并调用。
- `TC-RADAR-SEC-01`：GitHub/Reddit/X 等 token 只从 env 读取，SQLite 和 summary 中不落盘。

## 清理步骤

- 本设计阶段不创建临时服务和数据库，无需停止服务。
- 后续实现型用例必须删除临时 `BIFROST_DATA_DIR`、停止 fixture server，并确认未修改系统代理。
