# Cloudflare Pages 部署 VLESS 代理：2026 完整实战实录

> 目标：低成本搭建可用的 Google/YouTube 访问链路
> 架构：Cloudflare Pages + VLESS over WebSocket + 优选域名 + Clash Meta
> 日期：2026-10-25 | 总耗时约 12 小时 | 域名成本 ¥14

---

## 1. 最终架构

```
Clash Meta (Android)
    │  VLESS over WebSocket (TLS 443)
    ▼
优选域名 IP (ct.877774.xyz 等, SNI=自定义域名)
    │
    ▼
Cloudflare 边缘
    │
    ▼
Pages 项目 callpage-proxy (_worker.js 自写 VLESS 脚本)
    │  connect() TCP 出站
    ▼
目标站点 (Google / YouTube)
```

| 组件 | 值 |
|---|---|
| 自定义域名 | cf.callact.xyz（阿里云购买 .xyz ¥14，NS 托管到 Cloudflare） |
| 运行载体 | Cloudflare Pages 项目（direct upload） |
| 代理脚本 | 自写 119 行 VLESS over WS Worker（文末附录） |
| UUID | 自定（环境变量注入，勿提交到公开仓库） |
| 客户端 | Clash Meta for Android |
| 优选域名 | ct/cf/asia.877774.xyz、saas.sin.fan |

---

## 2. 踩坑记录（按时间线）

### 2.1 `*.workers.dev` 全域被 SNI 阻断

**现象**：Worker 部署成功后，用 `edge-app.xxx.workers.dev` 直连无法完成 TLS 握手。

**验证**：Python 直连测试证明——同一个 CF IP（104.16.148.1），`SNI=cloudflare.com` 握手成功，`SNI=workers.dev` 被 RST。

**结论**：GFW 对 workers.dev 全域实施 SNI 阻断，**必须绑定自定义域名**。

### 2.2 Workers 子域被举报 → 自定义域名全部 522

**现象**：自定义域名绑定 Worker 后一律 `HTTP 522`，响应头 `cfOrigin;dur=0`（边缘未进入 Worker 运行时）。

**根因**：账号的 workers.dev 子域于 2026-10-25 00:34 被"网络滥用"类举报（CF 滥用报告页可见），导致整个 Worker 运行时不可达。

**关键限制**：同一账号仅允许一个 Worker 子域（改子域报错 10036 "Account already has an associated subdomain"），无法通过改名规避。

**结论**：该账号 Workers 路线实质报废，**转 Pages 平台**（Pages 的 `_worker.js` 复用同一运行时，但不经 workers.dev 子域）。

### 2.3 Pages 部署原版 edgetunnel 被风控封禁（8000119）

**现象**：`wrangler pages deploy` 上传原版 edgetunnel（`_worker.js` 5MB）后，第二次部署时 API 返回：

```
Your Pages project has been blocked. Contact abusereply@cloudflare.com. [code: 8000119]
```

**根因**：CF 滥用风控识别 edgetunnel 代码特征，自动封禁项目。该账号累计 3 个 Pages 项目被封（edgetunnel / cf-ws-app / callpage-upload）。

**结论**：**不要部署原版 edgetunnel**，改用自写轻量脚本（无特征指纹，顺利通过风控）。

### 2.4 浏览器"拖放文件"上传无反应

**现象**：Android 内置浏览器里选择 zip 文件后页面无任何反应，网络面板无上传请求。

**根因**：
- 控制台警告 `File chooser dialog can only be shown with a user activation`（程序化点击被拦截）
- 内置浏览器与 CF 上传组件不兼容，选择文件后事件未正确触发

**结论**：放弃浏览器上传，改用 **API Token + Wrangler CLI**。

### 2.5 新版 Pages 直传 API 需要 manifest 字段

**现象**：multipart 直传 API 返回 8000096 "A 'manifest' field was expected"。

**文档**：2026 版 API 要求 `manifest`（JSON 编码的文件路径→hash 映射）。用 Wrangler CLI 可完全绕开该复杂度。

### 2.6 Pages Functions 与 Worker 的运行差异

**现象 A**：脚本报 `connect is not defined`。

**根因**：Worker 平台的全局 `connect()` 在 Pages Functions 中不存在，**必须显式导入**：

```js
import { connect } from 'cloudflare:sockets';
```

**现象 B**：WebSocket 收到消息后无任何响应。

**根因**：Pages 运行时可能以 `Blob` 类型传递二进制消息，需在 message handler 里处理：

```js
} else if (ev.data && typeof ev.data.arrayBuffer === 'function') {
  data = new Uint8Array(await ev.data.arrayBuffer());
}
```

### 2.7 UUID 格式不匹配

**现象**：服务器返回 close 帧 `uuid mismatch`。

**根因**：脚本从 16 字节生成的 hex 无连字符（`3f8e7d6c...`），环境变量里带连字符（`3f8e7d6c-...`）。比较前统一：

