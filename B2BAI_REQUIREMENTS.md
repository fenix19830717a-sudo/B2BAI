# B2BAI 功能需求规格

## 一、保留功能 (BuildAI 继承)

### 1.1 多租户系统
- 基于子域名的租户隔离
- 租户配置管理
- 自定义域名支持

### 1.2 网站模板系统
- 多套主题切换
- 主题自定义配置
- 模板渲染引擎

## 二、交易机器人策略 (新开发)

### 2.1 现有策略继承
从 `/opt/polmarket-bot/current/` 继承以下策略：
- **TIME_WINDOW_ARB** - 时间窗口套利
- **SPREAD_FARMING** - 价差农场
- **DIRECTION_HUNTING** - 方向狩猎

### 2.2 新交易机器人应用
- 作为 B2BAI 插件开发
- 前端设置界面
- 策略参数可配置
- 交易记录可视化

### 2.3 迁移策略
- 阶段1: 新机器人开发与测试
- 阶段2: 平台验收通过
- 阶段3: 停止现有机器人

## 三、API Key 分配规则

### 3.1 Gemini Key 分配
| Key | 用途 | 项目 | 备注 |
|-----|------|------|------|
| Key 1 | 主API/通用功能 | gen-lang-client-0903904152 | 预留20%额度 |
| Key 2 | 产品图片生成 | gen-lang-client-0094689764 | 图像生成专用 |
| Key 3 | 客户咨询 | gen-lang-client-0018028679 | 客服对话 |
| Key 4 | 管理功能 | gen-lang-client-0393516872 | 后台管理 |

### 3.2 轮询策略
```javascript
// 伪代码
class GeminiKeyRotator {
  constructor() {
    this.keys = [key1, key2, key3, key4];
    this.usage = { key1: 0, key2: 0, key3: 0, key4: 0 };
    this.limits = { rpm: 15, rpd: 1500 };
  }
  
  getKey(purpose) {
    // Key1 专用：通用功能且使用率<80%
    if (purpose === 'general' && this.usage.key1 < 0.8 * this.limits.rpd) {
      return this.keys[0];
    }
    
    // 按用途分配
    const purposeMap = {
      'image': 1,      // Key2
      'consult': 2,    // Key3
      'admin': 3       // Key4
    };
    
    const idx = purposeMap[purpose] || 0;
    
    // 如果指定Key超限，轮询其他Key
    if (this.usage[`key${idx+1}`] >= this.limits.rpd) {
      return this.findAvailableKey();
    }
    
    return this.keys[idx];
  }
}
```

### 3.3 Groq API
- 用途: 高速推理备选
- Key: [REDACTED]
- 限制: 30 RPM, 14400 RPD

## 四、系统架构

```
B2BAI Platform
├── Core Layer
│   ├── ProviderRegistry (LLM/Groq)
│   ├── AdapterRegistry (HTTP/WebSocket)
│   ├── StateManager (SQLite/Redis)
│   └── PluginLoader
│
├── Plugin Layer
│   ├── tenant-plugin (多租户)
│   ├── template-plugin (网站模板)
│   ├── trade-plugin (交易机器人)
│   ├── image-plugin (图片生成 - Key2)
│   ├── consult-plugin (客户咨询 - Key3)
│   └── admin-plugin (管理功能 - Key4)
│
└── Web Layer
    ├── 多租户网站前端
    ├── 管理后台
    └── 交易机器人控制面板
```

## 五、开发优先级

1. **P0 - Core 框架** (CTO)
   - ProviderRegistry + Gemini多Key轮询
   - HTTP Adapter
   - Plugin Loader

2. **P0 - Web UI** (Builder)
   - 修复乱码
   - Nginx反向代理
   - 基础对话界面

3. **P1 - 多租户插件** (CTO)
   - 租户隔离
   - 域名管理

4. **P1 - 模板插件** (Builder)
   - 主题系统
   - 模板渲染

5. **P2 - 交易机器人插件** (Builder)
   - 策略继承
   - 前端控制面板

## 六、验收标准

### 6.1 功能验收
- [ ] 多租户隔离正常
- [ ] 网站模板可切换
- [ ] 对话功能可用
- [ ] API Key 轮询正常
- [ ] Bot 守护在线

### 6.2 性能验收
- [ ] Core 体积 < 2MB
- [ ] 启动时间 < 3s
- [ ] 内存占用 < 200MB

### 6.3 部署验收
- [ ] 反向代理配置正确
- [ ] 访问路径正常
- [ ] 移动端适配
