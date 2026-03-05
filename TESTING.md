# B2BAI 测试与错误处理规范 (细化版 v1.2)

> 版本: v1.2 | 更新: 2026-03-05 | 细化轮次: 2/3

---

## 一、错误处理策略

### 1.1 错误层级

```
┌─────────────────────────────────────────────────────────┐
│                    错误处理金字塔                         │
├─────────────────────────────────────────────────────────┤
│  Level 4: 全局捕获 (Global Handler)                      │
│           - 未捕获异常兜底                                │
│           - 进程保护                                      │
├─────────────────────────────────────────────────────────┤
│  Level 3: Core层 (B2BAICore)                             │
│           - 插件隔离                                      │
│           - Provider故障转移                              │
├─────────────────────────────────────────────────────────┤
│  Level 2: 组件层 (Adapter/Provider/Plugin)               │
│           - 输入验证                                      │
│           - 业务错误处理                                  │
├─────────────────────────────────────────────────────────┤
│  Level 1: 工具层 (Utils/Libs)                            │
│           - 参数校验                                      │
│           - 类型检查                                      │
└─────────────────────────────────────────────────────────┘
```

### 1.2 Core层错误处理

```javascript
class B2BAICore {
  async dispatch(message, context) {
    const plugin = this.pluginLoader.get(context.pluginName);
    
    try {
      // 插件级隔离：单个插件崩溃不影响Core
      const result = await this.runWithTimeout(
        () => plugin.onMessage(message, context),
        30000  // 30秒超时
      );
      
      return result;
    } catch (error) {
      // 记录错误但不中断Core
      this.logger.error('Plugin error:', {
        plugin: plugin.name,
        error: error.message,
        stack: error.stack
      });
      
      // 返回友好的错误消息
      return {
        error: true,
        code: 'E2001',
        message: '插件处理失败，请稍后重试',
        fallback: true  // 标记为fallback响应
      };
    }
  }
  
  // Provider故障转移
  async completeWithFallback(params) {
    const providers = this.providerRegistry.getHealthyProviders();
    
    for (const provider of providers) {
      try {
        const result = await provider.complete(params);
        return result;
      } catch (error) {
        this.logger.warn(`Provider ${provider.name} failed:`, error.message);
        provider.markUnhealthy();
      }
    }
    
    throw new Error('All providers failed');
  }
}
```

### 1.3 Provider错误处理

| 错误类型 | Gemini | Groq | Kimi | 处理策略 |
|----------|--------|------|------|----------|
| 配额耗尽 (429) | ✅ | ✅ | ✅ | 切换Provider |
| 认证失败 (401) | ✅ | ✅ | ✅ | 告警，切换Key |
| 模型不可用 (503) | ✅ | ✅ | ✅ | 降级到备用模型 |
| 请求超时 | ✅ | ✅ | ✅ | 重试1次，然后切换 |
| 内容违规 (400) | ✅ | ✅ | ✅ | 返回错误，不重试 |
| 网络错误 | ✅ | ✅ | ✅ | 重试3次，指数退避 |

```javascript
class ProviderErrorHandler {
  static handle(error, provider) {
    const status = error.status || error.code;
    
    switch (status) {
      case 429:  // Rate limit
        return {
          action: 'switch_provider',
          retry: false,
          cooldown: 60000  // 1分钟冷却
        };
        
      case 401:  // Auth failed
        return {
          action: 'rotate_key',
          alert: true,
          retry: false
        };
        
      case 503:  // Service unavailable
        return {
          action: 'fallback_model',
          model: 'gemini-1.5-flash',  // 降级模型
          retry: true
        };
        
      case 'TIMEOUT':
        return {
          action: 'retry',
          maxRetries: 3,
          backoff: 'exponential'  // 1s, 2s, 4s
        };
        
      case 400:  // Bad request (content violation)
        return {
          action: 'reject',
          userMessage: '请求内容不符合规范',
          retry: false
        };
        
      default:
        return {
          action: 'retry',
          maxRetries: 2,
          backoff: 'linear'
        };
    }
  }
}
```

---

## 二、边界情况处理

### 2.1 会话边界

```javascript
// 会话过期处理
class SessionManager {
  async getSession(sessionId) {
    const session = await this.store.get(`session:${sessionId}`);
    
    if (!session) {
      // 会话不存在：创建新会话
      return this.createSession(sessionId);
    }
    
    if (this.isExpired(session)) {
      // 会话过期：归档并创建新会话
      await this.archiveSession(session);
      return this.createSession(sessionId);
    }
    
    if (this.isAlmostExpired(session)) {
      // 即将过期：续期
      await this.extendSession(sessionId, 3600);
    }
    
    return session;
  }
  
  // 会话大小限制
  addMessage(sessionId, message) {
    const session = this.getSession(sessionId);
    
    // 限制历史消息数量 (保留最近50条)
    if (session.context.history.length > 50) {
      session.context.history = session.context.history.slice(-50);
      // 保存摘要到长期存储
      this.summarizeAndArchive(session);
    }
    
    // 限制消息大小 (10KB)
    if (message.content.length > 10000) {
      message.content = message.content.substring(0, 10000) + '...';
    }
    
    session.context.history.push(message);
    return this.saveSession(session);
  }
}
```

