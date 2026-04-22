# OpenClaw 从 0 搭建正式日报系统操作手册

## 文档目的

这份文档面向一个**没有上下文记忆的外部 agent**，目标是让它只靠这份文档，就能从 0 搭建一条 OpenClaw 正式日报链路。

前置假设：

- Hermes 已经存在
- SQLite 主库已经存在
- 主库中已有结构化 AI 候选素材
- 飞书应用和 docx 写入能力可用

OpenClaw 的职责不是抓取，而是：

- 从结构化候选中选出正式日报内容
- 读原文
- 写“摘要/分析”
- 生成飞书正式日报文档

## 一、OpenClaw 的定位

OpenClaw 是：

- **正式日报生成层**

不是：

- 全量素材抓取层
- 候选池主维护层

OpenClaw 只负责：

1. 从 Hermes 主库读取候选
2. 做正式日报选题
3. 对少量入选条目读原文
4. 写正式日报
5. 生成飞书 docx
6. 可选发送通知

## 二、核心原则

### 原则 1：OpenClaw 不生成事实字段

标题、链接、来源、时间必须来自数据库或原文，不允许 LLM 自由生成。

### 原则 2：先选题，再读原文

OpenClaw 不应一开始就让 LLM 读所有候选全文。

正确流程是：

1. 根据标题和结构化字段做初筛
2. 只对最终入选条目读原文
3. 再写摘要和分析

### 原则 3：正式日报强调可信度

大模型可以写：

- 摘要
- 分析
- 栏目说明

但不能改写：

- 标题
- 链接
- 来源

## 三、推荐目录结构

OpenClaw 侧建议使用：

```text
feishu-industry-daily/
├── generate-report.mjs
├── logs/
│   └── report_*.result.json
├── scripts/
│   └── （可选辅助脚本）
└── .env
```

如果目录名不同，可替换，但建议保留：

- 一个主生成脚本
- 一套日志/result文件
- 一套独立配置

## 四、输入数据要求

OpenClaw 从 SQLite 读取的候选数据，至少应具备：

- `id`
- `title`
- `url`
- `platform`
- `published_at`
- `fetched_at`
- `category`
- `content_type`
- `ai_relevance`
- `source_tier`
- `quality_score`
- `event_key`
- `openclaw_status`
- `ingest_version`

## 五、时间窗口原则

正式日报推荐基于：

- `fetched_at`

来做窗口过滤，而不是 `published_at`。

原因：

- 正式日报关注的是“最近进入素材池、经过系统清洗后值得发布的内容”
- `published_at` 容易受到旧文回流、补档等影响

## 六、正式日报生成链路

推荐拆成 4 步：

## Step 1：候选读取

从 SQLite 中读取当天候选。

推荐过滤条件：

- `ingest_version` 已完成结构化处理
- `quality_score` 非空
- `event_key` 非空
- `content_type` 非空
- `openclaw_status = pending`
- `fetched_at` 落在日报窗口内

## Step 2：标题层选题

这里不要先读全文，而是先用结构化字段做脚本筛选：

- 标题
- 分类
- `quality_score`
- `source_tier`
- `event_key`
- 时间

推荐顺序：

1. base filter
2. quality gate
3. event 去重
4. 分类分配
5. 总量截断

目标：

- 把候选缩到少量高质量条目

## Step 3：原文层写作

只对最终入选的少量条目：

- 读取原文
- 提取正文
- 让 LLM 写：
  - 摘要
  - 分析

不要让 LLM 自由生成标题或事实字段。

## Step 4：文档输出

程序用真实字段回填：

- 标题
- 链接
- 来源
- 时间

再附加：

- 摘要
- 分析

最后生成飞书 docx。

## 七、正式日报栏目建议

OpenClaw 可以使用以下栏目：

- `今日热点`
- `技术突破`
- `企业动态`
- `商业模式`
- `工具/教程`

但不要为了凑栏目强行补内容。

原则：

- 有就有
- 没有就少
- 不为结构完整牺牲真实性和时效性

## 八、LLM 的职责边界

OpenClaw 中，LLM 应只负责：

- 判断哪些入选内容更值得展开
- 写摘要
- 写分析

LLM 不应负责：

- 改标题
- 写链接
- 自由发明事件

## 九、飞书 docx 输出

### 1. 生成 docx

OpenClaw 负责：

- 新建 docx
- 写入正文
- 可选插入封面图

### 2. 写入成功条件

只有同时满足以下条件，才视为本轮成功：

- `docx_created = true`
- `sqlite_writeback_ok = true`
- `selected_items_count > 0`

### 3. 写回状态

成功后，应写回：

- `openclaw_status = published`
- `openclaw_selected_at`
- `openclaw_doc_id`
- `openclaw_doc_url`

## 十、通知链路

OpenClaw 的通知能力建议与 docx 生成解耦。

### 原则

- docx 生成是主产物
- 通知只是附加能力

### 建议配置

通知功能应走配置项控制，例如：

```bash
FEATURES_NOTIFY=true
```

不要硬编码关闭。

### 发送目标

建议使用明确配置的：

- `open_id`
- 或群聊目标

并做最小验证，确保：

- `notify_sent = true`

## 十一、错误分级建议

### blocking

- `LLM_FAILED`
- `DOCX_CREATE_FAILED`
- `DOCX_WRITE_FAILED`
- `SQLITE_WRITEBACK_FAILED`

### non-blocking

- `COVER_FAILED`
- `NOTIFY_FAILED`
- `BITABLE_SYNC_FAILED`

原则：

- 只有真正不影响主产物闭环的错误，才允许 non-blocking

## 十二、result schema 建议

建议统一使用：

- `schema_version`
- `system`
- `job_type`
- `run_id`
- `success`
- `status`
- `started_at`
- `finished_at`
- `duration_ms`
- `timezone`
- `source_of_truth`
- `sqlite_path`
- `bitable_mode`
- `error`
- `warnings`
- `metrics`
- `artifacts`
- `debug`

并在 `metrics` 中至少记录：

- `candidates_before_filter`
- `candidates_after_filter`
- `candidates_after_quality_gate`
- `candidates_after_event_dedup`
- `candidates_for_llm`
- `selected_items_count`
- `llm_input_items_count`
- `llm_input_chars`
- `llm_output_chars`
- `stage_durations_ms`

## 十三、最小实现步骤

如果从 0 开始搭建，按下面顺序：

### Step 1：接 SQLite

- 确认主库路径
- 能读取候选字段

### Step 2：实现标题层选题

- 基于结构化字段做预筛
- 不先读全文

### Step 3：实现原文层写作

- 对入选条目读原文
- 让 LLM 写摘要/分析

### Step 4：实现 docx 输出

- 写入飞书 docx
- 验证链接可访问

### Step 5：实现状态写回

- 成功后写 `openclaw_status`

### Step 6：实现通知

- 通过配置开关控制
- 先做最小消息验证

### Step 7：接定时任务

- 每天 `08:30` 触发

## 十四、验收标准

一条可用的 OpenClaw 正式日报链路，应满足：

1. 候选来自 SQLite，不来自飞书多维表
2. 标题层先选题，原文层后写作
3. 标题/链接/来源/时间 100% 来自 DB 或原文
4. 可成功生成飞书 docx
5. 成功后写回 `openclaw_status`
6. 可选通知能成功送达
7. result 文件结构稳定、可用于排障

## 十五、一句话总结

OpenClaw 从 0 搭建的关键不是“让模型自动写一篇日报”，而是：

**先用结构化候选做选题，再对少量入选条目读原文，用大模型写摘要和分析，最后由程序回填真实事实字段生成正式日报。**
