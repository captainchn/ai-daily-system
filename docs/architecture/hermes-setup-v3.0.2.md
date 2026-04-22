# Hermes 从 0 搭建飞书 AI 日报系统操作手册

## 文档目的

这份文档面向一个**没有上下文记忆的 agent**，目标是让它只靠这份文档，就能在一台已有 `Hermes` 运行环境的机器上，从 0 搭建出一套可运行的飞书 AI 日报系统。

这份文档只讲 **Hermes** 侧，不包含 OpenClaw 正式日报链路。

目标系统能力：

1. 定时抓取 AI 相关资讯
2. 写入 SQLite 主库
3. 同步到飞书多维表观察面
4. 在 `08:00 / 12:00 / 16:00` 生成 Hermes 快报
5. 通过飞书发送给指定目标

## 一、系统定位

Hermes 在这套系统里的角色不是“完整日报生成器”，而是：

- 定时抓取最新 AI 资讯
- 做结构化清洗和入库
- 在固定时间点发送“最新值得关注的信息”

Hermes 的定位是：

- **精选提醒层**
- 不是完整长文日报层

如果要做正式日报长文，应该交给 OpenClaw。

## 二、架构原则

这套系统必须遵守一个核心原则：

### 事实层脚本化，判断层交给大模型

脚本负责：

- 标题
- 链接
- 来源
- 发布时间
- 抓取时间
- 候选池
- 去重
- 状态写回
- 多维表同步

大模型只负责：

- 在候选中选择“值得关注”的条目
- 写摘要
- 写分析

**禁止让 LLM 生成或改写事实字段。**

## 三、目标运行节奏

Hermes 的标准节奏是：

### 抓取

- `07:30`
- `11:30`
- `15:30`

### 发送

- `08:00`
- `12:00`
- `16:00`

### 健康检查

- `07:40 / 11:40 / 15:40`：抓取检查
- `08:10 / 12:10 / 16:10`：发送检查

## 四、目录约定

下面是推荐目录结构：

```text
/home/captaincc/hermes/
├── runtime/
│   ├── shared_materials.db
│   ├── db.py
│   └── config.py
├── scripts/
│   ├── fetch_and_write.py
│   ├── generate_short_report.py
│   ├── sync_to_bitable.py
│   ├── check_fetch.py
│   ├── check_report.py
│   └── ...
├── logs/
│   ├── fetch.log
│   ├── report.log
│   ├── check_fetch.log
│   ├── check_report.log
│   └── *.result.json
└── .hermes/
    └── hermes_env.sh
```

如果你的 Hermes 安装目录不同，按你的实际路径替换，但结构建议保持一致。

## 五、主库设计

### 1. SQLite 是唯一 source of truth

不要把飞书多维表当主链路。

主链路必须是：

- 抓取 → SQLite
- Hermes 快报 → 从 SQLite 读
- 多维表 → 只是观察面

### 2. 主表建议字段

`materials` 表建议至少包含：

#### 基础字段

- `id`
- `title`
- `url`
- `platform`
- `published_at`
- `fetched_at`
- `summary_raw`

#### 标签字段

- `category`
- `content_type`
- `ai_relevance`
- `source_tier`
- `quality_score`
- `fingerprint`
- `event_key`
- `ingest_version`
- `last_scored_at`

#### Hermes 状态字段

- `hermes_status` (`pending / used / ignored`)
- `hermes_selected_at`
- `hermes_sent_at`

#### OpenClaw 状态字段（为共享素材池预留）

- `openclaw_status`
- `openclaw_selected_at`
- `openclaw_doc_id`
- `openclaw_doc_url`

#### 审计字段

- `created_at`
- `updated_at`
- `sync_to_bitable_at`
- `sync_status`
- `error_note`

### 3. 时间字段语义

必须严格区分：

- `published_at`：原文发布时间
- `fetched_at`：被系统抓到/进入素材池的时间

Hermes 快报时间窗应基于：

- `fetched_at`

不要再拿 `published_at` 做 Hermes 快报窗口筛选。

## 六、配置项

建议把所有敏感和运行参数集中在 `runtime/config.py`，业务脚本统一从 `config.py` 读。

### 建议 env 变量

