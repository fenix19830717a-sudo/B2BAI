# B2BAI 开发详细任务拆解

> 版本: v1.0  
> 更新: 2026-03-05  
> 每个模块独立派单，负责人明确

---

## Phase 1: Core 基础架构 (2周)

目标: 搭建 B2BAI Core，实现 Provider/Adapter/Plugin 三大抽象

### M1: Core 引擎 (负责人: CTO)

| 任务ID | 任务描述 | 验收标准 | 预估工时 |
|--------|----------|----------|----------|
| M1-1 | Core 类骨架实现 | `B2BAICore` 类存在，含 `providers/adapters/managers/plugins` 四个注册表 | 4h |
| M1-2 | 插件生命周期管理 | `load()`/`unload()`/`reload()` 方法可用，插件热插拔正常 | 4h |
| M1-3 | 配置系统 | 支持 `config.yaml` 加载，环境变量覆盖，配置热更新 | 4h |
| M1-4 | 日志系统 | 结构化日志 (pino/winston)，支持分级和文件轮转 | 3h |
| M1-5 | 错误处理 | 全局错误捕获，插件隔离（单个插件崩溃不影响 Core） | 3h |
| M1-6 | Health Check | `/health` 端点返回各组件状态 | 2h |

**输出路径**: `/opt/b2bai/core/`  
**依赖**: 无  
**阻塞**: M2, M3, M4

---

### M2: LLM Provider 模块 (负责人: Builder)

| 任务ID | 任务描述 | 验收标准 | 预估工时 |
|--------|----------|----------|----------|
| M2-1 | LLM Provider 接口定义 | `LLMProvider` 接口符合 `MODULAR_SPEC.md` 2.1 节 | 2h |
| M2-2 | Kimi Provider 实现 | 支持 `complete()` + `health()`，API 调用正常 | 4h |
| M2-3 | Qwen Provider 实现 | 支持 `complete()` + `health()`，API 调用正常 | 4h |
| M2-4 | Provider 自动切换 | 主 Provider 失败时自动 fallback 到备选 | 3h |
| M2-5 | 流式响应支持 | `stream()` 方法可用，WebSocket 推送正常 | 4h |
| M2-6 | Token 用量统计 | 每次调用记录 `input/output/total` tokens | 2h |

**输出路径**: `/opt/b2bai/core/providers/`  
**依赖**: M1  
**阻塞**: M5, M6, M7

---

### M3: Channel Adapter 模块 (负责人: CTO)

| 任务ID | 任务描述 | 验收标准 | 预估工时 |
|--------|----------|----------|----------|
| M3-1 | Channel Adapter 接口定义 | `ChannelAdapter` 接口符合 `MODULAR_SPEC.md` 3.1 节 | 2h |
| M3-2 | HTTP Adapter 实现 | REST API + WebSocket 双协议支持 | 6h |
| M3-3 | 消息路由系统 | 根据 `message.channel` 路由到对应 Handler | 3h |
| M3-4 | 消息格式标准化 | 入站/出站消息统一 Schema 验证 | 3h |
| M3-5 | 会话状态绑定 | 消息自动关联 `sessionId`，支持多轮对话 | 3h |
| M3-6 | 预留 Slack/Telegram 接口 | 接口预留，文档说明扩展方式 | 2h |

**输出路径**: `/opt/b2bai/core/adapters/`  
**依赖**: M1  
**阻塞**: M5, M6, M7

---

### M4: State Manager 模块 (负责人: Builder)

| 任务ID | 任务描述 | 验收标准 | 预估工时 |
|--------|----------|----------|----------|
| M4-1 | State Manager 接口定义 | `StateManager` 接口符合 `MODULAR_SPEC.md` 4.1 节 | 2h |
| M4-2 | SQLite Adapter 实现 | `get/set/delete/keys` 方法可用，支持 TTL | 4h |
| M4-3 | Memory Adapter 实现 | 内存存储，用于测试/开发环境 | 2h |
| M4-4 | 数据迁移工具 | Schema 版本管理，自动升级脚本 | 3h |
| M4-5 | 会话持久化 | 聊天记录自动保存，支持历史查询 | 3h |

