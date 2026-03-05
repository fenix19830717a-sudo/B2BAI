# B2BAI 部署与运维手册 (细化版 v1.3)

> 版本: v1.3 | 更新: 2026-03-05 | 细化轮次: 3/3

---

## 一、部署架构

### 1.1 生产环境架构

```
┌─────────────────────────────────────────────────────────────┐
│                         用户访问层                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │   Web用户    │  │  API客户端  │  │  移动APP    │          │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘          │
└─────────┼────────────────┼────────────────┼─────────────────┘
          │                │                │
          ▼                ▼                ▼
┌─────────────────────────────────────────────────────────────┐
│                      负载均衡层 (Nginx)                       │
│  ┌─────────────────────────────────────────────────────────┐│
│  │  SSL终止 / 静态资源缓存 / 反向代理 / 限流               ││
│  │  Location: /buildai/ → B2BAI Core                      ││
│  │  Location: /buildai/api/ → B2BAI API                   ││
│  └─────────────────────────────────────────────────────────┘│
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                     B2BAI Core 集群                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │  Instance 1 │  │  Instance 2 │  │  Instance N │         │
│  │  Port:3456  │  │  Port:3457  │  │  Port:34XX  │         │
│  │  PM2管理    │  │  PM2管理    │  │  PM2管理    │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
└─────────────────────────────────────────────────────────────┘
          │                │                │
          ▼                ▼                ▼
┌─────────────────────────────────────────────────────────────┐
│                      数据存储层                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │   SQLite    │  │   Redis     │  │  File Store │         │
│  │  (主数据)   │  │  (缓存/Sess)│  │  (上传文件) │         │
│  │  定期备份   │  │  持久化开启 │  │  定期清理   │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
└─────────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────┐
│                      监控告警层                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │  Prometheus │  │   Grafana   │  │   Alertman  │         │
│  │  指标采集   │  │  可视化     │  │  告警通知   │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 服务器资源要求

| 环境 | CPU | 内存 | 磁盘 | 网络 | 实例数 |
|------|-----|------|------|------|--------|
| 开发 | 1核 | 2GB | 20GB | 10Mbps | 1 |
| 测试 | 2核 | 4GB | 50GB | 50Mbps | 1-2 |
| 生产 | 4核 | 8GB | 100GB | 100Mbps | 2+ |

---

## 二、部署流程

### 2.1 一键部署脚本

```bash
#!/bin/bash
# deploy.sh - B2BAI 一键部署脚本

set -e

# 配置
APP_NAME="b2bai"
APP_DIR="/opt/b2bai"
APP_USER="b2bai"
APP_PORT=3456
NODE_VERSION="20"

# 颜色输出
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

log() { echo -e "${GREEN}[$(date +%H:%M:%S)] $1${NC}"; }
warn() { echo -e "${YELLOW}[$(date +%H:%M:%S)] $1${NC}"; }
error() { echo -e "${RED}[$(date +%H:%M:%S)] $1${NC}"; exit 1; }

# 1. 系统检查
check_system() {
    log "检查系统环境..."
    
    # 检查Node.js
    if ! command -v node &> /dev/null; then
        warn "Node.js未安装，正在安装..."
        curl -fsSL https://deb.nodesource.com/setup_${NODE_VERSION}.x | bash -
        apt-get install -y nodejs
    fi
    
    # 检查版本
    NODE_CURRENT=$(node -v | cut -d'v' -f2 | cut -d'.' -f1)
    if [ "$NODE_CURRENT" -lt "18" ]; then
        error "Node.js版本过低，需要18+"
    fi
    
    # 检查PM2
    if ! command -v pm2 &> /dev/null; then
        warn "PM2未安装，正在安装..."
        npm install -g pm2
    fi
    
    # 检查Nginx
    if ! command -v nginx &> /dev/null; then
        warn "Nginx未安装，正在安装..."
        apt-get update && apt-get install -y nginx
    fi
    
    log "系统环境检查完成"
}