### 2.2 插件边界

```javascript
// 插件沙箱限制
class PluginSandbox {
  constructor(plugin, limits) {
    this.plugin = plugin;
    this.limits = {
      memory: limits.memory || 100 * 1024 * 1024,  // 100MB
      cpu: limits.cpu || 1000,  // 1秒CPU时间
      timeout: limits.timeout || 30000  // 30秒
    };
  }
  
  async execute(handler, args) {
    // 内存监控
    const memBefore = process.memoryUsage();
    
    // 超时控制
    const timeoutPromise = new Promise((_, reject) => {
      setTimeout(() => reject(new Error('Plugin timeout')), this.limits.timeout);
    });
    
    try {
      const result = await Promise.race([
        handler(...args),
        timeoutPromise
      ]);
      
      // 内存检查
      const memAfter = process.memoryUsage();
      const memUsed = memAfter.heapUsed - memBefore.heapUsed;
      
      if (memUsed > this.limits.memory) {
        throw new Error(`Memory limit exceeded: ${memUsed} bytes`);
      }
      
      return result;
    } catch (error) {
      // 隔离错误，不影响其他插件
      this.logger.error(`Plugin ${this.plugin.name} error:`, error);
      throw error;
    }
  }
}
```

### 2.3 资源边界

| 资源 | 限制 | 超限处理 |
|------|------|----------|
| 单个请求体 | 10MB | 返回413错误 |
| WebSocket消息 | 1MB | 截断并告警 |
| 文件上传 | 50MB | 拒绝并提示 |
| 并发请求/用户 | 10 | 排队或429 |
| 全局并发 | 1000 | 负载均衡 |
| 数据库连接 | 20 | 连接池等待 |
| 插件内存 | 100MB | OOM杀死并重启 |

---

## 三、测试策略

### 3.1 测试金字塔

```
        /\
       /  \
      / E2E \          # 端到端测试 (5%)
     /─────────\
    / Integration \    # 集成测试 (15%)
   /─────────────────\
  /     Unit Tests    \  # 单元测试 (80%)
 /─────────────────────────\
```

### 3.2 单元测试规范

```javascript
// core.test.js
describe('B2BAICore', () => {
  let core;
  
  beforeEach(() => {
    core = new B2BAICore({
      config: { ... },
      logger: mockLogger
    });
  });
  
  describe('插件管理', () => {
    test('应该成功加载有效插件', async () => {
      const plugin = await core.loadPlugin('test-plugin');
      expect(plugin.state).toBe('active');
    });
    
    test('应该拒绝无效manifest', async () => {
      await expect(core.loadPlugin('invalid-plugin'))
        .rejects.toThrow('Invalid manifest');
    });
    
    test('插件崩溃不应影响Core', async () => {
      const badPlugin = {
        onMessage: () => { throw new Error('Crash'); }
      };
      
      const result = await core.dispatch(message, { plugin: badPlugin });
      expect(result.error).toBe(true);
      expect(core.state).toBe('healthy');
    });
  });
  
  describe('Provider切换', () => {
    test('主Provider失败时应自动切换', async () => {
      const primary = { complete: jest.fn().mockRejectedValue(new Error('Down')) };
      const backup = { complete: jest.fn().mockResolvedValue({ content: 'OK' }) };
      
      core.registerProvider('primary', primary);
      core.registerProvider('backup', backup);
      
      const result = await core.completeWithFallback(params);
      expect(backup.complete).toHaveBeenCalled();
    });
    
    test('所有Provider失败时应抛出错误', async () => {
      const providers = [
        { complete: jest.fn().mockRejectedValue(new Error('Down')) },
        { complete: jest.fn().mockRejectedValue(new Error('Also down')) }
      ];
      
      await expect(core.completeWithFallback(params))
        .rejects.toThrow('All providers failed');
    });
  });
});

// provider.test.js
describe('GeminiProvider', () => {
  test('应该正确处理配额耗尽', async () => {
    const provider = new GeminiProvider({ key: 'test' });
    
    mockFetch.mockRejectedValue({ status: 429, message: 'Rate limited' });
    
    await expect(provider.complete(params))
      .rejects.toMatchObject({ code: 'RATE_LIMIT' });
  });
  
  test('应该正确统计Token用量', async () => {
    const provider = new GeminiProvider({ key: 'test' });
    
    mockFetch.mockResolvedValue({
      candidates: [{ content: { parts: [{ text: 'Hello' }] } }],
      usageMetadata: { promptTokenCount: 10, candidatesTokenCount: 5 }
    });
    
    const result = await provider.complete(params);
    expect(result.usage).toEqual({ input: 10, output: 5, total: 15 });
  });
});
```

### 3.3 集成测试

