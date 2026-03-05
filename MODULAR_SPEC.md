# B2BAI 模块化接口规范

> 版本: 1.0.0  
> 设计参考: OpenClaw Provider Pattern

## 目录
- [1. 架构概览](#1-架构概览)
- [2. LLM Provider](#2-llm-provider)
- [3. Channel Adapter](#3-channel-adapter)
- [4. State Manager](#4-state-manager)
- [5. 使用示例](#5-使用示例)

---

## 1. 架构概览

### 1.1 模块关系图

```
                    Business Plugin
                          │
          ┌───────────────┼───────────────┐
          │               │               │
          ▼               ▼               ▼
   ┌────────────┐  ┌────────────┐  ┌────────────┐
   │  LLM       │  │  Channel   │  │   State    │
   │  Provider  │  │  Adapter   │  │  Manager   │
   │  (抽象)     │  │  (抽象)     │  │  (抽象)     │
   └─────┬──────┘  └─────┬──────┘  └─────┬──────┘
         │               │               │
    ┌────┴────┐     ┌────┴────┐     ┌────┴────┐
    │         │     │         │     │         │
┌───▼───┐ ┌───▼───┐ │  ┌──────▼──┐  │ ┌───────▼───┐
│ Kimi  │ │ Qwen  │ │  │   HTTP  │  │ │  SQLite   │
│Provider│ │Provider│ │  │Adapter  │  │ │  Adapter  │
└───────┘ └───────┘ │  └─────────┘  │ └───────────┘
                    │               │
               ┌────┴────┐     ┌────┴────┐
               │ WebSocket│     │  Redis  │
               │ Adapter │     │ Adapter │
               └─────────┘     └─────────┘
```

### 1.2 核心设计原则

1. **依赖倒置**: 高层模块（业务插件）依赖抽象接口，不依赖具体实现
2. **开闭原则**: 新增 Provider/Adapter 无需修改 Core 代码
3. **单一职责**: 每个模块只做一件事，接口精简

---

## 2. LLM Provider

### 2.1 接口定义

```typescript
// llm-provider.interface.ts

interface LLMProvider {
  // 标识
  readonly name: string;
  readonly version: string;
  
  // 初始化
  init(config: LLMConfig): Promise<void>;
  
  // 核心调用
  complete(request: CompletionRequest): Promise<CompletionResponse>;
  
  // 流式调用（可选）
  stream?(request: CompletionRequest): AsyncIterable<StreamChunk>;
  
  // 健康检查
  health(): Promise<HealthStatus>;
}

interface LLMConfig {
  apiKey: string;
  baseUrl?: string;
  model: string;
  timeout?: number;
  maxRetries?: number;
}

interface CompletionRequest {
  messages: Message[];
  temperature?: number;
  maxTokens?: number;
  tools?: Tool[];
}

interface CompletionResponse {
  content: string;
  toolCalls?: ToolCall[];
  usage: TokenUsage;
  model: string;
  finishReason: string;
}
```

### 2.2 内置实现

| Provider | 支持模型 | 状态 |
|----------|----------|------|
| `kimi-provider` | kimi-coding/k2p5, kimi-k2.5 | ✅ 默认 |
| `qwen-provider` | qwen-coder-plus, qwen-max | ✅ 备选 |
| `openai-provider` | gpt-4, gpt-3.5 | 🟡 预留 |

### 2.3 实现示例

```javascript
// providers/kimi-provider.js
class KimiProvider {
  name = 'kimi';
  version = '1.0.0';
  
  async init(config) {
    this.apiKey = config.apiKey;
    this.baseUrl = config.baseUrl || 'https://api.moonshot.cn';
    this.model = config.model || 'kimi-coding/k2p5';
  }
  
  async complete(request) {
    const response = await fetch(`${this.baseUrl}/v1/chat/completions`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.apiKey}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        model: this.model,
        messages: request.messages,
        temperature: request.temperature || 0.7,
        max_tokens: request.maxTokens
      })
    });
    
    const data = await response.json();
    return {
      content: data.choices[0].message.content,
      usage: data.usage,
      model: data.model,
      finishReason: data.choices[0].finish_reason
    };
  }
  
  async health() {
    try {
      await this.complete({ messages: [{ role: 'user', content: 'hi' }], maxTokens: 1 });
      return { status: 'ok', latency: Date.now() };
    } catch (e) {
      return { status: 'error', error: e.message };
    }
  }
}

module.exports = KimiProvider;
```

### 2.4 插件使用方式

```javascript
// 在业务插件中使用
async function analyzeProduct(core, productData) {
  // 获取默认 LLM Provider
  const llm = core.providers.get('llm');
  
  const response = await llm.complete({
    messages: [
      { role: 'system', content: '你是一个产品分析专家' },
      { role: 'user', content: `分析以下产品：${JSON.stringify(productData)}` }
    ],
    temperature: 0.3
  });
  
  return response.content;
}

// 切换 Provider（运行时）
const qwen = core.providers.get('llm', 'qwen');
```

---

## 3. Channel Adapter

### 3.1 接口定义

```typescript
// channel-adapter.interface.ts

interface ChannelAdapter {
  readonly name: string;
  readonly type: 'incoming' | 'outgoing' | 'bidirectional';
  
  // 初始化
  init(config: ChannelConfig): Promise<void>;
  
  // 接收消息（incoming/bidirectional）
  onMessage?(handler: MessageHandler): void;
  
  // 发送消息（outgoing/bidirectional）
  send?(message: OutboundMessage): Promise<void>;
  
  // 回复消息（incoming/bidirectional）
  reply?(messageId: string, content: string): Promise<void>;
  
  // 启动/停止
  start(): Promise<void>;
  stop(): Promise<void>;
}

type MessageHandler = (message: InboundMessage) => Promise<void>;

interface InboundMessage {
  id: string;
  channel: string;        // 'web', 'slack', etc.
  sender: string;
  content: string;
  timestamp: number;
  metadata?: Record<string, any>;
}

interface OutboundMessage {
  channel: string;
  target: string;         // 用户ID/频道ID
  content: string;
  options?: {
    replyTo?: string;     // 回复某条消息
    format?: 'text' | 'markdown' | 'html';
  };
}
```

### 3.2 内置实现

| Adapter | 类型 | 状态 | 说明 |
|---------|------|------|------|
| `http-adapter` | bidirectional | ✅ 默认 | Web API + WebSocket |
| `slack-adapter` | bidirectional | 🟡 预留 | Slack Bot |
| `telegram-adapter` | bidirectional | 🟡 预留 | Telegram Bot |

### 3.3 HTTP Adapter 实现

```javascript
// adapters/http-adapter.js
const http = require('http');
const WebSocket = require('ws');

class HttpAdapter {
  name = 'http';
  type = 'bidirectional';
  
  async init(config) {
    this.port = config.port || 3000;
    this.messageHandler = null;
  }
  
  onMessage(handler) {
    this.messageHandler = handler;
  }
  
  async start() {
    // HTTP API
    this.server = http.createServer((req, res) => {
      if (req.url === '/api/message' && req.method === 'POST') {
        let body = '';
        req.on('data', chunk => body += chunk);
        req.on('end', async () => {
          const data = JSON.parse(body);
          const message = {
            id: data.id || crypto.randomUUID(),
            channel: 'web',
            sender: data.sender || 'anonymous',
            content: data.content,
            timestamp: Date.now(),
            metadata: { ip: req.socket.remoteAddress }
          };
          
          // 调用业务处理
          if (this.messageHandler) {
            await this.messageHandler(message);
          }
          
          res.writeHead(200, { 'Content-Type': 'application/json' });
          res.end(JSON.stringify({ received: true, id: message.id }));
        });
      }
    });
    
    // WebSocket 支持
    this.wss = new WebSocket.Server({ server: this.server });
    this.wss.on('connection', (ws) => {
      ws.on('message', async (data) => {
        const msg = JSON.parse(data);
        if (this.messageHandler) {
          await this.messageHandler({
            id: msg.id,
            channel: 'web',
            sender: msg.sender,
            content: msg.content,
            timestamp: Date.now(),
            metadata: { ws: true }
          });
        }
      });
    });
    
    this.server.listen(this.port);
  }
  
  async send(message) {
    // WebSocket 广播
    this.wss.clients.forEach(client => {
      if (client.readyState === WebSocket.OPEN) {
        client.send(JSON.stringify(message));
      }
    });
  }
  
  async stop() {
    this.wss.close();
    this.server.close();
  }
}

module.exports = HttpAdapter;
```

### 3.4 预留扩展（Slack/Telegram）

```javascript
// adapters/slack-adapter.js (预留接口)
class SlackAdapter {
  name = 'slack';
  type = 'bidirectional';
  
  async init(config) {
    this.token = config.token;
    this.signingSecret = config.signingSecret;
    // ... Slack SDK 初始化
  }
  
  onMessage(handler) {
    // 注册 Slack Events API 处理器
    // this.app.message(async ({ message, say }) => {
    //   await handler(transformSlackMessage(message));
    // });
  }
  
  async send(message) {
    // 调用 Slack Web API
    // await this.client.chat.postMessage({
    //   channel: message.target,
    //   text: message.content
    // });
  }
}
```

---

## 4. State Manager

### 4.1 接口定义

```typescript
// state-manager.interface.ts

interface StateManager {
  readonly name: string;
  
  init(config: StateConfig): Promise<void>;
  
  // 基础 CRUD
  get(key: string): Promise<any>;
  set(key: string, value: any, ttl?: number): Promise<void>;
  delete(key: string): Promise<void>;
  
  // 原子操作
  increment(key: string, amount?: number): Promise<number>;
  
  // 查询
  keys(pattern?: string): Promise<string[]>;
  
  // 事务（可选）
  transaction?(operations: Operation[]): Promise<void>;
}
```

### 4.2 内置实现

| Adapter | 适用场景 | 状态 |
|---------|----------|------|
| `sqlite-adapter` | 单机/轻量 | ✅ 默认 |
| `redis-adapter` | 分布式/缓存 | 🟡 预留 |
| `memory-adapter` | 测试/开发 | ✅ 内置 |

---

## 5. 使用示例

### 5.1 完整业务插件示例

```javascript
// plugins/chat-plugin/index.js
class ChatPlugin {
  name = 'chat-plugin';
  version = '1.0.0';
  
  async init(core, config) {
    this.core = core;
    this.config = config;
    
    // 获取模块
    this.llm = core.providers.get('llm');
    this.channel = core.adapters.get('channel', 'http');
    this.state = core.managers.get('state');
    
    // 注册消息处理器
    this.channel.onMessage(async (msg) => {
      await this.handleMessage(msg);
    });
  }
  
  async handleMessage(message) {
    // 1. 读取会话历史
    const historyKey = `chat:${message.sender}:history`;
    let history = await this.state.get(historyKey) || [];
    
    // 2. 调用 LLM
    const response = await this.llm.complete({
      messages: [
        { role: 'system', content: '你是 B2B 助手' },
        ...history.slice(-10),
        { role: 'user', content: message.content }
      ]
    });
    
    // 3. 更新历史
    history.push(
      { role: 'user', content: message.content },
      { role: 'assistant', content: response.content }
    );
    await this.state.set(historyKey, history);
    
    // 4. 回复
    await this.channel.send({
      channel: message.channel,
      target: message.sender,
      content: response.content,
      options: { replyTo: message.id }
    });
  }
  
  async getState() {
    return {
      name: this.name,
      llmProvider: this.llm.name,
      channelAdapter: this.channel.name
    };
  }
  
  async destroy() {
    // 清理
  }
}

module.exports = ChatPlugin;
```

### 5.2 Core 初始化

```javascript
// core/index.js
const B2BAICore = require('./b2bai-core');

async function bootstrap() {
  const core = new B2BAICore();
  
  // 1. 注册 Providers
  await core.providers.register('llm', 'kimi', new KimiProvider(), {
    apiKey: process.env.KIMI_API_KEY,
    model: 'kimi-coding/k2p5'
  });
  
  // 2. 注册 Adapters
  await core.adapters.register('channel', 'http', new HttpAdapter(), {
    port: 3000
  });
  
  // 3. 注册 State
  await core.managers.register('state', new SQLiteAdapter(), {
    path: '/data/state.db'
  });
  
  // 4. 加载业务插件
  await core.plugins.load('chat-plugin', {
    // 插件配置
  });
  
  // 5. 启动
  await core.start();
}

bootstrap();
```

---

## 附录 A: 接口变更日志

| 版本 | 日期 | 变更 |
|------|------|------|
| 1.0.0 | 2026-03-05 | 初始版本 |

---

**参考文档**: OpenClaw Provider Pattern  
**适用范围**: B2BAI Core ≥ 1.0.0