# 2. 创建用户和目录
setup_user() {
    log "创建用户和目录..."
    
    # 创建用户
    if ! id "$APP_USER" &> /dev/null; then
        useradd -r -s /bin/false -m -d "$APP_DIR" "$APP_USER"
    fi
    
    # 创建目录结构
    mkdir -p "$APP_DIR"/{core,plugins,config,data,logs}
    chown -R "$APP_USER:$APP_USER" "$APP_DIR"
    
    log "用户和目录创建完成"
}

# 3. 部署代码
deploy_code() {
    log "部署代码..."
    
    cd "$APP_DIR"
    
    # 克隆代码
    if [ -d ".git" ]; then
        git pull origin main
    else
        git clone https://github.com/fenix19830717a-sudo/B2BAI.git .
    fi
    
    # 安装依赖
    cd core
    npm ci --production
    
    # 设置权限
    chown -R "$APP_USER:$APP_USER" "$APP_DIR"
    
    log "代码部署完成"
}

# 4. 配置环境
setup_config() {
    log "配置环境..."
    
    # 创建配置文件
    cat > "$APP_DIR/config/core.yaml" << 'EOF'
server:
  port: ${APP_PORT}
  host: "127.0.0.1"

database:
  type: "sqlite"
  path: "./data/b2bai.db"

logging:
  level: "info"
  format: "json"
  output: "./logs/b2bai.log"
  maxSize: "100m"
  maxFiles: 7

providers:
  gemini:
    keys:
      - "${GEMINI_KEY_1}"
      - "${GEMINI_KEY_2}"
    defaultModel: "gemini-2.0-flash"
  
  groq:
    key: "${GROQ_KEY}"
    defaultModel: "llama-3.3-70b-versatile"
EOF
    
    # 创建环境变量文件
    touch "$APP_DIR/.env"
    chmod 600 "$APP_DIR/.env"
    
    log "配置完成，请编辑 $APP_DIR/.env 设置API密钥"
}