```javascript
// integration/api.test.js
describe('API Integration', () => {
  let app;
  
  beforeAll(async () => {
    app = await createTestApp();
  });
  
  test('完整对话流程', async () => {
    // 1. 创建会话
    const sessionRes = await request(app)
      .post('/api/v1/sessions')
      .send({ tenantId: 'test' });
    
    const sessionId = sessionRes.body.id;
    
    // 2. 发送消息
    const messageRes = await request(app)
      .post('/api/v1/message')
      .set('X-Session-Id', sessionId)
      .send({ content: 'Hello' });
    
    expect(messageRes.status).toBe(200);
    expect(messageRes.body.content).toBeTruthy();
    
    // 3. 验证会话历史
    const historyRes = await request(app)
      .get('/api/v1/sessions/' + sessionId)
      .set('X-Session-Id', sessionId);
    
    expect(historyRes.body.context.history).toHaveLength(2); // user + assistant
  });
  
  test('Provider故障转移', async () => {
    // Mock Gemini失败
    mockGemini.complete.mockRejectedValue(new Error('Down'));
    
    // Groq应该接管
    const res = await request(app)
      .post('/api/v1/message')
      .send({ content: 'Test' });
    
    expect(res.status).toBe(200);
    expect(mockGroq.complete).toHaveBeenCalled();
  });
});
```

### 3.4 E2E测试

```javascript
// e2e/conversation.spec.js
describe('端到端对话测试', () => {
  test('用户完整对话流程', async () => {
    const browser = await puppeteer.launch();
    const page = await browser.newPage();
    
    // 访问页面
    await page.goto('https://stdmaterial.com/buildai/');
    
    // 输入消息
    await page.type('[data-testid="message-input"]', '你好');
    await page.click('[data-testid="send-button"]');
    
    // 等待响应
    await page.waitForSelector('[data-testid="assistant-message"]');
    
    const response = await page.$eval('[data-testid="assistant-message"]', el => el.textContent);
    expect(response).toBeTruthy();
    
    await browser.close();
  });
});
```

### 3.5 性能测试

```javascript
// performance/load.test.js
describe('性能测试', () => {
  test('并发100用户', async () => {
    const promises = Array(100).fill().map(() =>
      request(app)
        .post('/api/v1/message')
        .send({ content: 'Hello' })
    );
    
    const results = await Promise.all(promises);
    const successCount = results.filter(r => r.status === 200).length;
    
    expect(successCount).toBeGreaterThanOrEqual(95); // 95%成功率
  });
  
  test('内存泄漏检测', async () => {
    const initialMem = process.memoryUsage().heapUsed;
    
    // 执行1000次请求
    for (let i = 0; i < 1000; i++) {
      await request(app).post('/api/v1/message').send({ content: 'Test' });
    }
    
    // 强制GC
    if (global.gc) global.gc();
    
    const finalMem = process.memoryUsage().heapUsed;
    const growth = (finalMem - initialMem) / 1024 / 1024; // MB
    
    expect(growth).toBeLessThan(50); // 内存增长 < 50MB
  });
});
```

### 3.6 混沌测试

```javascript
// chaos/network.test.js
describe('混沌测试', () => {
  test('网络抖动下的稳定性', async () => {
    // 模拟网络延迟
    networkInterceptor.addDelay(100, 500);
    
    // 模拟随机丢包
    networkInterceptor.setPacketLoss(0.1);
    
    const results = await Promise.allSettled(
      Array(50).fill().map(() =>
        request(app).post('/api/v1/message').send({ content: 'Test' })
      )
    );
    
    const successRate = results.filter(r => r.status === 'fulfilled').length / results.length;
    expect(successRate).toBeGreaterThan(0.8); // 80%成功率
  });
  
  test('数据库故障恢复', async () => {
    // 模拟数据库断开
    await db.disconnect();
    
    // 请求应该使用内存缓存
    const res = await request(app).get('/health');
    expect(res.status).toBe(200);
    expect(res.body.components.database).toBe('degraded');
    
    // 恢复数据库
    await db.reconnect();
    
    // 验证自动恢复
    const res2 = await request(app).get('/health');
    expect(res2.body.components.database).toBe('healthy');
  });
});
```

---

## 四、测试覆盖率要求

| 模块 | 行覆盖率 | 分支覆盖率 | 关键路径 |
|------|----------|------------|----------|
| Core | >= 90% | >= 85% | 100% |
| Providers | >= 85% | >= 80% | 100% |
| Adapters | >= 85% | >= 80% | 100% |
| Plugins | >= 80% | >= 75% | 100% |
| Utils | >= 75% | >= 70% | - |

---

## 五、CI/CD测试流程

```yaml
# .github/workflows/test.yml
name: Test

on: [push, pull_request]

jobs:
  unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm ci
      - run: npm run test:unit -- --coverage
      - uses: codecov/codecov-action@v3

  integration:
    runs-on: ubuntu-latest
    needs: unit
    steps:
      - uses: actions/checkout@v3
      - run: docker-compose up -d db redis
      - run: npm ci
      - run: npm run test:integration

  e2e:
    runs-on: ubuntu-latest
    needs: integration
    steps:
      - uses: actions/checkout@v3
      - run: npm ci
      - run: npm run test:e2e

  performance:
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v3
      - run: npm ci
      - run: npm run test:perf
```

---

*下一轮细化(v1.3): 增加部署指南、监控指标、故障排查手册*