```js
const uuidStr = uuidFromBytes(uuidBytes).toLowerCase().replace(/-/g, '');
const expected = (env.UUID || '').toLowerCase().replace(/-/g, '');
```

### 2.8 删除 Worker 域名绑定会连带清空 DNS 记录

**现象**：解绑 Worker domains 后，`cf.callact.xyz` 解析失败（zone 内 DNS 记录数变 0）。

**处理**：重新添加 `CNAME cf → <项目>.pages.dev`（proxied）。

### 2.9 测试方法本身会误导结论

对 443 端口发明文 HTTP 会被目标服务器直接断连（收到 close 帧 "done"），误以为隧道故障。**正确姿势**：验证隧道用 80 端口明文 HTTP，验证真实场景用 TLS ClientHello 字节流。

---

## 3. 可行部署流程（已验证）

### 3.1 创建 API Token

CF 控制台 API Tokens → 创建 token，权限组选 **Pages Write**（`8d28297797f24fb8a0c332fe0866ec89`）。也可以用 dashboard session 调 `/api/v4/user/tokens` 生成。

### 3.2 创建 Pages 项目 + 环境变量

```bash
export TOKEN=cfut_xxxx
export ACCOUNT=<account_id>

curl -X POST "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT/pages/projects" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"name":"callpage-proxy","production_branch":"main"}'

curl -X PATCH "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT/pages/projects/callpage-proxy" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"deployment_configs":{"production":{"env_vars":{"UUID":{"value":"你的UUID"}}}}}'
```

### 3.3 Wrangler 部署

```bash
mkdir -p pages-assets && cp edge-vless-worker.js pages-assets/_worker.js
CLOUDFLARE_API_TOKEN=$TOKEN CLOUDFLARE_ACCOUNT_ID=$ACCOUNT \
  npx wrangler@latest pages deploy pages-assets --project-name=callpage-proxy
```

> 网络波动导致 `fetch failed` 时直接重试即可。项目内环境变量会在部署时自动注入。

### 3.4 绑定自定义域名 + DNS

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT/pages/projects/callpage-proxy/domains" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"name":"cf.callact.xyz"}'
```

zone 内加 DNS 记录：`CNAME cf → callpage-proxy.pages.dev`（**proxied**）。

### 3.5 链路验证（Python）

WS 升级期望 `101` → 发 VLESS 头（`0x00 + UUID16 + 0x00 + 0x01 + port2 + 0x02 + len + host`）+ `GET /generate_204` → 期望回包含 `204`。

---

## 4. Clash 配置要点

```yaml
proxies:
  - name: "电信优选"
    type: vless
    server: ct.877774.xyz        # 优选域名(IP)
    port: 443
    uuid: <你的UUID>
    network: ws
    tls: true
    udp: true
    servername: cf.callact.xyz   # SNI 用自定义域名
    client-fingerprint: chrome
    ws-opts:
      path: "/"
      headers:
        Host: cf.callact.xyz     # Host 头必须与 SNI 一致
```

**要点**：server 填优选域名 IP 池，servername 与 Host 填自定义域名；优选域名失效时 server 可直接改 `cf.callact.xyz`。

---

## 5. 安全提醒（血泪教训）

1. **UUID 是节点的唯一凭据**——拿到 UUID+域名即等于拿到你的节点。切勿提交到公开仓库/对话。
2. **GitHub token 泄露立即吊销**：Settings → Developer settings → Personal access tokens。
3. **CF 账号高风险画像**：本账号已含 1 条滥用举报 + 3 个被封 Pages 项目。同样操作请用干净账号。
4. **原版 edgetunnel 已进风控指纹库**：2026 年新部署大概率 8000119，自写脚本是最稳路线。
5. Pages 免费版限制：`_worker.js` ≤ 25MiB；免费域名（us.kg/desec/freedns 等）2026 年基本全被滥用收紧，.xyz 首年 ¥14 是最低成本正解。

---

## 附录：自写 VLESS over WS Worker 脚本（119 行）

```js
import { connect } from 'cloudflare:sockets';

const encoder = new TextEncoder();
const decoder = new TextDecoder();

export default {
  async fetch(request, env) {
    const upgrade = (request.headers.get('Upgrade') || '').toLowerCase();
    if (upgrade !== 'websocket') {
      return new Response(
        'VLESS proxy is running.\nUUID: ' + (env.UUID || 'not set') + '\n',
        { status: 200, headers: { 'content-type': 'text/plain; charset=utf-8' } }
      );
    }
    const pair = new WebSocketPair();
    const clientWs = pair[0];
    const serverWs = pair[1];
    serverWs.accept();
    startSession(serverWs, env);
    return new Response(null, { status: 101, webSocket: clientWs });
  }
};