# 5. 配置Nginx
setup_nginx() {
    log "配置Nginx..."
    
    cat > /etc/nginx/sites-available/b2bai << EOF
upstream b2bai_backend {
    least_conn;
    server 127.0.0.1:${APP_PORT} max_fails=3 fail_timeout=30s;
}

server {
    listen 80;
    server_name _;
    
    location /buildai/ {
        alias /var/www/html/buildai/;
        try_files \$uri \$uri/ /buildai/index.html;
    }
    
    location /buildai/api/ {
        proxy_pass http://b2bai_backend/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade \$http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_cache_bypass \$http_upgrade;
        
        # 超时设置
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
    
    # 静态资源缓存
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
}
EOF
    
    ln -sf /etc/nginx/sites-available/b2bai /etc/nginx/sites-enabled/
    nginx -t && systemctl reload nginx
    
    log "Nginx配置完成"
}

# 6. 配置PM2
setup_pm2() {
    log "配置PM2..."
    
    cat > "$APP_DIR/ecosystem.config.js" << 'EOF'
module.exports = {
  apps: [{
    name: 'b2bai',
    script: './core/index.js',
    cwd: '/opt/b2bai',
    instances: 1,
    exec_mode: 'fork',
    env: {
      NODE_ENV: 'production',
      PORT: 3456
    },
    log_file: './logs/combined.log',
    out_file: './logs/out.log',
    error_file: './logs/error.log',
    log_date_format: 'YYYY-MM-DD HH:mm:ss Z',
    merge_logs: true,
    max_memory_restart: '500M',
    min_uptime: '10s',
    max_restarts: 10,
    restart_delay: 3000,
    kill_timeout: 5000,
    listen_timeout: 10000,
    source_map_support: true,
    // 自动重启策略
    autorestart: true,
    // 崩溃后重启
    exp_backoff_restart_delay: 100,
    // 监控
    monitoring: true,
    // 环境变量文件
    env_file: '.env'
  }]
};
EOF
    
    log "PM2配置完成"
}

# 7. 启动服务
start_service() {
    log "启动服务..."
    
    cd "$APP_DIR"
    
    # 使用PM2启动
    sudo -u "$APP_USER" pm2 start ecosystem.config.js
    sudo -u "$APP_USER" pm2 save
    
    # 设置开机自启
    env PATH=$PATH:/usr/bin pm2 startup systemd -u "$APP_USER" --hp "$APP_DIR"
    
    log "服务启动完成"
}

# 8. 健康检查
health_check() {
    log "健康检查..."
    
    sleep 3
    
    if curl -sf http://localhost:${APP_PORT}/health > /dev/null; then
        log "✅ 服务健康检查通过"
    else
        error "❌ 服务健康检查失败，请查看日志"
    fi
}

# 主流程
main() {
    log "开始部署 B2BAI..."
    
    check_system
    setup_user
    deploy_code
    setup_config
    setup_nginx
    setup_pm2
    start_service
    health_check
    
    log "部署完成！"
    log "访问地址: http://your-domain/buildai/"
    log "日志查看: pm2 logs b2bai"
    log "状态检查: pm2 status"
}

# 执行
main "$@"
```

### 2.2 Docker部署

```dockerfile
# Dockerfile
FROM node:20-alpine

WORKDIR /app

# 安装依赖
COPY core/package*.json ./
RUN npm ci --production && npm cache clean --force

# 复制代码
COPY core/ ./
COPY config/ ./config/

# 创建数据目录
RUN mkdir -p /app/data /app/logs /app/plugins

# 非root用户
USER node

EXPOSE 3456

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD node -e "require('http').get('http://localhost:3456/health', (r) => r.statusCode === 200 ? process.exit(0) : process.exit(1))"

CMD ["node", "index.js"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  b2bai:
    build: .
    container_name: b2bai
    restart: unless-stopped
    ports:
      - "127.0.0.1:3456:3456"
    environment:
      - NODE_ENV=production
      - PORT=3456
    env_file:
      - .env
    volumes:
      - ./data:/app/data
      - ./logs:/app/logs
      - ./plugins:/app/plugins
      - ./config:/app/config
    networks:
      - b2bai-network
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 1G
        reservations:
          cpus: '0.5'
          memory: 256M
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:3456/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  redis:
    image: redis:7-alpine
    container_name: b2bai-redis
    restart: unless-stopped
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis-data:/data
    networks:
      - b2bai-network
    deploy:
      resources:
        limits:
          memory: 256M

  nginx:
    image: nginx:alpine
    container_name: b2bai-nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
      - ./web:/var/www/html:ro
    networks:
      - b2bai-network
    depends_on:
      - b2bai

volumes:
  redis-data:

networks:
  b2bai-network:
    driver: bridge
```

---

## 三、监控指标

### 3.1 核心指标

| 指标名 | 类型 | 说明 | 告警阈值 |
|--------|------|------|----------|
| `b2bai_requests_total` | Counter | 总请求数 | - |
| `b2bai_request_duration_seconds` | Histogram | 请求延迟 | P99 > 1s |
| `b2bai_active_sessions` | Gauge | 活跃会话数 | > 5000 |
| `b2bai_plugin_errors_total` | Counter | 插件错误数 | > 10/min |
| `b2bai_provider_latency` | Gauge | Provider延迟 | > 5s |
| `b2bai_memory_usage_bytes` | Gauge | 内存使用 | > 80% |
| `b2bai_cpu_usage_percent` | Gauge | CPU使用 | > 80% |
| `b2bai_disk_usage_percent` | Gauge | 磁盘使用 | > 85% |

### 3.2 业务指标

```javascript
// metrics.js
const promClient = require('prom-client');

// 创建指标
const requestCounter = new promClient.Counter({
  name: 'b2bai_requests_total',
  help: 'Total number of requests',
  labelNames: ['method', 'route', 'status']
});

const requestDuration = new promClient.Histogram({
  name: 'b2bai_request_duration_seconds',
  help: 'Request duration in seconds',
  labelNames: ['method', 'route'],
  buckets: [0.1, 0.5, 1, 2, 5, 10]
});

const activeSessions = new promClient.Gauge({
  name: 'b2bai_active_sessions',
  help: 'Number of active sessions'
});

const providerLatency = new promClient.Gauge({
  name: 'b2bai_provider_latency_seconds',
  help: 'Provider latency',
  labelNames: ['provider', 'model']
});

const tokenUsage = new promClient.Counter({
  name: 'b2bai_token_usage_total',
  help: 'Total token usage',
  labelNames: ['provider', 'model', 'type']
});

// 导出指标
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', promClient.register.contentType);
  res.end(await promClient.register.metrics());
});
```

### 3.3 健康检查端点

```javascript
// health.js
app.get('/health', async (req, res) => {
  const checks = {
    timestamp: new Date().toISOString(),
    version: process.env.npm_package_version,
    uptime: process.uptime(),
    components: {}
  };
  
  // Core状态
  checks.components.core = 'healthy';
  
  // 数据库状态
  try {
    await db.ping();
    checks.components.database = 'healthy';
  } catch (err) {
    checks.components.database = 'unhealthy';
  }
  
  // Provider状态
  for (const [name, provider] of Object.entries(providers)) {
    try {
      await provider.health();
      checks.components[`provider_${name}`] = 'healthy';
    } catch (err) {
      checks.components[`provider_${name}`] = 'unhealthy';
    }
  }
  
  // 插件状态
  for (const [name, plugin] of Object.entries(plugins)) {
    checks.components[`plugin_${name}`] = plugin.state;
  }
  
  // 系统资源
  const memUsage = process.memoryUsage();
  checks.resources = {
    memory: {
      used: Math.round(memUsage.heapUsed / 1024 / 1024),
      total: Math.round(memUsage.heapTotal / 1024 / 1024),
      unit: 'MB'
    },
    cpu: process.cpuUsage()
  };
  
  const isHealthy = Object.values(checks.components)
    .every(s => s === 'healthy' || s === 'active');
  
  res.status(isHealthy ? 200 : 503).json(checks);
});

