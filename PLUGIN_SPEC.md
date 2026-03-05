# B2BAI 插件开发规范

## 一、插件结构

```
my-plugin/
├── index.js           # 主入口
├── package.json       # 元数据
├── config.json        # 默认配置
├── lib/
│   └── utils.js       # 工具函数
└── README.md          # 插件文档
```

## 二、最小示例

```javascript
// index.js
const { EventEmitter } = require('events');

class MyPlugin extends EventEmitter {
  constructor() {
    super();
    this.name = 'my-plugin';
    this.version = '1.0.0';
    this.core = null;
    this.config = {};
  }

  // 必须: 初始化
  async init(core, config) {
    this.core = core;
    this.config = { ...this.config, ...config };
    
    // 注册命令处理
    core.on('command', (cmd, payload) => {
      if (cmd.startsWith(this.name)) {
        this.handleCommand(cmd, payload);
      }
    });
    
    this.emit('ready');
  }

  // 必须: 状态上报
  async getState() {
    return {
      name: this.name,
      status: 'running',
      uptime: Date.now() - this.startTime,
      metrics: await this.collectMetrics()
    };
  }

  // 可选: 命令处理
  async handleCommand(cmd, payload) {
    switch(cmd) {
      case `${this.name}:start`:
        return this.start(payload);
      case `${this.name}:stop`:
        return this.stop(payload);
      case `${this.name}:config`:
        return this.updateConfig(payload);
    }
  }

  // 必须: 清理
  async destroy() {
    // 清理资源、关闭连接
    this.removeAllListeners();
  }

  // 自定义方法
  async start(options) { /* ... */ }
  async stop() { /* ... */ }
  async updateConfig(newConfig) { /* ... */ }
  async collectMetrics() { return {}; }
}

module.exports = MyPlugin;
```

## 三、package.json 规范

```json
{
  "name": "b2bai-plugin-myplugin",
  "version": "1.0.0",
  "description": "插件描述",
  "main": "index.js",
  "b2bai": {
    "minCoreVersion": "1.0.0",
    "maxCoreVersion": "2.0.0",
    "permissions": ["network", "filesystem"],
    "configSchema": {
      "apiKey": { "type": "string", "required": true },
      "timeout": { "type": "number", "default": 5000 }
    }
  },
  "dependencies": {
    "axios": "^1.0.0"
  }
}
```

## 四、Core API

```javascript
// 日志
this.core.logger.info('消息');
this.core.logger.warn('警告');
this.core.logger.error('错误');

// 配置
const value = this.core.config.get('key');
this.core.config.set('key', value);

// 状态存储
await this.core.state.set('myKey', data);
const data = await this.core.state.get('myKey');

// 定时任务
this.core.scheduler.add('jobName', '*/5 * * * *', callback);

// 发送指令到其他节点
await this.core.sync.send(targetNode, command, payload);

// 监听远程指令
this.core.sync.on('command', (from, cmd, payload) => {});
```

## 五、性能约束

| 指标 | 限制 | 说明 |
|------|------|------|
| 内存 | < 100MB | 插件独占内存 |
| 启动时间 | < 3s | init() 完成时间 |
| CPU | < 10% | 持续占用 |
| 依赖体积 | < 10MB | node_modules |

## 六、开发流程

```bash
# 1. 创建插件目录
mkdir -p /opt/b2bai/plugins/my-plugin
cd /opt/b2bai/plugins/my-plugin

# 2. 初始化
npm init -y

# 3. 编写代码 (index.js)
# ...

# 4. 本地测试
core.loadPlugin('./my-plugin', { test: true });

# 5. 打包发布
npm pack
# 上传至 B2BAI 插件仓库
```

## 七、插件类型模板

### 7.1 独立站插件（使用模块化接口）

```javascript
// 处理HTTP请求
this.core.adapters.get('channel', 'http').onMessage(async (req) => {
  const products = await this.fetchProducts();
  return { products };
});

// 使用 LLM 生成产品描述
const description = await this.core.providers.get('llm').complete({
  messages: [
    { role: 'system', content: '你是电商文案专家' },
    { role: 'user', content: `为以下产品生成描述：${product.name}` }
  ]
});
```

### 7.2 交易插件

```javascript
// 定时执行策略
this.core.scheduler.add('trade-scan', '*/1 * * * *', async () => {
  const signals = await this.scanMarket();
  if (signals.length > 0) {
    await this.executeTrades(signals);
  }
});
```

### 7.3 数据插件

```javascript
// 增量同步
async sync() {
  const lastSync = await this.core.managers.get('state').get('lastSync');
  const changes = await this.fetchChanges(lastSync);
  await this.saveToLocal(changes);
  await this.core.managers.get('state').set('lastSync', Date.now());
}
```

---

## 八、模块化最佳实践

### 8.1 不要直接依赖具体实现

❌ **错误做法**:
```javascript
// 直接硬编码 Kimi API
const response = await fetch('https://api.moonshot.cn/v1/...', {...});
```

✅ **正确做法**:
```javascript
// 通过 Provider 抽象层
const llm = this.core.providers.get('llm');
const response = await llm.complete({ messages: [...] });
```

### 8.2 处理 Provider 不可用

```javascript
async function safeComplete(messages) {
  const llm = this.core.providers.get('llm');
  
  // 检查健康状态
  const health = await llm.health();
  if (health.status !== 'ok') {
    // 切换到备选 Provider
    const fallback = this.core.providers.get('llm', 'qwen');
    return fallback.complete(messages);
  }
  
  return llm.complete(messages);
}
```

### 8.3 配置文件示例

```json
// config.json
{
  "providers": {
    "llm": {
      "default": "kimi",
      "kimi": {
        "apiKey": "${KIMI_API_KEY}",
        "model": "kimi-coding/k2p5"
      },
      "qwen": {
        "apiKey": "${QWEN_API_KEY}",
        "model": "qwen-coder-plus"
      }
    }
  },
  "adapters": {
    "channel": {
      "default": "http",
      "http": {
        "port": 3000
      }
    }
  },
  "managers": {
    "state": {
      "type": "sqlite",
      "path": "/data/state.db"
    }
  }
}
```

## 八、调试技巧

```javascript
// 开发模式热重载
if (process.env.B2BAI_DEV) {
  const fs = require('fs');
  fs.watchFile('./index.js', () => {
    console.log('文件变更，重新加载...');
    // Core 会自动重载
  });
}

// 本地状态查看
curl http://localhost:3000/api/plugins/my-plugin/state
```

---
**版本**: 1.0.0  
**适用 Core**: >= 1.0.0
