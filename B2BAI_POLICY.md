# B2BAI 平台国策文档

> 版本: 2026-03-05  
> 适用范围: 阿里云 + 日本云双部署

## 一、核心原则

### 1.1 轻量优先
- **B2BAI Core** 体积控制在 **2MB 以内**
- 禁止引入重型依赖（如完整React/Vue框架）
- 优先使用原生 API + 轻量库

### 1.2 统一架构
```
所有业务应用基于同一 B2BAI Core
├── 独立站 → B2BAI + site-plugin
├── 交易机器人 → B2BAI + trade-plugin
└── 数据抓取 → B2BAI + crawl-plugin
```

### 1.3 模块化设计（参考 OpenClaw）

B2BAI 采用分层模块化架构，核心层与业务层解耦：

```
┌─────────────────────────────────────────────────────────┐
│                    Business Plugins                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐  │
│  │  site    │  │  trade   │  │  crawl   │  │ future  │  │
│  │  -plugin │  │  -plugin │  │  -plugin │  │   ...   │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬────┘  │
└───────┼─────────────┼─────────────┼─────────────┼───────┘
        │             │             │             │
        └─────────────┴──────┬──────┴─────────────┘
                             │
┌────────────────────────────▼────────────────────────────┐
│                    B2BAI Core Layer                      │
│  ┌──────────────────────────────────────────────────┐   │
│  │              Provider Registry                    │   │
│  │   ┌──────────┐  ┌──────────┐  ┌──────────┐      │   │
│  │   │   LLM    │  │ Channel  │  │  State   │      │   │
│  │   │ Provider │  │ Adapter  │  │ Manager  │      │   │
│  │   └──────────┘  └──────────┘  └──────────┘      │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

**设计原则**:
- **Provider 可插拔**: LLM、Channel、Storage 等通过统一接口注册
- **接口抽象**: 业务插件只依赖抽象接口，不依赖具体实现
- **水平扩展**: 新增渠道（如 Slack、Telegram）只需实现 Channel Adapter

### 1.4 双云协同
| 节点 | 职责 | 数据流向 |
|------|------|----------|
| **阿里云** (北京) | 状态控制器、轻量API、定时任务 | 发送指令 |
| **日本云** (4U8G) | 计算密集型、实时交易、AI服务 | 接收指令 + 上报状态 |

## 二、技术规范

### 2.1 插件接口
```javascript
// 插件必须实现的接口
module.exports = {
  name: 'plugin-name',
  version: '1.0.0',
  
  // 初始化
  async init(core, config) {},
  
  // 状态上报
  async getState() { return {}; },
  
  // 接收指令
  async onCommand(cmd, payload) {},
  
  // 清理
  async destroy() {}
};
```

### 2.2 状态同步协议
- **频率**: 阿里云 → 日本云 每30秒心跳
- **内容**: 任务队列、资源使用率、关键指标
- **格式**: JSON，压缩后传输

### 2.3 资源限制
```bash
# 所有 Node.js 进程必须设置
ulimit -v 1048576  # 虚拟内存 1GB
ulimit -m 524288   # 物理内存 512MB
```

## 三、部署规范

### 3.1 阿里云部署清单
```
/opt/b2bai-controller/
├── core/              # B2BAI Core (共享)
├── plugins/
│   └── site-plugin/   # 独立站插件
├── config/
│   └── sync.json      # 双云同步配置
└── logs/
```

### 3.2 日本云部署清单
```
/opt/b2bai/
├── core/              # B2BAI Core (共享)
├── plugins/
│   ├── trade-plugin/  # 交易机器人
│   ├── crawl-plugin/  # 数据抓取
│   └── ai-plugin/     # AI服务
└── logs/
```

### 3.3 Nginx 路由
```nginx
# 阿里云
location /b2bai/ {
    proxy_pass http://localhost:3000/;
}

# 日本云
location /api/b2bai/ {
    proxy_pass http://localhost:3001/;
}
```

## 四、开发纪律

### 4.1 Git 规范
- **主仓库**: `B2BAI-Core` (单一core)
- **插件仓库**: `B2BAI-Plugins/*` (独立插件)
- **分支策略**: 
  - `main`: 稳定版
  - `develop`: 开发版
  - `plugin/*`: 插件开发

### 4.2 测试规范
- Core 变更必须通过所有插件兼容性测试
- 插件独立版本号，遵循 SemVer

### 4.3 发布流程
1. Core 版本更新 → 同步更新所有节点
2. 插件更新 → 可独立热更新
3. 双云状态同步验证通过后方可上线

## 五、监控告警

### 5.1 关键指标
| 指标 | 阈值 | 告警方式 |
|------|------|----------|
| 内存使用 | >80% | Slack #ops |
| 双云延迟 | >5s | Slack #ops |
| 插件崩溃 | 任意 | 邮件+Slack |

### 5.2 日志规范
- 统一使用 JSON 格式
- 字段: `timestamp`, `level`, `plugin`, `message`, `meta`
- 保留 7 天，自动清理

## 六、模块化接口规范

### 6.1 为什么模块化
- **复用**: LLM层、Channel层可被未来项目直接使用
- **解耦**: 业务插件不依赖具体实现（如 Kimi vs GPT）
- **测试**: 可 Mock Provider 进行单元测试
- **扩展**: 新增渠道/模型只需实现标准接口

### 6.2 模块清单

| 模块 | 职责 | 当前实现 | 未来复用 |
|------|------|----------|----------|
| **LLM Provider** | 大模型调用统一接口 | Kimi/百炼 | ✅ 通用 |
| **Channel Adapter** | 通讯渠道抽象 | HTTP/Web | ✅ 可扩展 Slack/Discord |
| **State Manager** | 状态持久化 | SQLite/Redis | ✅ 通用 |
| **Plugin Loader** | 插件生命周期 | 自研 | ✅ 通用 |
| **Task Scheduler** | 定时任务 | node-cron | ✅ 通用 |

### 6.3 当前范围约束

> ⚠️ **当前阶段**: B2BAI 仅支持 **Web 端访问**
> 
> Channel Adapter 初始仅实现 HTTP 模式，但接口设计预留 Slack、Telegram、WhatsApp 等扩展点。

---

**最后更新**: 2026-03-05  
**负责人**: CTO