// 就绪检查
app.get('/ready', (req, res) => {
  const ready = core.state === 'initialized' && db.isConnected();
  res.status(ready ? 200 : 503).json({ ready });
});
```

---

## 四、故障排查手册

### 4.1 常见问题速查

#### 🔴 服务无法启动

```bash
# 1. 检查日志
pm2 logs b2bai --lines 100

# 2. 检查端口占用
sudo lsof -i :3456

# 3. 检查配置
node -e "console.log(require('./config/core.yaml'))"

# 4. 检查权限
ls -la /opt/b2bai/

# 5. 手动启动调试
cd /opt/b2bai && node core/index.js
```

#### 🟡 请求超时

```bash
# 检查Provider状态
curl http://localhost:3456/health | jq '.components'

# 检查Provider延迟
curl -w "@curl-format.txt" http://localhost:3456/api/v1/message \
  -d '{"content":"test"}' -H "Content-Type: application/json"

# 查看慢请求日志
grep "timeout" /opt/b2bai/logs/error.log
```

#### 🟠 内存泄漏

```bash
# 查看内存趋势
pm2 monit

# 堆快照分析
node --heapsnapshot-near-heap-limit=1 core/index.js

# 使用 clinic.js 诊断
npx clinic doctor -- node core/index.js
```

#### 🔵 数据库锁定

```bash
# 检查SQLite锁定
lsof /opt/b2bai/data/b2bai.db

# 修复数据库
sqlite3 /opt/b2bai/data/b2bai.db ".recover" | sqlite3 fixed.db
mv fixed.db /opt/b2bai/data/b2bai.db
```

### 4.2 日志分析

```bash
# 实时日志
pm2 logs b2bai --raw

# 错误日志
pm2 logs b2bai --err

