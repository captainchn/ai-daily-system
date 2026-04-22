# 更新日志

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/zh-CN/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/lang/zh-CN/).

## [3.0.2] - 2026-04-22

### 新增

- 公开版文档和方法论发布
- 最小可运行 scaffold
- Hermes + OpenClaw 双层输出架构文档
- "脚本优先"架构原则文档
- 信源配置模板

### 架构特点

- SQLite 作为唯一主库
- 脚本负责事实层（标题、链接、来源、时间、去重）
- LLM 仅负责判断、摘要、分析
- 飞书作为观察面展示层