```bash
HERMES_DB=/home/captaincc/hermes/runtime/shared_materials.db

HERMES_BITABLE_BASE_TOKEN=PAuubav0tajgnasG1ztcQQZ3n2g
HERMES_BITABLE_TABLE=tblQIB2KGJJj8LRq
HERMES_BITABLE_SYNC=true

HERMES_SEND_ENABLED=true

HERMES_MMX_MODEL=MiniMax-M2.7-highspeed
HERMES_MMX_PROVIDER=minimax-cn
```

### 说明

- `HERMES_BITABLE_BASE_TOKEN`：飞书多维表 app token
- `HERMES_BITABLE_TABLE`：当前正式观察面表 ID
- `HERMES_BITABLE_SYNC`：是否启用观察面同步
- `HERMES_SEND_ENABLED`：是否实际发送快报
- `HERMES_MMX_MODEL`：Hermes 使用的 MiniMax 模型
- `HERMES_MMX_PROVIDER`：建议显式使用 `minimax-cn`

### 环境注入建议

建议将配置放在：

```bash
~/.hermes/hermes_env.sh
```

由守护脚本或 cron 启动时 `source` 注入。

## 七、抓取链路

### 1. 抓取脚本职责

`fetch_and_write.py` 负责：

- 抓取 RSS / 博客 / 社区源
- 抽取标题、链接、时间、正文或原始摘要
- 计算 `fingerprint`
- 判定 `category / content_type / ai_relevance / source_tier`
- 计算 `quality_score`
- 写入 SQLite
- 可选：同步到飞书观察面

### 2. 基本原则

- 标题、链接、时间、来源必须保真
- 不允许 LLM 参与抓取层字段生成
- 入库成功以 SQLite 为准
- 多维表同步失败只能 warning，不得阻塞主链路

### 3. 内容摘要建议

不要在抓取层依赖 LLM 摘要。

如果需要给下游辅助判断，可增加一个脚本字段，例如：

- `extractive_summary`
- 或 `raw_excerpt_200`

推荐优先用：

- TextRank 提取式摘要
- 若效果不稳，回退到正文前 `180~220` 字原文摘录

但这个字段只是辅助判断字段，不是事实主键。

## 八、观察面（飞书多维表）

### 1. 观察面角色

飞书多维表不是主链路数据库，它只是：

- SQLite 的可视化同步视图

也就是：

- SQLite = 主库
- Feishu Bitable = 观察面

### 2. 推荐单独使用“干净观察面表”

不要在历史脏表上继续修补。

推荐新建一张干净表，例如：

- `观察面_干净版`

只同步当前系统记录，不再往旧表写。

### 3. 推荐字段

观察面建议至少有这些字段：

- 标题
- 链接
- 平台
- 发布时间
- 原文发布时间
- 原文摘要
- 分类
- AI相关度
- 来源层级
- quality_score
- content_type
- 去重指纹

不一定需要放：

- `Hermes状态`
- `OpenClaw状态`

这些是内部状态，观察面不一定要暴露给使用者。

### 4. 同步脚本

`sync_to_bitable.py` 负责：

- 将 SQLite 记录同步到当前观察面表
- 只作为旁路同步

缺失配置时，脚本应：

- 输出 warning
- `sys.exit(0)`

不能影响主链路。

## 九、快报生成链路

### 1. Hermes 不要让 LLM 输出完整自由日报

旧做法的问题是：

- LLM 自由改写标题
- 大量幻觉
- 标题/链接与 DB 脱钩

因此 Hermes 必须采用新结构：

### 2. 推荐的新结构

```text
脚本 → SQLite 候选（含 RID）
     → LLM 只输出选中的 RID + 摘要/分析
     → 程序根据 RID 回填真实标题/链接/来源/时间
     → 生成最终快报
```

也就是说：

- LLM 不再直接写标题
- LLM 不再直接写链接
- 最终事实字段 100% 来自数据库

### 3. 输出协议建议

输入给 LLM 的候选，用稳定编号，例如：

```text
RID_001 | 标题 | src:IT之家 | qs:72 | 摘要:……
RID_002 | 标题 | src:GitHub | qs:81 | 摘要:……
```

LLM 输出格式建议：

```text
## SELECTED
🚀 技术突破: RID_001, RID_003
🛠️ 工具/教程: RID_005
🏢 企业动态: RID_002

## ANALYSIS
🚀 技术突破：……
🛠️ 工具/教程：……
🏢 企业动态：……

## SUMMARIES
RID_001: …
RID_003: …
RID_005: …
RID_002: …
```

然后由程序：

- 解析 RID
- 回填真实标题、链接、来源、时间
- 拼成最终快报