**输出路径**: `/opt/b2bai/core/managers/`  
**依赖**: M1  
**阻塞**: M5, M6, M7

---

## Phase 2: 核心插件开发 (3周)

目标: 实现 3 个业务插件，覆盖服务器运维、交易管理、社媒发布

### M5: server-ops 插件 (负责人: CTO)

| 任务ID | 任务描述 | 验收标准 | 预估工时 |
|--------|----------|----------|----------|
| M5-1 | 插件脚手架 | 符合 Plugin 接口规范，`manifest.json` 完整 | 2h |
| M5-2 | 服务器状态监控 | CPU/内存/磁盘/网络指标采集，数据库存储 | 4h |
| M5-3 | 监控 Dashboard API | `/api/server/metrics` 返回时序数据 | 3h |
| M5-4 | SSH 终端接口 | WebSocket 实现交互式 SSH，支持命令执行 | 6h |
| M5-5 | 告警规则引擎 | 阈值配置，超限时自动告警到 Slack | 4h |
| M5-6 | Gemini 故障诊断 | 异常时自动调用 Gemini 分析日志并给出建议 | 4h |
| M5-7 | 插件 UI 面板 | 前端页面展示监控图表和 SSH 终端 | 6h |

**输出路径**: `/opt/b2bai/plugins/server-ops/`  
**依赖**: M1, M2, M3, M4  
**阻塞**: 无

---

### M6: trading-bot 插件 (负责人: Builder)

| 任务ID | 任务描述 | 验收标准 | 预估工时 |
|--------|----------|----------|----------|
| M6-1 | 插件脚手架 | 符合 Plugin 接口规范，`manifest.json` 完整 | 2h |
| M6-2 | 交易数据同步 | 从 Polymarket API 拉取交易历史，本地存储 | 4h |
| M6-3 | 钱包状态监控 | 实时余额、盈亏统计、收益率计算 | 3h |
| M6-4 | 策略参数面板 | Web UI 调整阈值，实时生效 | 4h |
| M6-5 | 交易信号模拟 | 模拟交易功能，验证策略效果 | 4h |
| M6-6 | Gemini 策略优化 | 根据历史数据调用 Gemini 给出策略建议 | 4h |
| M6-7 | 可视化图表 | 收益曲线、胜率统计、风险指标图表 | 5h |

**输出路径**: `/opt/b2bai/plugins/trading-bot/`  
**依赖**: M1, M2, M3, M4  
**阻塞**: 无

---

### M7: social-poster 插件 (负责人: KimiClaw)

| 任务ID | 任务描述 | 验收标准 | 预估工时 |
|--------|----------|----------|----------|
| M7-1 | 插件脚手架 | 符合 Plugin 接口规范，`manifest.json` 完整 | 2h |
| M7-2 | 社媒账号管理 | 支持 Twitter/X, LinkedIn 账号配置 | 3h |
| M7-3 | 内容模板系统 | 模板引擎，变量替换，多语言支持 | 4h |
| M7-4 | Gemini 文案生成 | 调用 LLM 根据主题生成社媒文案 | 3h |
| M7-5 | 定时发布调度 | Cron 表达式支持，任务队列持久化 | 4h |
| M7-6 | 发布状态追踪 | 发布成功/失败记录，支持重试 | 3h |
| M7-7 | 发布管理面板 | Web UI 管理内容、查看发布日历 | 5h |

**输出路径**: `/opt/b2bai/plugins/social-poster/`  
**依赖**: M1, M2, M3, M4  
**阻塞**: 无

---

## Phase 3: 插件市场 & 集成优化 (2周)

目标: 实现插件市场，完成 Gemini 深度集成，接入现有系统

### M8: 插件市场 (负责人: CTO)

