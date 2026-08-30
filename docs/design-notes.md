# 秋招备考打卡系统 — 实施计划

## 目标与交付物
一个**自包含单文件**交互应用 `index.html`：内置商定的秋招备考计划，支持每日打卡、反思/心得/调整建议留档、本地数据库持久化、统计面板。零外部依赖、离线可用、双击即开。

- 文件：`C:\Users\wujue\WorkBuddy\2026-08-11-01-25-41\秋招打卡系统\index.html`（CSS/JS 全内联，不引 CDN，不依赖构建）
- 浅色主题（IDE 为 light），响应式（375px 手机可用），单页 SPA + 底部 4 Tab

## 已确认决策（与用户对齐）
1. 存储：**IndexedDB**（本地数据库，事务/持久化）；`window.indexedDB` 不可用时静默降级 localStorage（DataLayer 封装，上层无感，提示"只读降级"）
2. **打卡项可增删改**：存在库里，可增改任务、调顺序；删除用软删除 `archived:true`（保历史可回溯）
3. **带统计面板**：连续天数、GitHub 风格日历热力图、周/月完成率

## 数据模型（IndexedDB，库 `autumn-recruit-checkin` v1，`onupgradeneeded` 建表）
- `planItems`（keyPath `id`，自增；索引 `groupId`）：`{ id, groupId, title, sort, archived }`
  - groupId ∈ morning / afternoon / evening / bedtime
- `days`（keyPath `date` = `YYYY-MM-DD`）：`{ date, doneIds: [itemId], reflect, insight, adjust, updatedAt }`
- `meta`（keyPath `key`）：存 `seedVersion`，首次启动写入种子计划，避免重复播种
- 历史打卡记录存 **item_id**（不存快照）：轻量、改标题/排序零影响；统计时排除 archived 项

## 页面结构（底部 Tab：今日 / 计划 / 统计 / 日志）
1. **今日打卡**：日期切换器（可补卡）→ 4 组打卡列表（早上·算法 60min / 午后·主攻 90min / 晚上·面经投递 60min / 睡前·复盘 10min）→ 当日完成率进度条 → 三个文本框：反思 / 心得 / 计划调整建议
2. **计划管理**：按组分组的打卡项增删改、拖拽排序、一键重置种子数据；附"计划总览"静态展示（四阶段时间线、50/30/20 精力分配、周节奏、习惯四原则）
3. **统计**：连续打卡天数（口径：**当天至少完成 1 项**，从今天反向遍历，遇空当天即止）+ 热力图（以今天所在周为右端、按周回排，完成率 4 档着色）+ 本周/本月完成率卡片
4. **反思日志**：按日期倒序时间线，类型筛选（全部/反思/心得/调整建议），可回看、可编辑历史

## 交互要点
- 勾选 checkbox **即时写库**（合并单事务 + 防抖）；文本 500ms 防抖自动保存；切换日期前强制 flush
- 日期一律用本地时区 `getFullYear/Month/Date` 拼 `YYYY-MM-DD`，**禁用 toISOString**（UTC 跨日错位）
- `days` 以 date 为主键 upsert，doneIds 用 Set 去重 → 重复勾选/补卡幂等
- 所有写操作走 Store 串行写队列，消除 IndexedDB 异步竞态
- 导出：全量 JSON 备份 + 每日记录 CSV；导入：清库后还原（含 meta，防种子重复播种）

## 种子数据（初始打卡项，按 4 组）
- **早上·算法**：刷题 2~3 道（Hot100/剑指Offer）、错题入库整理
- **午后·主攻**：Agent/LLM 学习（第一周 API+提示词 → 第二周 Agent 框架 → 第三周起做"秋招助手"作品）、岗位知识 1 主题并输出笔记
- **晚上·投递面经**：投递 3 家、复盘 1 场面经/笔试
- **睡前·复盘**：列明日 3 件事、今日打卡与弹性检视

## 代码分区（单文件内，总计约 1300 行）
- `DataLayer`（~150 行）：init / upsert / query / export / import，含 localStorage 降级
- `Store`（~80 行）：当前日期、当日缓存、串行写队列
- `render`（~450 行）：纯渲染函数，按 Tab 划分
- `events`（~200 行）：事件绑定 + Tab 路由
- CSS（~300 行）+ HTML 骨架（~150 行）

## 高风险点（实现时重点处理）
① IndexedDB 写后读竞态 → 统一 Store 队列；② 日期 UTC 错位 → 本地格式化；③ file:// 隐私模式禁用 IndexedDB → 降级检测；④ 快速连续勾选丢写 → 合并事务；⑤ 文本防抖与切日期冲突 → 保存后再切页

## 实施步骤
1. 建目录 `秋招打卡系统/`，写 index.html 骨架（HTML + CSS，4 Tab 布局）
2. DataLayer：IndexedDB 初始化 + 降级 + 三 store
3. 种子数据播种（meta.seedVersion 控制）
4. 今日打卡页：勾选即时保存、三文本框防抖、补卡、进度条
5. 计划管理页：CRUD + 排序 + 重置种子 + 计划总览静态内容
6. 统计页：连续天数 + 热力图 + 周/月完成率
7. 反思日志页：时间线 + 筛选 + 编辑
8. 导出 JSON / CSV + 导入还原
9. 375px 响应式打磨 + 按验收清单自测

## 验收清单
1. 双击打开即用，控制台零报错
2. 勾选/写日志后刷新，数据不丢
3. 导出 JSON → 清库 → 导入可完整还原
4. 375px 模拟下 4 个 Tab 均可打卡、补卡
5. 断网离线可正常使用
