# Cloudflare Pages VLESS 代理部署教程（2026-10 全流程实测）

> 目标：免费部署可用 Google/YouTube/ChatGPT 的 VLESS 节点
> 方案：甬哥 Cloudflare-vless-trojan V26.9.14（Pages 直传 + 内置 ProxyIP 套壳）
> 本教程记录完整命令与全部踩坑，照着走可少走 80% 弯路

---

## 一、方案选型（为什么选它）

| 方案 | 结论 |
|---|---|
| cmliu/edgetunnel | ❌ 原版已进 CF 指纹库，部署即封；ProxyIP 对 ChatGPT 返回 403 challenge |
| 自写 VLESS Worker | ✅ Google 可用；❌ ChatGPT 无解（OpenAI 封杀 CF 数据中心出站 IP） |
| **甬哥 Cloudflare-vless-trojan V26.9.14** | ✅ **Google + ChatGPT 均可用**（内置 ProxyIP + nat64 套壳） |

关键点：**免费 CF 方案解锁 ChatGPT 的核心是"内置 ProxyIP 套壳"**，普通 Workers 直连必然被 OpenAI 拒。

---

## 二、前置准备

- 一个**全新邮箱**（建议新注册，别用绑定过 CF 账号的）
- 一台能跑 curl 的机器（本教程命令均可用 curl 执行）
- 手机装 CMFA（Clash Meta for Android）或 v2rayNG
- CFSTAPP（安卓版测速 APP，GitHub: Hsia97/CFSTAPP）

---

## 三、注册新 CF 账号（坑最多的一步）

### 3.1 注册
1. 手机浏览器打开 `https://dash.cloudflare.com/sign-up`
2. 填邮箱 + 密码（要求：8+ 字符、含数字、含特殊字符）
3. **人机验证必须真人完成**——自动化浏览器（WebView）中 Turnstile 无法渲染（控制台报 "No available adapters"），页面空白
4. 点注册 → 邮箱收验证邮件

### 3.2 邮箱验证（踩坑重灾区）
- 邮件里**底部 "Resources" 小字链接是帮助文档页**，不是验证链接！点它只会打开文档，白忙一场
- 要点正文里的**蓝色大按钮** "Verify email address"（在 "If this was you, verify your email address below." 下方）
- 如果按钮显示不出来（QQ 邮箱拦截图片）：长按按钮文字 → "复制链接地址" → 粘贴到浏览器打开
- **链接打开后若跳登录页，必须登录账号**（光点链接不登录 = 没验证）
- 验证成功的判断标准：API `/api/v4/user` 返回 `email_verified: true`
- 邮箱未验证时，创建 API Token 会报错：`{"code":1211,"message":"Please verify your email."}`

### 3.3 同浏览器旧账号问题
- 若浏览器已登录旧账号，sign-up 页会直接跳转旧账号主页或空白页
- 解决：先访问 `https://dash.cloudflare.com/logout` 登出，再访问 sign-up

---

## 四、下载代码并做指纹变异（防封核心！）

### 4.1 下载
```bash
# raw 域名比 codeload(zip) 稳定，zip 经常下载失败
curl -s "https://raw.githubusercontent.com/yonggekkk/Cloudflare-vless-trojan/main/Vless_workers_pages/_worker.js" -o _worker.js
```
- 文件约 610KB，4 行，结构：头部明文配置 + obfuscator 混淆主体
- `_worker明.js` 是未混淆版，但**不要用**（说明文件明确：仅混淆版可用）

### 4.2 指纹变异（不改必被封）
原版文件头部有明文配置行，必须个性化：
```js
let userID = "86c50e3a-...";  // 默认万人骑UUID → 必须改成自己的
const proxyIPs = ["pyip.ygkkk.dpdns.org"];  // 内置ProxyIP，保留
```

