# 🚀 Reddit 热帖自动化 · n8n → 飞书多维表 → LLM 洞察指南

## ✨ 能力概览
- 🤖 定时抓取指定 Subreddit 的热门帖子，自动过滤 NSFW、置顶或低分内容。
- 📊 将帖子关键信息结构化写入飞书多维表，便于追踪与二次加工。
- 🧠 调用 DeepSeek LLM 产出爆火分析、选题建议与改写文案，并推送飞书群机器人。

Reddit → n8n (三套工作流) → Feishu Bitable & Webhook
↘ DeepSeek LLM ↗



本项目提供三份 n8n 工作流，请按顺序导入和配置：

| 工作流 | 作用 | 触发方式 |
| --- | --- | --- |
| `Reddit · Initialize Feishu Bitable Fields (Run Once)` | 初始化飞书表字段（含 LLM 解析字段） | 手动运行一次 |
| `Reddit Hotposts to Feishu Bitable (Batch)v3` | 周期性抓取 + 写入飞书表 | 每 6 小时 + 手动测试 |
| `Reddit - High Score Rewrite Broadcast v3` | 调用 LLM 生成洞察并推送飞书群 | 每日一次 + 手动测试 |

---

## ✅ 准备工作

### 1. 环境要求
- 已部署的 n8n 实例（建议 v1.40+）。
- Reddit OAuth2 应用（只读权限）。
- 飞书自建应用，已开通 Bitable 与机器人 Webhook 能力。
- DeepSeek API Key（或兼容 OpenAI 接口的模型 API）。

### 2. n8n Credentials
- **Reddit OAuth2**：用于拉取帖子。
- **DeepSeek API**：HTTP 节点调用 `chat/completions`。
- 飞书接口通过 HTTP 节点动态获取 `tenant_access_token`，无需额外 Credential。

### 3. 环境变量

| 变量 | 示例 | 说明 |
| --- | --- | --- |
| `REDDIT_CLIENT_ID` | `abc123xyz` | Reddit OAuth2 Client ID |
| `REDDIT_CLIENT_SECRET` | `def456uvw` | Reddit OAuth2 Client Secret |
| `REDDIT_USER_AGENT` | `hotpost-n8n/0.1 (by u/yourname)` | Reddit API User-Agent |
| `FEISHU_APP_ID` | `cli_xxxxx` | 飞书应用 ID |
| `FEISHU_APP_SECRET` | `xxxxxx` | 飞书应用 Secret |
| `FEISHU_BITABLE_REDDIT_APP_TOKEN` | `HGeEb…` | Reddit 贴子表 `app_token` |
| `FEISHU_BITABLE_REDDIT_TABLE_ID` | `tblOa9…` | Reddit 贴子表 `table_id` |
| `FEISHU_WEBHOOK_URL` | `https://open.feishu.cn/…` | 飞书群机器人地址 |
| `DEEPSEEK_MODEL` | `deepseek-chat` | LLM 模型名 |
| `DEEPSEEK_BASE_URL` *(可选)* | `https://api.deepseek.com/v1/chat/completions` | LLM 接口地址 |
| `LIMIT_PER_SUB` *(可选)* | `30` | 每个 Subreddit 默认抓取数量 |
| `SCORE_THRESHOLD` *(可选)* | `50` | 过滤分数阈值 |

> 预留变量 `FEISHU_BITABLE_REDDIT2_APP_TOKEN` / `FEISHU_BITABLE_REDDIT2_TABLE_ID` 暂未使用，可留空或用于后续扩展。

---

## 🛠️ 导入与初始化

1. **导入工作流 JSON**  
   在 n8n 中依次导入三份 JSON，检查工作流名称与结构是否正确。

2. **配置 Reddit OAuth2 Credential**  
   - Redirect URL 可用 cloudflare tunnel 固定为 http://临时公⽹/rest/oauth2-credential/callback （如：https://job-partnerships-anatomy-wrote.trycloudflare.com/rest/oauth2-credential/callback ）。
   - Scope 选择 `read`。

3. **配置 DeepSeek Credential**  
   - 在 Credentials 中创建 `DeepSeek API`，填写 API Key。若更换模型服务商，请同步更新 `DEEPSEEK_BASE_URL`。

4. **初始化飞书表字段**  
   - 打开 `Reddit · Initialize Feishu Bitable Fields (Run Once)`，点击 *Execute Workflow*。
   - 工作流会自动创建包括 LLM 输出在内的全部字段：  
     `subreddit`, `post_id`, `title`, `author`, `content`, `score`, `num_comments`, `created_utc`, `permalink`, `url`, `thumbnail`, `over_18`, `stickied`, `created_at`, `热点趋势`, `爆火原因`, `选题建议`, `文案建议（含推荐标签）`, `相关性`。
   - 若字段已存在，工作流会跳过并在总结节点里提示。

