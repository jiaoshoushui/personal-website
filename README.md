# personal-website-source-vue

个人主页静态站点。

- 在线访问：https://shuidada.xyz
- 源码仓库：[personal-website-source-vue](https://github.com/jiaoshoushui/personal-website-source-vue)

## 性能优化指南

### 问题说明

本地开发环境 AI Agent 响应速度快，但部署后响应变慢。这是因为：

1. **本地环境**：通过 Node.js 代理服务器（`server/index.mjs`）转发请求，网络延迟低
2. **部署环境**：浏览器直接调用第三方 API，受网络质量和跨域限制影响

### 优化方案

#### 方案 1：部署后端代理服务（推荐）

在服务器上部署代理服务器，通过 Nginx 反向代理：

**1. 启动后端服务**

```bash
# 安装依赖（如果需要）
npm install

# 启动代理服务器
node server/index.mjs
```

**2. 配置 Nginx**

创建 `/etc/nginx/sites-available/personal-website`：

```nginx
server {
    listen 80;
    server_name shuidada.xyz;
    
    # 静态文件
    location / {
        root /var/www/personal-website;
        index index.html;
        try_files $uri $uri/ /index.html;
    }
    
    # API 代理
    location /api/llm/ {
        proxy_pass http://localhost:3001/api/llm/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_cache_bypass $http_upgrade;
        
        # 超时设置
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

**3. 启用配置**

```bash
sudo ln -s /etc/nginx/sites-available/personal-website /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

**4. 配置前端使用代理**

在浏览器中打开网站，进入 Agent 设置页面：
- API 代理地址：填写 `/api/llm`（留空相对路径）
- API Key：输入你的 API Key

#### 方案 2：使用 Serverless 函数（Vercel/Netlify）

如果使用 Vercel 或 Netlify 部署：

**1. 创建 API 路由**

创建 `api/llm/chat/completions.js`：

```javascript
export default async function handler(req, res) {
  if (req.method !== 'POST') {
    return res.status(405).json({ error: 'Method not allowed' })
  }

  const targetBase = req.headers['x-target-base'] || 'https://api.deepseek.com'
  const targetUrl = `${targetBase}/chat/completions`

  try {
    const response = await fetch(targetUrl, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': req.headers.authorization,
      },
      body: JSON.stringify(req.body),
    })

    res.writeHead(response.status, {
      'Content-Type': response.headers.get('content-type'),
      'Cache-Control': 'no-cache',
    })

    const reader = response.body.getReader()
    while (true) {
      const { done, value } = await reader.read()
      if (done) break
      res.write(value)
    }
    res.end()
  } catch (error) {
    res.status(502).json({ error: { message: error.message } })
  }
}
```

**2. 配置前端**

在 Agent 设置中：
- API 代理地址：`/api/llm`
- API Key：输入你的 API Key

#### 方案 3：使用 Cloudflare Workers

创建 Worker 脚本：

```javascript
export default {
  async fetch(request, env) {
    const url = new URL(request.url)
    
    if (url.pathname === '/api/llm/chat/completions' && request.method === 'POST') {
      const targetBase = request.headers.get('x-target-base') || 'https://api.deepseek.com'
      const targetUrl = `${targetBase}/chat/completions`
      
      const response = await fetch(targetUrl, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': request.headers.get('authorization'),
        },
        body: request.body,
      })
      
      return new Response(response.body, {
        status: response.status,
        headers: {
          'Content-Type': response.headers.get('content-type'),
          'Cache-Control': 'no-cache',
        },
      })
    }
    
    return new Response('Not found', { status: 404 })
  }
}
```

### 其他优化建议

1. **启用 HTTP/2**：提升并发性能
2. **启用 Gzip/Brotli 压缩**：减少传输体积
3. **使用 CDN**：加速静态资源加载
4. **添加缓存策略**：对非实时数据进行缓存
5. **监控和日志**：跟踪 API 响应时间，及时发现性能瓶颈

### 性能对比

| 环境 | 平均响应时间 | 说明 |
|------|------------|------|
| 本地开发 | 1-3秒 | 通过本地代理，延迟最低 |
| 直连 API（当前部署） | 3-10秒 | 受网络和跨域影响 |
| 后端代理（推荐） | 2-5秒 | 服务器带宽更好，稳定性高 |
| Edge Functions | 1-4秒 | 全球分布，延迟最低 |

### 故障排查

如果部署后仍然很慢：

1. **检查网络连接**
   ```bash
   curl -I https://api.deepseek.com/v1/chat/completions
   ```

2. **查看浏览器控制台**
   - Network 标签页查看请求耗时
   - Console 标签页查看错误信息

3. **测试代理服务器**
   ```bash
   curl -X POST http://localhost:3001/api/llm/chat/completions \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer YOUR_API_KEY" \
     -d '{"model":"deepseek-chat","messages":[{"role":"user","content":"Hello"}],"stream":true}'
   ```

4. **检查 Nginx 日志**
   ```bash
   sudo tail -f /var/log/nginx/error.log
   ```