变异操作（全部安全，已验证）：
1. **替换 UUID**：`86c50e3a-5b87-49dd-bd20-03c7f2735e40` → 自己生成的 UUID（`python3 -c "import uuid; print(uuid.uuid4())"`）
2. **改写头部注释**（去品牌痕迹）：如 `//可以把...改为你自定义的uuid` → `// instance key, keep private`
3. **尾部追加随机填充**：`\n/*<300字节随机base64>*/\n;0\n`——改变文件大小与哈希
4. **语法校验**：`node --check _worker.js` 必须输出 SYNTAX_OK
5. 确认 SHA256 与原始文件不同

> ⚠️ 只改 UUID 不动混淆主体是安全的；不要尝试修改混淆代码内部（极易破坏运行时逻辑）

---

## 五、创建 API Token

在 CF 控制台（已登录新账号）的开发者工具里执行：
```js
fetch('/api/v4/user/tokens', {
  method:'POST', credentials:'include',
  headers:{'Content-Type':'application/json'},
  body: JSON.stringify({
    name:'pages-deploy',
    policies:[{effect:'allow',
      resources:{'com.cloudflare.api.account.*':'*'},
      permission_groups:[
        {id:'8d28297797f24fb8a0c332fe0866ec89'}, // Pages Write
        {id:'e247aedd66bd41cc9193af0213416666'}, // Pages Read
        {id:'e086da7e2179491d91ee5f35b3ca210a'}, // Workers Scripts Write
        {id:'1a71c399035b4950a1bd1466bbe4f420'}  // Workers Scripts Read
      ]}]
  })
})
```
- permission_groups ID 是**全局的**（各账号相同），可在 `/api/v4/user/tokens/permission_groups` 查询
- resources 必须写 `'com.cloudflare.api.account.*':'*'`（写 `account.zone.*` 会报 1001 权限检查失败）
- 保存返回的 token（cfut_ 开头，只显示一次）

---

## 六、Pages 直传部署（API 全链路）

### 6.1 创建项目
```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$AID/pages/projects" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"name":"pg-xxxx","production_branch":"main"}'
```
（或 `npx wrangler pages project create pg-xxxx --production-branch=main`）

### 6.2 三步直传（关键！wrangler 走不通）
**为什么不用 wrangler：** `wrangler pages deploy` 会用 esbuild 构建，对 obfuscator 混淆代码报 7 个错误（`Cannot assign to "G" because it is a constant`——**原版文件同样报错**，是 esbuild 与混淆代码不兼容，不是变异问题）。wrangler 4.152 已移除 `--script-path` 参数。必须走 Direct Upload API：

**第 1 步：拿 upload-token（JWT）**
```bash
curl -s "https://api.cloudflare.com/client/v4/accounts/$AID/pages/projects/$PROJ/upload-token" \
  -H "Authorization: Bearer $TOKEN"
# → result.jwt（含 projectNamespace 字段，30分钟有效）
```

**第 2 步：上传资产（拿内容 hash）**
```bash
HASH=$(md5sum _worker.js | cut -d' ' -f1)
B64=$(base64 -w0 _worker.js)
curl -s -X POST "https://api.cloudflare.com/client/v4/pages/assets/upload" \
  -H "Authorization: Bearer $JWT" -H "Content-Type: application/json" \
  -d "[{\"base64\":true,\"key\":\"$HASH\",\"metadata\":{\"contentType\":\"application/javascript\"},\"value\":\"$B64\"}]"
# ⚠️ 必须用 api.cloudflare.com 主域名！
# api.pages.cloudflare.com 会报 "Missing assetNamespace"（JWT 里的 projectNamespace 参数实际不用传）
```

**第 3 步：创建部署（multipart）**
```bash
curl -s -X POST "https://api.cloudflare.com/client/v4/accounts/$AID/pages/projects/$PROJ/deployments" \
  -H "Authorization: Bearer $TOKEN" \
  -F "branch=main" \
  -F "manifest={\"_worker.js\":\"$HASH\"}" \
  -F "_worker.js=@_worker.js;type=application/javascript"
```

