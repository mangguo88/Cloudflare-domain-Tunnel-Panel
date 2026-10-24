# Cloudflare Tunnel 管理面板（修改版）

> 本项目基于 [eooce/Cloudflare-Tunnel-Panel](https://github.com/eooce/Cloudflare-Tunnel-Panel) 修改，遵循原项目 GPL-3.0 许可证，LICENSE 文件保持原样。
> 感谢佬王。

## 本版主要改动（修1–修4）

- 仪表板改为内嵌统一展示框，统计卡（总域名 / 总 Tunnel / 在线 / 离线 / 总 Worker / 在线 Worker / 离线 Worker）点击切换展示内容
- 后端新增 `POST /api/worker_check` 服务端批量探测绑定域名有效性
- 新增每周 Worker 巡检 `runWorkerInspection`（7 天门控、离线时 Telegram 汇总通知）
- 侧栏新增「Worker 管理」视图（按域名分组，Worker/Pages 带管理链接跳转 Cloudflare 后台）

自查标记：`_worker.js` 共 2743 行，搜 `runWorkerInspection` 和 `#v-workers` 可验证为本版。

---

# Cloudflare Tunnel 管理面板

Cloudflare Worker部署的Argo tunnel网页管理面板，用于管理多账户下的 Argo Tunnel：新建/编辑/删除隧道、查看连接 Token、增删域名路由、查看在线状态，并可选地把隧道上线/离线事件通过 Telegram 推送提醒。

整个项目只有 **一个文件**：`_worker.js`（Worker 代码 + 内嵌管理页面，无需构建、无外部依赖）,之前部署了旧的版本，只需要复制_worker.js里的代码替换部署即可，其他的都不变

## 功能一览

- **密码登录**：访问面板需输入密码，会话 Cookie 签名有效期 7 天，登录接口有防爆破延时。
- **多账号管理**：顶栏下拉切换 Cloudflare 账号；可在"账号管理"里添加多个账号（各自 API Token，验证后存 D1）；环境变量配置的账号作为默认账号。隧道列表、路由、提醒规则、定时检查均按账号隔离。
- **隧道管理**：列表展示所有隧道及状态（在线/降级/离线/未启用 + 运行时长）、搜索过滤、查看启动 Token、修改名称与本地服务地址（多路由可选要编辑的那条）、删除隧道。
- **新建隧道**：选择域名 (Zone)、子域名、可选路径、本地服务地址（如 `http://localhost:8080`、`tcp://localhost:22`），自动创建隧道并自动配置 DNS CNAME 记录。
- **多路由管理**：在已有隧道上增删"域名 + 路径 → 服务"的 ingress 规则（每条隧道最多 60 条），自动处理 DNS，冲突时拒绝写入。
- **Telegram 上下线提醒**：状态**发生变化**才推送（上线一次、离线一次），可按隧道逐个开关（**默认关闭**）；支持多个接收者（私聊/群组/频道）、暂停/恢复推送。
- **PWA**：可安装到桌面/手机主屏，带离线壳页面。

## 快速部署（Cloudflare Dashboard）

1. 登录 [Cloudflare Dashboard](https://dash.cloudflare.com) → **Workers & Pages** → **Create** → 创建 Worker，把 `_worker.js` 内容全部粘贴进去，保存。
2. **【非必须】绑定 D1数据库管理多账号**：参考下方D1数据库的设置
3. **【必须】配置环境变量**：Settings → Variables and Secrets，添加 `CF_API_TOKEN`、`CF_ACCOUNT_ID`、`PASSWORD`（详见下表）。
4. 部署后打开 `https://<Worker 域名>`，输入密码登录即可。


## 环境变量

| 变量 | 必填 | 说明 |
| --- | --- | --- |
| `CF_API_TOKEN` | ✅ | API 令牌。权限见下方「API Token 权限配置」 |
| `CF_ACCOUNT_ID` | ✅ | Cloudflare 账户 ID,创建token时会显示 |
| `PASSWORD` | ✅ | 面板登录密码，至少 6 位 |
| `SESSION_SECRET` | 建议 | 会话签名密钥，一串随机字符。不设置时由密码派生（改密码会使所有人重新登录） |
| `TUNNEL_STATE` | 可选 | **KV 命名空间绑定**（Bindings，不是环境变量），存储登录外的全部状态与提醒配置。**推荐改用 D1**：绑定变量名 `TUNNEL_DB`（建表见下），两者同时绑定时 D1 优先 |
| `DEFAULT_SERVICE` | 可选 | 新建/编辑隧道时的默认本地服务地址，如 `http://localhost:8080` |
| `TG_BOT_TOKEN` | 可选 | Telegram 机器人 Token（环境变量兜底；也可在面板里保存到 D1） |
| `TG_CHAT_ID` | 可选 | 默认接收者 chat_id（环境变量兜底） |

## API Token 权限配置

创建 API Token（My Profile → API Tokens → Create Token → Custom token）时，按下面两组勾选。注意区分**账户级（整个账户）**和**区域级（所有域名/Zone Resources 选 All zones）**两组，名字相近但作用完全不同：

**账户级（Account Resources 选整个账户）：**

| 分组 | 权限项 | 级别 |
| --- | --- | --- |
| DNS & Zones | Account DNS Settings | Read + Edit |
| DNS & Zones | DNS View | Read + Edit |
| Cloudflare Tunnel（或 Argo Tunnel） | Cloudflare Tunnel | Read + Edit |

**区域级（Zone Resources 选"所有域名 / All zones"）：**

| 分组 | 权限项 | 级别 |
| --- | --- | --- |
| DNS & Zones | DNS | Read + Edit |
| DNS & Zones | Zone | Read + Edit |
| DNS & Zones | Zone DNS Settings | Read + Edit |
| DNS & Zones | Zone Settings | Read + Edit |

> 最小可用集其实只需：账户级 `Cloudflare Tunnel → Edit` + 区域级 `DNS → Read/Edit`、`Zone → Read`；多勾的项是无害的冗余。常见错误是只配了账户级的 Account DNS Settings/DNS View（管的是账户 DNS 全局设置，不是 Zone 里的 DNS 记录），会导致面板报"该域名已存在 DNS 记录"或删除隧道后 DNS 清理失败——**区域级的 `DNS → Edit` 必须有**。


## 创建令牌图片展示
<img width="1640" height="843" alt="image" src="https://github.com/user-attachments/assets/3e8b12fa-8afb-40cf-ad9d-a8da0bcfe390" />
<img width="1277" height="787" alt="image" src="https://github.com/user-attachments/assets/766fa369-5528-4c9c-b312-4034edd4e889" />
<img width="1257" height="589" alt="image" src="https://github.com/user-attachments/assets/1923be3a-5d4d-42be-9ba8-9337d2324463" />
<img width="1479" height="911" alt="image" src="https://github.com/user-attachments/assets/9314e000-4749-4808-8bc6-9bd8a075e304" />
<img width="1242" height="591" alt="image" src="https://github.com/user-attachments/assets/67070035-4fd9-44cf-95e7-7a96b04057d8" />
<img width="1203" height="666" alt="image" src="https://github.com/user-attachments/assets/e9ddd190-e25d-4d51-8a1d-d527bf5ccdaa" />
<img width="1368" height="516" alt="image" src="https://github.com/user-attachments/assets/c105016f-6e7a-4c8e-acae-8e18dde988dc" />
<img width="1486" height="906" alt="image" src="https://github.com/user-attachments/assets/5e840344-e09c-4c7a-b408-514c8d1431f0" />
<img width="910" height="670" alt="image" src="https://github.com/user-attachments/assets/6fdf5d00-a238-4f37-9c46-32cee9a9077c" />
<img width="1103" height="864" alt="image" src="https://github.com/user-attachments/assets/354ee277-6e3f-45da-b69d-6e5f7599823c" />

## 注意：在面板增加多账号也是填入以上对应的令牌变量

## 存储选择：D1（推荐）或 KV

KV 有每日写入次数限制且存在最终一致延迟（写后立刻读可能读到旧值），本面板读写频繁，推荐改用 D1：

1. Dashboard → Storage & Databases → D1 → Create，建一个数据库,名称随意，例如：**`tunnel-state`**；
2. 点击控制台，在数据库执行框里运行以下命令：

   ```sql
   CREATE TABLE IF NOT EXISTS state (k TEXT PRIMARY KEY, v TEXT NOT NULL);
   ```

<img width="1357" height="685" alt="image" src="https://github.com/user-attachments/assets/ee0105e1-9fe4-4df6-9b32-df0687c80cac" />


3. Worker → Settings → Bindings → D1 Database，变量名填 **`TUNNEL_DB`**；
4. 重新部署。未绑定 D1 时自动回退到 KV（`TUNNEL_STATE`），行为不变；两者同时绑定则 D1 优先。

数据自动互通：D1 和 KV 里存的是同样的键值（`meta:settings`、`tunnel:state:<账号>:<隧道>`、`chat:*`），从 KV 迁到 D1 时如需保留旧提醒配置，可在 D1 控制台执行 `INSERT OR REPLACE INTO state SELECT k, v FROM ...`（或重新在面板里保存一次 Telegram 配置即可，提醒开关重新打开一次）。

## 多账号管理

- 环境变量 `CF_API_TOKEN` + `CF_ACCOUNT_ID` 配置的账号永远存在，名为"默认账号"，不可删除。
- 顶栏下拉选择账号，切换后隧道列表、域名、路由、提醒开关随账号切换；选择记忆在 KV，下次打开保持。
- 点击右上角的**账号管理**：填名称、账户ID该账户的 API Token，**先验证再保存**（Token 只存 D1，不回显到页面）。
- 提醒状态（上下线记录、20 秒防抖窗口）按账号隔离，互不串扰；Cron 定时检查会遍历**所有**账号。
- 未绑定 KV 时多账号功能不可用（无法保存），回退为环境变量单账号。

## Telegram 提醒配置

1. 在 Telegram 找 [@BotFather](https://t.me/BotFather)，`/newbot` 创建机器人，拿到 **Bot Token**。
2. **想让谁收提醒，就让谁先给机器人发一条消息（或 `/start`）**——机器人无法主动私聊从未互动过的用户。
3. 获取接收者的 chat_id：
   - **个人私聊**：给 [@userinfobot](https://t.me/userinfobot) 发任意消息，它会回复你的**正数**数字 ID；
   - **群组**：把机器人拉进群，群的 ID 为负数（超级群组通常 `-100` 开头）；
   - **公开频道/群组**：直接填 `@用户名` 也可以，面板会解析成真实 chat_id 存入 KV。
4. 打开面板 → **Telegram 配置**：填入 chat_id（接收者，**不是机器人的 ID**）和 Bot Token，点"**测试发送**"确认可达，再点"**保存到 KV**"。
5. 回到 **Tunnel 管理**，把需要监控的隧道的"🔕 提醒关"点成"🔔 提醒开"。

> ⚠️ 常见误区：chat_id 是"谁来**接收**消息"（你的用户 ID / 群 ID），Bot Token 才是机器人的。填机器人自己的 ID 会提示"机器人不能给自己发消息"。

### 提醒机制

- **默认关闭**：每条隧道需手动开启，开启状态保存在 KV。
- 只在状态**变化**时发送一次（上线一次、离线一次），同一状态不重复提醒；首次记录到状态时不发送，部署后不会刷屏。
- 检查时机：每次打开/刷新面板时检查一次，面板连续刷新最少间隔 20 秒（防止把网络抖动看成多次上下线）。
- **无人值守检查**：给 Worker 添加 Cron 触发器（如 `*/5 * * * *`），代码中的 `scheduled` 处理会自动执行同样的检查，无需有人打开面板。
- **暂停推送**：面板可暂停（限时或无限期），期间状态照常记录，只是不发消息。

## API 接口概览

除 `/api/login` 外，所有接口需登录（Cookie 会话）。均为 `POST` + JSON：

| 路径 | 说明 |
| --- | --- |
| `/api/login` `/api/logout` | 登录 / 退出 |
| `/api/init` | 域名 (Zone) 列表、默认服务地址 |
| `/api/list` | 隧道列表 + 在线状态 + 提醒状态（同时触发一次检查） |
| `/api/create` `/api/update` `/api/delete` | 新建 / 编辑 / 删除隧道 |
| `/api/token` | 获取隧道启动 Token（cloudflared 用） |
| `/api/uptime` | 连接器运行时长（官方后台"运行时间"同源，用于在线徽章"在线 4d"） |
| `/api/detail` `/api/route` | 查看 / 增删隧道 ingress 路由 |
| `/api/alert` | 隧道提醒开关 (toggle) / 吞掉一次变化 (skip) |
| `/api/tg` | Telegram 配置：保存、测试、增删接收者、暂停/恢复 |
| `/api/acc` | 多账号：列表 (list) / 添加 (add, 先验证) / 删除 (del) / 切换 (switch) |

## 常见问题

- **"未绑定 KV 命名空间 TUNNEL_STATE"**：在 Worker → Settings → Bindings 里绑定 KV，变量名必须为 `TUNNEL_STATE`。
- **"找不到该 chat_id"**：私聊接收者没给机器人发过 `/start`；或把机器人 ID/正数当成了群 ID 填写。个人填正数用户 ID，群组填负数 ID。
- **"机器人 Token 无效"**：检查 Bot Token 是否复制完整、机器人是否被 BotFather 停用。
- **"隧道仍有活跃连接,请先停止 cloudflared"**：删除前需先在目标机器上停止 `cloudflared` 进程（Cloudflare 平台限制）。
- **"该域名已存在其他 DNS 记录"**：先到 Zone 的 DNS 页面删除冲突记录，或换个子域名。
- **"DNS 记录清理失败"（删除隧道/路由后提示）**：检查 API Token 是否有**区域级 Zone → DNS → Edit** 权限（见「API Token 权限配置」），账户级的 Account DNS Settings 不能替代它；toast 里会显示具体原因。
- **改密码后所有人掉线**：未设置 `SESSION_SECRET` 时会话密钥由密码派生；设置独立的 `SESSION_SECRET` 可避免。
- **之前是"默认开提醒"的旧版本，现在想全部关**：新版已改为默认关，并带一次性迁移，打开面板后旧数据会自动清成关闭状态。

## 安全说明

- 所有 Cloudflare API 调用都在 Worker 服务端完成，Token 不会下发到浏览器。
- 隧道启动 Token（可冒充隧道接管流量）仅在登录后按需单独获取。
- 登录密码比较使用固定时间算法，防止时序侧信道；连续失败有延时惩罚。
- 建议同时设置 `SESSION_SECRET`，并给 Worker 域名加上 Cloudflare Access 作为第二层防护（可选）。