# 按时间过滤
grep "2026-03-05" /opt/b2bai/logs/combined.log | grep ERROR

# 统计错误
pm2 logs b2bai --lines 1000 | grep ERROR | wc -l

# 慢请求分析
grep "duration" /opt/b2bai/logs/combined.log | \
  awk '{print $NF}' | sort -n | tail -20
```

### 4.3 紧急恢复流程

```bash
#!/bin/bash
# emergency-restore.sh - 紧急恢复脚本

echo "🚨 开始紧急恢复..."

# 1. 停止服务
pm2 stop b2bai

# 2. 备份当前数据
tar -czf /backup/b2bai-emergency-$(date +%Y%m%d-%H%M%S).tar.gz /opt/b2bai/data

# 3. 检查磁盘空间
df -h

# 4. 清理日志
gzip /opt/b2bai/logs/*.log
find /opt/b2bai/logs -name "*.gz" -mtime +7 -delete

# 5. 修复权限
chown -R b2bai:b2bai /opt/b2bai

# 6. 检查配置
if [ -f /opt/b2bai/config/core.yaml ]; then
    echo "✅ 配置文件存在"
else
    echo "❌ 配置文件丢失，恢复默认配置"
    cp /opt/b2bai/config/core.yaml.default /opt/b2bai/config/core.yaml
fi

# 7. 启动服务
pm2 start b2bai

# 8. 健康检查
sleep 5
if curl -sf http://localhost:3456/health; then
    echo "✅ 恢复成功"
else
    echo "❌ 恢复失败，请检查日志"
    pm2 logs b2bai --lines 50
fi
```

---

## 五、备份策略

### 5.1 自动备份脚本

```bash
#!/bin/bash
# backup.sh - 自动备份

BACKUP_DIR="/backup/b2bai"
DB_PATH="/opt/b2bai/data/b2bai.db"
CONFIG_PATH="/opt/b2bai/config"
DATE=$(date +%Y%m%d-%H%M%S)
RETENTION_DAYS=30

mkdir -p "$BACKUP_DIR"

# 备份数据库
cp "$DB_PATH" "$BACKUP_DIR/db-$DATE.sqlite"
sqlite3 "$BACKUP_DIR/db-$DATE.sqlite" ".backup '$BACKUP_DIR/db-$DATE.sqlite'"

# 备份配置
tar -czf "$BACKUP_DIR/config-$DATE.tar.gz" -C "$CONFIG_PATH" .

# 上传到S3 (可选)
# aws s3 cp "$BACKUP_DIR/db-$DATE.sqlite" s3://my-bucket/b2bai-backups/

# 清理旧备份
find "$BACKUP_DIR" -name "*.sqlite" -mtime +$RETENTION_DAYS -delete
find "$BACKUP_DIR" -name "*.tar.gz" -mtime +$RETENTION_DAYS -delete

echo "备份完成: $DATE"
```

```cron
# 每天凌晨3点备份
0 3 * * * /opt/b2bai/scripts/backup.sh >> /var/log/b2bai-backup.log 2>&1
```

---

## 六、安全加固

### 6.1 基础安全

```bash
# 1. 防火墙配置
ufw allow 80/tcp
ufw allow 443/tcp
ufw deny 3456/tcp  # 仅允许本地访问
ufw enable

# 2. 文件权限
chmod 600 /opt/b2bai/.env
chmod 600 /opt/b2bai/config/core.yaml
chmod 750 /opt/b2bai/data
chmod 755 /opt/b2bai/logs

# 3. 禁用root运行
useradd -r -s /bin/false b2bai
chown -R b2bai:b2bai /opt/b2bai
```

### 6.2 安全头配置

```nginx
# nginx安全头
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'" always;

# 限流
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
limit_req zone=api burst=20 nodelay;

# 连接限制
limit_conn_zone $binary_remote_addr zone=addr:10m;
limit_conn addr 10;
```

---

*文档细化完成 (v1.3)*