function uuidFromBytes(b) {
  const h = Array.from(b).map((x) => x.toString(16).padStart(2, '0')).join('');
  return h.slice(0, 8) + '-' + h.slice(8, 12) + '-' + h.slice(12, 16) + '-' + h.slice(16, 20) + '-' + h.slice(20);
}

async function startSession(ws, env) {
  let writer = null;
  let opened = false;

  ws.addEventListener('message', async (ev) => {
    let data;
    if (typeof ev.data === 'string') {
      data = encoder.encode(ev.data);
    } else if (ev.data instanceof ArrayBuffer) {
      data = new Uint8Array(ev.data);
    } else if (ArrayBuffer.isView(ev.data)) {
      data = new Uint8Array(ev.data.buffer, ev.data.byteOffset, ev.data.byteLength);
    } else if (ev.data && typeof ev.data.arrayBuffer === 'function') {
      data = new Uint8Array(await ev.data.arrayBuffer());
    } else {
      ws.send(new Uint8Array([0, 0]).buffer);
      ws.close(1011, 'bad data type');
      return;
    }

    if (!opened) {
      if (data.length < 2) { ws.send(new Uint8Array([0, 0]).buffer); ws.close(1011, 'short header'); return; }
      const uuidBytes = data.slice(1, 17);
      const uuidStr = uuidFromBytes(uuidBytes).toLowerCase().replace(/-/g, '');
      const expected = (env.UUID || '').toLowerCase().replace(/-/g, '');
      if (uuidStr !== expected) { ws.send(new Uint8Array([0, 0]).buffer); ws.close(1011, 'uuid mismatch'); return; }
      let off = 17;
      const addonsLen = data[off]; off += 1;
      if (addonsLen > 0) off += addonsLen;
      if (data.length <= off) { ws.send(new Uint8Array([0, 0]).buffer); ws.close(1011, 'bad header'); return; }
      const cmd = data[off]; off += 1;
      if (cmd !== 1) { ws.send(new Uint8Array([0, 0]).buffer); ws.close(1011, 'not tcp'); return; }
      if (data.length < off + 3) { ws.send(new Uint8Array([0, 0]).buffer); ws.close(1011, 'bad port'); return; }
      const port = (data[off] << 8) | data[off + 1]; off += 2;
      const atyp = data[off]; off += 1;
      let host = '';
      if (atyp === 1) {
        if (data.length < off + 4) { ws.send(new Uint8Array([0, 0]).buffer); ws.close(1011); return; }
        host = data[off] + '.' + data[off + 1] + '.' + data[off + 2] + '.' + data[off + 3];
        off += 4;
      } else if (atyp === 2) {
        const len = data[off]; off += 1;
        host = decoder.decode(data.slice(off, off + len));
        off += len;
      } else if (atyp === 3) {
        host = decoder.decode(data.slice(off));
        off = data.length;
      } else { ws.send(new Uint8Array([0, 0]).buffer); ws.close(1011, 'bad atyp'); return; }

      const firstPayload = data.slice(off);
      try {
        const conn = await Promise.race([
          connect({ hostname: host, port: port }),
          new Promise((_, reject) => setTimeout(() => reject(new Error('connect timeout')), 10000))
        ]);
        writer = conn.writable.getWriter();
        opened = true;
        ws.send(new Uint8Array([0, 0]).buffer);
        if (firstPayload.length > 0) await writer.write(firstPayload);
        pump(conn.readable, ws);
      } catch (e) {
        ws.send(new Uint8Array([0, 0]).buffer);
        ws.close(1011, 'connect fail: ' + (e && e.message ? e.message.slice(0, 30) : ''));
        return;
      }
    } else {
      if (writer && data.length > 0) {
        try { await writer.write(data); } catch (e) { try { ws.close(); } catch (e2) {} }
      }
    }
  });

  ws.addEventListener('close', () => {
    if (writer) { try { writer.close(); } catch (e) {} }
  });

  ws.addEventListener('error', () => {
    if (writer) { try { writer.close(); } catch (e) {} }
  });
}

async function pump(readable, ws) {
  const reader = readable.getReader();
  try {
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;
      const chunk = value instanceof Uint8Array ? value : new Uint8Array(value);
      const MAX = 16384;
      for (let i = 0; i < chunk.length; i += MAX) {
        const piece = chunk.slice(i, Math.min(i + MAX, chunk.length));
        const buf = piece.buffer.slice(piece.byteOffset, piece.byteOffset + piece.byteLength);
        ws.send(buf);
      }
    }
  } catch (e) {}
  try { ws.close(1000, 'done'); } catch (e) {}
}
```

---

## 参考链接

- edgetunnel 原项目：bbs.binmt.cc/thread-175318-1-1.html（MT 论坛帖内下载）
- CF Pages 直传文档：developers.cloudflare.com/pages/get-started/direct-upload/
- TCP sockets：developers.cloudflare.com/workers/runtime-apis/tcp-sockets/
