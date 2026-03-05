# B2BAI - B2B AI Platform

> 热拔插插件架构的B2B AI平台，统一管理服务器运维、交易机器人、社媒发布

---

## 文档索引

| 文档 | 内容 | 状态 |
|------|------|------|
| [ARCHITECTURE.md](./ARCHITECTURE.md) | 系统架构、时序图、API规范、数据模型 | ✅ v1.1 |
| [TESTING.md](./TESTING.md) | 错误处理、边界情况、测试策略 | ✅ v1.2 |
| [DEPLOYMENT.md](./DEPLOYMENT.md) | 部署指南、监控指标、故障排查 | ✅ v1.3 |
| [B2BAI_REQUIREMENTS.md](./B2BAI_REQUIREMENTS.md) | 功能需求规格 | ✅ |
| [B2BAI_TASK_BREAKDOWN.md](./B2BAI_TASK_BREAKDOWN.md) | 开发任务拆解 | ✅ |
| [B2BAI_POLICY.md](./B2BAI_POLICY.md) | 开发策略与国策 | ✅ |
| [MODULAR_SPEC.md](./MODULAR_SPEC.md) | 模块化架构规范 | ✅ |
| [PLUGIN_SPEC.md](./PLUGIN_SPEC.md) | 插件开发规范 | ✅ |

---

## 快速开始

### 开发环境

```bash
# 克隆仓库
git clone https://github.com/fenix19830717a-sudo/B2BAI.git
cd B2BAI

# 安装依赖
cd core && npm install

# 配置环境变量
cp .env.example .env
# 编辑 .env 填写 API Keys

# 启动开发服务器
npm run dev
```

### 生产部署

```bash
# 一键部署
./scripts/deploy.sh

# 或使用 Docker
docker-compose up -d
```

详细部署指南参见 [DEPLOYMENT.md](./DEPLOYMENT.md)

---

## 系统架构

```
B2BAI Platform
├── Core Layer
│   ├── ProviderRegistry (LLM多Provider管理)
│   ├── AdapterRegistry (HTTP/WebSocket)
│   ├── StateManager (SQLite/Redis)
│   └── PluginLoader (热拔插)
│
├── Plugin Layer
│   ├── tenant-plugin (多租户)
│   ├── template-plugin (网站模板)
│   ├── trade-plugin (交易机器人)
│   ├── server-ops-plugin (服务器运维)
│   └── social-poster-plugin (社媒发布)
│
└── Web Layer (管理后台)
```

---

## 核心特性

- 🔌 **热拔插插件** - 不停机安装/卸载/更新插件
- 🔄 **多Provider** - Gemini/Groq/Kimi自动故障转移
- 🏢 **多租户** - 基于子域名的租户隔离
- 📊 **监控告警** - Prometheus + Grafana集成
- 🧪 **完整测试** - 单元/集成/E2E/混沌测试
- 🚀 **一键部署** - 脚本/Docker/K8s支持

---

## API示例

```http
POST /api/v1/message
Content-Type: application/json
X-Session-Id: {sessionId}

{
  "content": "你好",
  "options": {
    "provider": "gemini",
    "stream": false
  }
}
```

完整API文档参见 [ARCHITECTURE.md#api接口规范](./ARCHITECTURE.md#四api接口规范)

---

## 插件开发

```javascript
// 插件示例
module.exports = class MyPlugin {
  manifest = {
    name: 'my-plugin',
    version: '1.0.0',
    hooks: { onMessage: true }
  };

  async onMessage(message, context) {
    // 处理消息
    return { content: '处理结果' };
  }
};
```

详细规范参见 [PLUGIN_SPEC.md](./PLUGIN_SPEC.md)

---

## 开发路线图

### Phase 1: 基础架构 (2周)
- [x] Core引擎 (M1)
- [x] LLM Provider (M2)
- [ ] Channel Adapter (M3)
- [ ] State Manager (M4)

### Phase 2: 核心插件 (3周)
- [ ] server-ops插件 (M5)
- [ ] trading-bot插件 (M6)
- [ ] social-poster插件 (M7)

### Phase 3: 市场+集成 (2周)
- [ ] 插件市场 (M8)
- [ ] Gemini深度集成 (M9)
- [ ] 系统集成 (M10)

---

## 监控

- 健康检查: `GET /health`
- 就绪检查: `GET /ready`
- 指标: `GET /metrics` (Prometheus格式)

---

## 贡献

1. Fork 仓库
2. 创建特性分支: `git checkout -b feature/xxx`
3. 提交更改: `git commit -am 'Add feature'`
4. 推送分支: `git push origin feature/xxx`
5. 创建 Pull Request

---

## License

MIT License

---

*最后更新: 2026-03-05*