5. **校验环境变量**  
   - `Reddit Hotposts to Feishu Bitable (Batch)v3` 中的 `Config` 节点可直观查看变量解析结果。
   - 修改 `SUBREDDIT_LIST` 为目标社区（例如 `["MachineLearning","LocalLLaMA","Automation"]`）。
   - 想额外通过关键词补充热点时，可设置 `KEYWORD_LIST`（如 `["n8n workflow","automation growth"]`）。

---

## 🔄 工作流说明

### 1. Reddit · Initialize Feishu Bitable Fields (Run Once)
- 自动创建飞书多维表字段，支持重复执行。
- 关键节点：`Configuration` → `Get Feishu Token` → `Prepare Field List` → `Create Field` → `Summarize`。
- 运行结果会说明新增 / 已存在 / 失败的字段数量。

### 2. Reddit Hotposts to Feishu Bitable (Batch)v3
- **触发**：内置 Cron 每 6 小时，可改成任意频率。
- **主要步骤**：
  - 拆分 Subreddit → Reddit API 获取热帖 → 过滤 NSFW/置顶/低分 → 字段映射 → 批内去重。
  - 与飞书表对比 `post_id`，跳过已存在记录。
  - 分批写入飞书表（默认每批 10 条）。
  - 如设置 `KEYWORD_LIST`，还会额外搜索全站关键词结果。
- **Config 节点可自定义的关键参数**：
  - `SUBREDDIT_LIST`：数组格式的社区列表，示例 `["MachineLearning","LocalLLaMA"]`。可直接在节点里编辑，也可以改为从环境变量读取。
  - `LIMIT_PER_SUB`：每个社区抓取的帖子数，默认取环境变量 `LIMIT_PER_SUB`（无则为 30）。
  - `SCORE_THRESHOLD`：保留的最低得分，默认取环境变量 `SCORE_THRESHOLD`（无则为 50）。
  - `KEYWORD_LIST`：可选的关键词数组，用于触发 “Search for a post” 分支进行全站搜索；留空或删除即不执行关键词检索。
  - `REDDIT_USER_AGENT`：Reddit API 的 UA 字符串，若需区分环境可分别设置。
- **可调参数**：Cron 周期、抓取数量、分数阈值、关键词列表等。

### 3. Reddit - High Score Rewrite Broadcast v3
- **触发**：默认每日执行，可自行调整。
- **主要步骤**：
  - 通过视图 ID （默认 `vewbLDXZ8j`）读取飞书表记录。
  - 初筛包含 `n8n` 等关键词的帖子。
  - 每批随机 10 条调用 DeepSeek LLM，生成热点趋势、爆火原因、选题、文案及相关性评分。
  - 将结果写回表中对应记录，同时筛出相关性 ≥8 的帖子推送飞书群卡片。
- **配置要点**：确保飞书群机器人 URL、DeepSeek Credential 和相关字段类型正确（建议多行文本 / 数字）。
- **视图配置说明**：
  - 默认使用的视图 ID `vewbLDXZ8j` 来自飞书多维表中的一个自定义视图：过滤条件为 `score ≥ 500` 且 `created_utc` 晚于 2025/01/01，并按 `subreddit` 分组。
  - 若需替换，可在飞书表中创建自己的视图（设置好过滤、排序、分组），然后从视图链接（例如 `https://...&view=XXXXXXXX`）中复制 `view=` 后面的 ID，替换工作流内 `Build Query` 节点中的值。

---

## ✅ 验证流程

1. 手动执行 **初始化** 工作流，确认 Summary 输出。
2. 测试执行 **抓取工作流**：确保 Reddit 节点返回 200，飞书写入返回 `code: 0`。
3. 测试执行 **洞察工作流**：检查 DeepSeek 返回的 JSON 是否完整，飞书表字段与卡片推送同步更新。
4. 在 n8n Execution 日志中查看详细输入输出，排查任何异常。

---

## 🐞 常见问题排查

- **字段重复错误（code 1254044）**  
  飞书字段已存在，忽略即可或手动确认字段名。
- **LLM 输出格式异常**  
  若解析失败，可降低 `temperature`，或检查源帖子正文是否超长。
- **抓取数量少**  
  Reddit 热门接口受社区活跃度限制，可增加更多 Subreddit 或开启关键词搜索。

---

## 🔮 后续扩展建议

- 将 `KEYWORD_LIST` 从飞书表中读取，实现动态配置。
- 在抓取流程添加自动翻译、情绪分析等节点。
- 使用预留的第二张表变量，将 LLM 结果同步至独立洞察表。

Happy automating! ✨