### 4. 必须加发送前白名单校验

发送前必须检查：

- 最终条目是否都来自当前候选池
- 标题是否都能在 DB 候选中命中

任何不在候选池中的条目：

- 整轮失败
- 不发送

## 十、MiniMax 调用方式

### 1. 不要再走错误的 Anthropic 兼容旧链路

已经验证过，旧链路可能走到：

```text
/anthropic/v1/messages
```

与当前套餐/通道不匹配。

### 2. 建议显式使用官方支持的模型

Hermes 侧建议显式用：

```bash
hermes chat -Q --max-turns 1 \
  --provider minimax-cn \
  -m MiniMax-M2.7-highspeed \
  -q "<prompt>"
```

默认模型建议：

- `MiniMax-M2.7-highspeed`

### 3. 不建议让旧 minimax CLI 黑箱决定调用通道

因为：

- 通道不透明
- 计费/授权不透明
- 排障成本高

## 十一、调度（cron）

当前推荐 crontab：

```cron
# AI 日报：抓取入库 (7:30 / 11:30 / 15:30)
30 7,11,15 * * * cd /home/captaincc/hermes/scripts && python3 -u fetch_and_write.py >> /home/captaincc/hermes/logs/fetch.log 2>&1

# AI 日报：生成发送 (8:00 / 12:00 / 16:00)
0 8,12,16 * * * cd /home/captaincc/hermes/scripts && python3 -u generate_short_report.py >> /home/captaincc/hermes/logs/report.log 2>&1

# AI 日报：抓取健康检查
40 7,11,15 * * * python3 /home/captaincc/hermes/scripts/check_fetch.py >> /home/captaincc/hermes/logs/check_fetch.log 2>&1

# AI 日报：发送健康检查
10 8,12,16 * * * python3 /home/captaincc/hermes/scripts/check_report.py >> /home/captaincc/hermes/logs/check_report.log 2>&1
```

## 十二、监控和结果文件

### 1. 统一 result schema

建议 result 文件统一为：

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

### 2. send_ok 语义必须准确

如果 `DRY_RUN=True`：

- `send_ok = false`
- `draft_ok = true`
- `status = draft`

只有真实发送成功时：

- `send_ok = true`

不要再出现“草稿已保存但 send_ok=true”的误导状态。

## 十三、从 0 搭建的最小步骤

如果一个没有记忆的 agent 要从 0 落地，按下面顺序做：

### Step 1：准备目录和配置

- 建立目录结构
- 建立 `runtime/config.py`
- 建立 `~/.hermes/hermes_env.sh`

### Step 2：建立 SQLite 主库

- 创建 `materials` 表
- 建立必要索引
- 明确 `published_at / fetched_at` 语义

### Step 3：实现抓取链路

- 写 `fetch_and_write.py`
- 保真写入：
  - 标题
  - 链接
  - 来源
  - 时间
- 计算 `fingerprint / category / quality_score`

### Step 4：建立观察面同步

- 新建飞书干净表
- 写 `sync_to_bitable.py`
- 让 SQLite → 观察面同步成为旁路

### Step 5：实现 Hermes 快报生成

- 写 `generate_short_report.py`
- 采用 RID 选择 + 程序 assemble 回填结构
- 禁止 LLM 改写标题
- 加发送前白名单校验

### Step 6：接 MiniMax 正式调用链

- 显式使用 `minimax-cn + MiniMax-M2.7-highspeed`
- 做最小 prompt 调用验证

### Step 7：接 cron

- 挂抓取
- 挂发送
- 挂健康检查

### Step 8：先 DRY_RUN

- 先生成草稿
- 验证无幻觉
- 验证发送内容与 DB 一致
- 最后再恢复自动发送

## 十四、判断系统是否可用的验收标准

一套可用的 Hermes 飞书日报系统，至少应满足：

1. 标题/链接/来源/时间 100% 来自 DB
2. 快报可按 cron 自动运行
3. 多维表同步失败不影响主链路
4. 发送前可拦截任何脱离候选池的幻觉输出
5. `send_ok` 只在真实发送成功时为 `true`
6. 下次运行时，历史已发送状态不会被错误污染

## 十五、一句话总结

Hermes 从 0 搭建飞书 AI 日报系统的关键，不是“让 LLM 把日报写漂亮”，而是：

**先用脚本把事实层锁死，再让大模型只做值得关注判断和摘要分析。**

只有这样，这套系统才会真正可落地、可维护、可复现。