| 任务ID | 任务描述 | 验收标准 | 预估工时 |
|--------|----------|----------|----------|
| M8-1 | 市场索引服务 | `/api/market/plugins` 返回可用插件列表 | 3h |
| M8-2 | 插件元数据规范 | `manifest.json` Schema 定义，验证工具 | 3h |
| M8-3 | 一键安装 API | `POST /api/plugins/install` 下载+解压+注册 | 4h |
| M8-4 | 一键卸载 API | `POST /api/plugins/uninstall` 清理+注销 | 3h |
| M8-5 | 插件配置管理 | 每个插件独立配置页面，支持热更新 | 4h |
| M8-6 | 市场前端页面 | 插件浏览/搜索/安装/配置 UI | 6h |

**输出路径**: `/opt/b2bai/market/`  
**依赖**: M1  
**阻塞**: 无

---

### M9: Gemini Pro 深度集成 (负责人: Builder)

| 任务ID | 任务描述 | 验收标准 | 预估工时 |
|--------|----------|----------|----------|
| M9-1 | Gemini Provider 实现 | 支持 `complete()` + `stream()` | 3h |
| M9-2 | 代码生成助手 | 自然语言描述 → 生成代码脚手架 | 4h |
| M9-3 | 代码审查助手 | 自动 Review PR，给出改进建议 | 4h |
| M9-4 | 测试用例生成 | 根据函数签名生成单元测试 | 3h |
| M9-5 | 智能客服集成 | 知识库问答，工单自动分类 | 4h |
| M9-6 | 多语言翻译 | 内容自动翻译，支持 10+ 语言 | 3h |

**输出路径**: 分散到各模块  
**依赖**: M2 (LLM Provider)  
**阻塞**: 无

---

### M10: 系统集成 (负责人: CTO)

| 任务ID | 任务描述 | 验收标准 | 预估工时 |
|--------|----------|----------|----------|
| M10-1 | STD 独立站 API 接入 | 产品数据同步到 B2BAI | 4h |
| M10-2 | 交易机器人数据接入 | 实时数据推送到 trading-bot 插件 | 4h |
| M10-3 | 统一认证系统 | JWT/OAuth2 集成，单点登录 | 4h |
| M10-4 | 部署脚本 | Docker Compose + Nginx 配置 | 4h |
| M10-5 | 文档完善 | API 文档、部署指南、开发教程 | 4h |

**输出路径**: `/opt/b2bai/integrations/`  
**依赖**: M5, M6, M7, M8  
**阻塞**: 无

---

## 任务依赖图

```
Phase 1 (基础架构)
├── M1: Core 引擎 ──┬── M2: LLM Provider ───┐
│                   ├── M3: Channel Adapter ─┼── Phase 2 (业务插件)
│                   └── M4: State Manager ───┘      ├── M5: server-ops
│                                                   ├── M6: trading-bot
│                                                   └── M7: social-poster
│
Phase 3 (市场+集成)
├── M8: 插件市场 ── M10: 系统集成
├── M9: Gemini 集成 ───┘
```

---

## 排期建议

| 周次 | 任务 | 负责人 |
|------|------|--------|
| W1 | M1, M2 | CTO, Builder |
| W2 | M3, M4, M5-1~3 | CTO, Builder |
| W3 | M5-4~7, M6-1~4 | CTO, Builder |
| W4 | M6-5~7, M7 | Builder, KimiClaw |
| W5 | M8, M9 | CTO, Builder |
| W6 | M10, 集成测试 | CTO |
| W7 | Bug 修复, 文档, 上线 | All |

**总计**: 7 周 (含 1 周缓冲)

---

## 派单建议

1. **立即派单 M1 (Core 引擎)** → CTO
   - 阻塞所有后续任务，优先级最高

2. **W1 并行派单 M2 (LLM Provider)** → Builder
   - 与 M1 可并行，只需接口约定

3. **M1/M2 完成后** 并行派单 M3, M4 → CTO, Builder

4. **Phase 1 完成后** 并行派单 M5, M6, M7
   - 三个插件可完全并行开发

5. **Phase 2 进行中** 派单 M8, M9 → CTO, Builder
   - 与插件开发部分重叠

6. **全部完成后** 派单 M10 → CTO
   - 集成收尾

---

*文档路径: `/root/.openclaw/workspace-main/docs/B2BAI_TASK_BREAKDOWN.md`*
