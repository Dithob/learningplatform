# 学习工作台 · 秋招备考

一个**零依赖的单文件打卡应用**：四个功能页签（今日 / 计划 / 统计 / 日志）、IndexedDB 本地存储、Pointer Events 拖拽排序，外加一层基于 GitHub Gist 的多设备云同步。

全部代码（HTML + CSS + JS）打包在**一个 HTML 文件**里——没有后端、没有框架、没有 `npm install`，双击即可运行，丢到任何静态服务器上也能运行。

| 属性 | 说明 |
| --- | --- |
| 形态 | 单文件应用（`dist/index.html`，约 76 KB） |
| 依赖 | 零外部依赖（不引 CDN，无构建步骤） |
| 本地存储 | IndexedDB（不可用时自动降级 localStorage） |
| 云同步 | GitHub Gist（secret gist，token 永不上云） |
| 运行环境 | 任意现代浏览器，移动端优先（375px 可用） |

---

## 目录

- [项目结构](#项目结构)
- [功能特性](#功能特性)
- [主要使用方法](#主要使用方法)
  - [1. 本地使用](#1-本地使用)
  - [2. 部署到线上](#2-部署到线上)
  - [3. 多设备云同步（5 步配置）](#3-多设备云同步5-步配置)
  - [4. 数据备份与迁移](#4-数据备份与迁移)
  - [常见问题](#常见问题)
- [技术实现思路](#技术实现思路)
  - [整体架构：三层分工](#整体架构三层分工)
  - [数据模型（IndexedDB 三张表）](#数据模型indexeddb-三张表)
  - [DataLayer：存储封装与降级](#datalayer存储封装与降级)
  - [Store：串行写队列](#store串行写队列)
  - [渲染层：四个页签](#渲染层四个页签)
  - [拖拽排序：Pointer Events 实现](#拖拽排序pointer-events-实现)
  - [云同步：GitHub Gist](#云同步github-gist)
  - [冲突保护：两个时间戳对账](#冲突保护两个时间戳对账)
  - [设计取舍与局限](#设计取舍与局限)
- [开发与维护](#开发与维护)

---

## 项目结构

```
learningplatform/
├── README.md                # 本文件：项目说明 + 使用方法 + 技术实现
├── .gitignore               # 忽略 .workbuddy/ 等工作区元数据
├── dist/                    # ★ 工作台本体（单文件，源文件 = 部署产物，零构建）
│   └── index.html           # 学习工作台（唯一真源，改这里）
├── docs/                    # 项目文档
│   ├── workbench-guide.html # 完整介绍页：使用方法 + 技术实现（与源码严格对齐）
│   └── design-notes.md      # 设计笔记：数据模型、交互要点、高风险点、验收清单
└── backup/                  # 历史版本存档（保留旧版，仅作追溯）
    └── index.html           # v1 旧版（2026-08-11，蓝紫主题「秋招备考打卡」）
```

**设计约定**

| 目录 | 职责 | 说明 |
| --- | --- | --- |
| `dist/` | 交付物 | 单文件应用无构建过程，`index.html` 既是源码也是部署产物 |
| `docs/` | 文档 | 介绍页 + 设计笔记，跟随代码演进 |
| `backup/` | 历史存档 | 已被 git 记录替代；保留 v1 仅为对照视觉与早期逻辑 |

> **为什么只有一个 HTML 文件？** 计算与存储全部发生在浏览器里——服务器（如果有的话）只负责把 HTML 原样递给访问者，此后即退场。这让项目可以零成本部署、无限离线可用。

---

## 功能特性

| 页签 | 能力 |
| --- | --- |
| **今日** | 4 个打卡组（早上算法 / 午后主攻 / 晚上面经投递 / 睡前复盘，每组独立进度条与时间段）+ 3 个复盘文本框（反思 / 心得 / 调整），支持任意日期回填补卡 |
| **计划** | 打卡项增删改与归档、Pointer Events 拖拽排序（鼠标 + 触屏统一，拖动显示绿色插入指示线）、一键重置种子数据；附计划总览（四阶段时间线 + 50/30/20 精力分配） |
| **统计** | 打卡率、连续打卡天数、GitHub 风格热力图、周/月完成率 |
| **日志** | 按类型（反思 / 心得 / 调整）筛选的历史记录流，每条带彩色左边框，可回看、可编辑历史 |
| **全局** | JSON / CSV 一键导出、JSON 导入恢复、多设备 Gist 云同步（8 秒防抖自动上传 + 时间戳冲突保护） |

---

## 主要使用方法

### 1. 本地使用

**双击 `dist/index.html` 即可使用**，无需任何环境。数据写入浏览器 IndexedDB，关闭页面、重启电脑都不丢。

> ⚠️ 本地 `file://` 打开与线上网址打开是浏览器眼中的两个「源」，各自拥有独立的空仓库。换源必须用「导出 JSON → 导入」迁移一次（详见[数据备份与迁移](#4-数据备份与迁移)），这不是 bug，是浏览器安全模型的设计。

### 2. 部署到线上

任意能托管静态文件的服务器都可以：

- **CloudStudio 沙箱**（推荐，当前分享链接基于此）：把 `dist/index.html` 部署到沙箱，得到一个固定域名，手机 / 平板 / 电脑收藏同一个地址即可随时打开；
- 其他任意静态托管（GitHub Pages、对象存储 CDN 等）直接上传 `index.html` 即可。

沙箱链接失效时重新部署一次即可恢复，**打卡数据不受影响**——数据在本机 IndexedDB 和云端 Gist，不在沙箱里。

### 3. 多设备云同步（5 步配置）

本质困难：IndexedDB 按「协议 + 域名 + 端口」隔离，每台设备的浏览器各有一个独立仓库。云同步让所有设备围绕同一个 Gist 数据文件读写。配置一次，之后全自动：

| 步骤 | 操作 |
| --- | --- |
| ① 生成 token | GitHub → Settings → Developer settings → Personal access tokens → **Tokens (classic)** → Generate new token，勾选 `gist` 权限，复制 `ghp_` 开头字符串 |
| ② 旧设备启用 | 在**有打卡数据**的设备打开工作台 → 计划页 → 「云同步」卡片 → 粘贴 token → 点「启用云同步」（首次上传自动创建 secret gist） |
| ③ 迁移历史数据（仅换源时） | 旧数据若在 `file://` 版本里：先「导出 JSON」→ 线上「导入恢复」→ 自动上传到云端 |
| ④ 其他设备接入 | 打开同一网址 → 粘贴同一 token → 点一次「从云端下载」，之后全自动同步 |
| ⑤ 日常使用 | 正常打卡即可。每次改动 8 秒后自动上传；打开另一台设备时如有云端新数据会提示先下载。token 建议定期在 GitHub 后台轮换 |

**同步行为一览**

| 操作 | 触发时机 | 说明 |
| --- | --- | --- |
| 自动上传 | 用户改动后 **8 秒防抖** | 连续勾选只合并为一次 PATCH；上传前先 GET 云端对账 |
| 手动上传 / 下载 | 计划页「云同步」卡片按钮 | 上传覆盖 / 下载覆盖均有 confirm 双重确认 |
| 断开同步 | 卡片「断开」按钮 | 云端 Gist 保留作为备份，仅清除本机凭证 |

### 4. 数据备份与迁移

- **备份**：右上角导出按钮 → 导出 JSON（全量）或 CSV（每日记录）；
- **迁移到新源 / 新设备**：旧处「导出 JSON」→ 新处「导入恢复」→ 完成（导入的数据会自动触发一次云同步上传）。

### 常见问题

| 问题 | 处理 |
| --- | --- |
| GitHub API 国内访问慢 / 失败 | 重试即可；自动上传失败不丢数据（数据仍在本机） |
| 提示「云端有较新数据，已暂停自动上传」 | 先「从云端下载」再继续操作——说明另一台设备推过你没见过的版本 |
| 沙箱链接失效 | 重新部署一次 `dist/index.html`，数据不受影响 |
| 想彻底重来 | 计划页可清空数据；或导出备份后在浏览器开发者工具中清除该源站点数据 |

---

## 技术实现思路

> 以下内容与 `docs/workbench-guide.html` 一致，代码片段摘自真实源码（`dist/index.html`）。

### 整体架构：三层分工

「分发」与「运行」彻底分离，每一层只做一件事：

| 层级 | 角色 | 实现 |
| --- | --- | --- |
| **分发层** | 固定网址 | CloudStudio 沙箱（或其他静态服务器）把 `index.html` 原样递给访问者，此后退场，不执行任何业务逻辑 |
| **运行层** | 浏览器单文件应用 | 渲染、打卡、统计、拖拽全部在本机执行。内部三块：`renderXxx` 渲染函数（4 个页签）、`Store` 串行写队列（所有写操作必经之路）、`CloudSync` 同步模块 |
| **存储层** | IndexedDB + Gist | 本地三张仓库（`planItems` / `days` / `meta`）；云端一个 secret Gist 的 JSON 文件，作为多设备共享的「数据真相」 |

一次完整请求的生命周期：

```
手机打开网址 → 沙箱返回 HTML → 浏览器本地运行（渲染/打卡全在本机）→ 写入本机 IndexedDB
    → 写队列盖时间戳 → 8s 防抖到点 → GET 云端对账 → PATCH 到 Gist → 更新 csSyncedAt
```

### 数据模型（IndexedDB 三张表）

库名 `autumn-recruit-checkin`（v1），`onupgradeneeded` 中建表：

| 表 | keyPath | 字段 | 说明 |
| --- | --- | --- | --- |
| `planItems` | `id` | `{ id, groupId, title, sort, archived }` | 打卡项；`groupId ∈ morning / afternoon / evening / bedtime`；删除用软删除 `archived:true`（保历史可回溯）；索引 `groupId` |
| `days` | `date`（`YYYY-MM-DD`） | `{ date, doneIds: [], reflect, insight, adjust, updatedAt }` | 每日打卡；`doneIds` 存 **item_id**（不存快照，改标题/排序零影响）；upsert + Set 去重保证幂等 |
| `meta` | `key` | `{ key, value }` | 配置与时间戳：`seedVersion`（种子去重播种）、`lastLocalWrite`、`csToken` / `csGistId` / `csSyncedAt` / `csAutoPush` |

日期一律用本地时区 `getFullYear / getMonth / getDate` 拼 `YYYY-MM-DD`，**禁用 `toISOString()`**（UTC 跨日错位）。

### DataLayer：存储封装与降级

`DataLayer` 封装全部 IndexedDB 操作（init / tx / getAll / put / clear / putAll / 导入导出）。关键设计：

- `window.indexedDB` 不可用或打开失败时，**静默降级到 localStorage**（键前缀 `wb_arc_`，约 5MB 上限），上层调用方无感；
- 三张表按 store 名 + key 语义统一读写，`tx()` 用 Promise 包装事务的 `oncomplete / onerror / onabort`；
- 数据库版本管理走 `onupgradeneeded`，兼容旧库增量建表。

### Store：串行写队列

IndexedDB 所有操作都是异步事务，打卡、改计划、拖拽排序并发触发会产生竞态。因此**所有写操作都经过 `Store.enqueue` 排队串行执行**：

```js
// 所有写操作的必经之路：串行队列 + 时间戳 + 同步钩子
function enqueue(fn, silent){
  chain = chain.then(fn).then(res => {
    if(silent) return res;
    state.meta.lastLocalWrite = Date.now();   // ① 记录最后修改时间
    return DataLayer.put('meta', { key:'lastLocalWrite', value: state.meta.lastLocalWrite }).then(() => res);
  }).then(res => {
    if(!silent && typeof CloudSync !== 'undefined' && CloudSync.scheduleAutoPush) CloudSync.scheduleAutoPush(); // ② 启动同步定时器
    return res;
  }).catch(e => console.error('[Store] 写队列错误', e));
  return chain;
}
```

这个队列顺手做了两件事：给每次用户写入盖上 `lastLocalWrite` 时间戳（冲突保护要用），并启动云同步的 8 秒防抖定时器。`silent` 参数是同步机制的关键——详见[冲突保护](#冲突保护两个时间戳对账)。

### 渲染层：四个页签

纯函数式渲染：`renderToday()` / `renderPlan()` / `renderStats()` / `renderLog()` 各自读取 `Store.state` 生成 DOM 字符串注入对应 `section`，数据变更后调用对应 render 刷新。统计口径：

- **连续天数**：当天至少完成 1 项算打卡；从今天反向遍历，遇空当天即止；
- **热力图**：以今天所在周为右端、按周回排 26 周，完成率 4 档着色（`.c1~c3`），今天描边高亮；
- **今日打卡**：`doneIds` 命中即勾选，`saveDay` 合并单事务写库，空记录（无勾选且三文本框全空）不落库。

### 拖拽排序：Pointer Events 实现

早期版本用 HTML5 原生 `draggable`，在移动端完全失效且无视觉反馈。现实现用 **Pointer Events**（鼠标 + 触屏统一），要点：

1. **5px 移动阈值**：`pointerdown` 不立即进入拖动，记录起点；位移 > 5px 才标记 `started=true`，避免误触 click；
2. **`getBoundingClientRect` 遍历判目标**：`findTargetRow(clientY, excludeRow)` 遍历同组 `.plan-row`（`.closest('.card')` 判同组，跨组不算），优先返回指针 Y 落在 `[top, bottom]` 内的行；落进行间缝隙时回退到 Y 距离最近的行——彻底替代早期失效的 `elementFromPoint` 方案；
3. **`touch-action: none`**：让 JS 全权处理拖动（`pan-y` 会被浏览器先吃掉，触屏收不到 pointermove）；
4. **松手定位**：`onPointerUp` 按 `clientY` 与目标行中线比较，决定 `before` / `after`，`reorderItems(fromId, toId, position)` 把 `sort` 归一化为 0..N-1 持久化；
5. **视觉反馈**：源行 `.dragging` 半透明 + 灰底；目标行 `.drag-over-top / .drag-over-bottom` 用 `::before / ::after` 画 2px 翡翠色指示线 + 12% 光晕；
6. 按在 `button/input` 上时直接 return，保证 ↑↓ 编辑删除等点击正常触发。

### 云同步：GitHub Gist

纯前端直连第三方存储有一道硬门槛——**CORS**。选型对比：

| 候选 | 结果 | 原因 |
| --- | --- | --- |
| **api.github.com** | ✅ 入选 | 官方支持 CORS，浏览器可直接调用，免费，Gist 带完整 REST API |
| 坚果云 WebDAV | ❌ 被否 | 服务器不返回跨域头，纯前端无法直连 |
| 自建后端 | ❌ 不值得 | 能做，但违背「零依赖单文件」初衷 |

同步链路（`CloudSync` 模块）：

- **`verifyToken`**：`GET /gists?per_page=1` 校验 token（401 判无效）后写入 `csToken`；
- **`ensureGist`**：首次上传自动 `POST /gists` 创建 **secret gist**（`public: false`，文件 `workbench-data.json`），gistId 存本地 `meta.csGistId`；
- **`push`**：`PATCH /gists/{id}` 整包替换文件内容（`collectPayload()` 序列化），成功后更新 `csSyncedAt`；
- **`fetchRemote`**：`GET /gists/{id}` 拉取；内容被截断时走 `raw_url` 兜底；
- **`applyRemote`**：清库（planItems / days / meta）→ 批量导入云端数据 → 重新渲染 → 更新 `csSyncedAt`。

**凭证安全：token 永不上云。** GitHub token（`ghp_` 开头）只存在本机 `meta`，随 `Authorization: Bearer` 请求头发给 GitHub。打包同步数据时，**所有 `cs` 开头配置键（token、gistId、同步时间戳）被过滤**，永远不会被同步到云端或别的设备：

```js
meta: Object.keys(Store.state.meta)
  .filter(k => k.indexOf('cs') !== 0)   // 过滤所有凭证相关键
  .map(k => ({ key: k, value: Store.state.meta[k] }))
```

**自动上传：8 秒防抖。** 连续勾几个框如果每勾一次发一次 PATCH，既浪费请求又容易触发 GitHub 限流。写队列只负责启动 / 重置一个 8 秒定时器——连续操作不断重置，直到停下来 8 秒后才真正上传一次，一串操作合并成一次网络请求：

```js
function scheduleAutoPush(){
  clearTimeout(pushTimer);
  pushTimer = setTimeout(autoPushNow, 8000);  // 连续操作只保留最后一次
}
async function autoPushNow(){
  const c = cfg();
  if(!c.token || !c.autoPush || busy) return;
  if((Store.state.meta.lastLocalWrite||0) <= c.syncedAt) return; // 没有新改动
  const remote = await fetchRemote();           // 先 GET 云端对账
  if(remote && Date.parse(remote.exportedAt) > c.syncedAt){
    toast('云端有较新数据，已暂停自动上传，请手动「从云端下载」'); return;
  }
  await push();
}
```

### 冲突保护：两个时间戳对账

最关键的问题：手机电脑都快，谁的数据算准？系统维护两个时间戳，差值就是对账依据：

| 时间戳 | 含义 | 谁在维护 |
| --- | --- | --- |
| `lastLocalWrite` | 我最后一次**改数据**的时间 | 写队列，每次用户写入自动盖戳 |
| `csSyncedAt` | 我上次和云端**对齐**的时间 | 同步模块，每次上传 / 下载成功后更新 |

自动上传前先 GET 一次云端，比较云端 `exportedAt`：

- **云端 ≤ 我的 `csSyncedAt`** → 云端内容我都见过，是我的数据，放心推；
- **云端 > 我的 `csSyncedAt`** → 有别的设备推过我没见过的版本。盲目上传会用旧数据把云端整个盖掉，另一台设备的打卡直接丢失。所以选择**暂停自动上传 + 提示手动下载，宁停不盲写**。手动上传遇到同样情况也会弹 confirm 双重确认。

> **为什么「云端更新」反而要停下来？** 你在手机打卡（已推送），然后打开电脑（还停留在上次同步状态）随手改了一笔。若 B 直接自动上传，会用没见过 A 数据的旧仓库把云端盖掉——手机打卡就丢了。停下来不是防冲突本身，是防丢数据。

**`silent` 写通道：防自激循环。** 同步动作本身也要写库（更新 `csSyncedAt`）。若这些写入也触发「8 秒后自动上传」，会形成 `push → 写时间戳 → 触发 push → 再写 → 无限循环`，烧光 GitHub 限流配额。因此写队列的 `silent` 参数让同步内部写入走静默通道——**不更新 `lastLocalWrite`、不触发定时器**。只有真正打卡、改计划这类用户操作，才会驱动上传。

### 设计取舍与局限

方案是**整包 last-write-wins（最后写入者胜）+ 人工确认兜底**，不是 Git 式增量合并。极端场景：两台设备**同时**离线打卡再同时上线——后推的那台会弹确认框，要在「上传覆盖」和「下载覆盖」之间二选一，**不会自动合并**两边的打卡项。

| 方案 | 冲突处理 | 复杂度 | 额外依赖 |
| --- | --- | --- | --- |
| 本方案（整包 + 时间戳） | 人工二选一，绝不丢已见数据 | 约 180 行，零依赖 | 无 |
| 字段级合并 | 自动合并 | 需 CRDT / 向量时钟 | 通常需后端 |

真要双向合并需引入 CRDT 或向量时钟——在一个零依赖单文件应用里不值得。这个复杂度预算，留给了真正每天在用的功能：打卡的速度、拖拽的手感、统计的直观。

> **一句话总结**：部署解决「文件跟着网址走」，Gist 解决「数据跟着 token 走」，时间戳对账保证「谁都没见过的改动不会被覆盖」。

---

## 开发与维护

- **改代码**：直接编辑 `dist/index.html`（唯一真源），保存即生效，无构建；
- **重新部署**：编辑后把 `dist/index.html` 重新部署到沙箱 / 静态服务器即可，用户数据不受影响；
- **新增功能**：遵循既有分层——数据操作走 `DataLayer` + `Store.enqueue`（不要绕过写队列），UI 用 `renderXxx` 纯渲染函数，涉及同步逻辑的写操作保留 `silent` 语义；
- **自查**：改完用浏览器实测四页签 + 拖拽 + 导入导出 + 云同步；也可用 `node -e "new Function(js)"` 做一次 JS 语法解析校验（代码在 `<script>` 内，需先抽取）。