### 6.3 查询部署状态
```bash
curl -s ".../pages/projects/$PROJ/deployments/$DEPLOY_ID" -H "Authorization: Bearer $TOKEN"
# latest_stage.name=deploy 且 status=success 即完成（一般 <1 分钟）
```

---

## 七、验证部署

```bash
# 根路径：返回请求信息 JSON（正常）
curl -s "https://$PROJ.pages.dev/"

# 配置页（显示节点链接与订阅链接）
curl -s "https://$PROJ.pages.dev/$UUID"

# Clash 订阅
curl -s "https://$PROJ.pages.dev/$UUID/pcl"
# Sing-box 订阅: /psb    通用订阅: /pty
```

配置页输出的关键参数：
- WS 路径：`/?ed=2560`
- TLS 端口：443 / 8443 / 2053 / 2083 / 2087 / 2096
- 优选域名：yg8~yg13.ygkkk.dpdns.org（可替换为测速 IP）
- 内置 ProxyIP 运行中：pyip.ygkkk.dpdns.org

---

## 八、客户端接入

### CMFA（Clash Meta）
1. 下载订阅 `/$UUID/pcl` 存为 yaml 文件
2. CMFA → 新建配置 → 从文件导入
3. 启用配置 → 开系统代理
4. 测试 Google + ChatGPT

### 换优选 IP（随时提速）
1. **关代理** → 打开 CFSTAPP 测速
2. 取 0 丢包 + 速度最高的 3-4 个 IP
3. 编辑订阅文件，替换前几个节点的 `server:` 值
4. 建议保留最后 1-2 个节点用 yg 优选域名兜底
5. CMFA 重新导入生效

### 快速验证 IP 可用性（WS 升级测试）
```python
# TLS 连接 IP:port，SNI 与 Host 都填 $PROJ.pages.dev，path=/?ed=2560
# 返回 HTTP/1.1 101 Switching Protocols 即链路正常
```

---

## 九、踩坑清单（血泪总结）

| # | 坑 | 解决 |
|---|---|---|
| 1 | Turnstile 在自动化浏览器中不渲染 | 注册必须真人操作 |
| 2 | 验证邮件底部链接是帮助文档 | 点正文蓝色按钮，或复制链接 |
| 3 | 邮箱未验证 → API 报 1211 | 点完链接还要登录账号 |
| 4 | 同浏览器旧账号登录态干扰 sign-up | 先 /logout |
| 5 | codeload 下载 zip 失败 | 用 raw.githubusercontent.com |
| 6 | 不改指纹直接部署 → 秒封 | 必改 UUID + 注释 + 尾部填充 |
| 7 | wrangler pages deploy esbuild 报错 | 与变异无关（原版也报），走 Direct Upload API |
| 8 | Workers --no-bundle 部署后 1101 | CF 脚本级动态检测，首次成功后续全挂，无日志；放弃 Workers 路线 |
| 9 | assets/upload 用 api.pages.cloudflare.com 报 Missing assetNamespace | 用 api.cloudflare.com 主域名 |
| 10 | API token resources 写 account.zone.* 报 1001 | 写 account.* |
| 11 | 自写脚本 VLESS 帧被 close | 不必纠结，直接用官方订阅 + 真实客户端验证 |
| 12 | 快速连发测试触发网络临时封锁 | 控制频率，测试间隔拉长 |

---

## 十、风控与维护注意事项

1. **一个账号只放一个代理项目**，别反复部署同类代码（旧账号被封 4 个项目 = 前车之鉴）
2. 项目名**避免** proxy/vless/vpn/fq 等敏感词，用无意义随机名
3. ChatGPT 解锁有时效性（CF/OpenAI 持续封堵），失效后换 ProxyIP 池或等新版脚本
4. 订阅链接含 UUID，**别公开分享**（分享 = 把节点送人）
5. 换 IP 只改 `server` 字段，servername/Host/uuid/path 全部不动
6. 若整个项目被封：换新邮箱注册新账号，重复本教程第三~八节，代码重新做一次指纹变异

---

*文档版本：2026-10-25 · 方案 V26.9.14 · 全流程实测通过*
