# 学习工作台 · 秋招备考

一个**零依赖的单文件打卡应用**：今日打卡 / 计划管理 / 统计热力图 / 学习日志四个页签，IndexedDB 本地存储 + GitHub Gist 多设备云同步。全部代码打包在一个 HTML 文件里，双击即用。

## 📖 使用说明与技术实现

👉 **[docs/workbench-guide.html](docs/workbench-guide.html)** —— 完整介绍页：功能一览、架构与数据流、IndexedDB 与串行写队列、Gist 多设备同步、冲突保护、上手指南、设计取舍。

（技术实现细节不再在此重复，见上方介绍页。）

## 快速开始

| 场景 | 操作 |
| --- | --- |
| **本地使用** | 双击 `dist/index.html`，数据存本机浏览器 IndexedDB |
| **部署上线** | 把 `dist/index.html` 部署到任意静态服务器 / CloudStudio 沙箱，获得固定网址 |
| **多设备同步** | 计划页「云同步」卡片 → 粘贴 GitHub token（`gist` 权限）→ 启用；其他设备同 token 点「从云端下载」即可，之后每次改动 8 秒后自动上传 |
| **备份迁移** | 右上角导出 JSON/CSV；换源（如 `file://` → 线上）时用「导出 JSON → 导入恢复」迁移一次 |

## 项目结构

```
learningplatform/
├── README.md                # 本文件
├── dist/index.html          # 工作台本体（单文件，源 = 部署产物，改这里）
├── docs/
│   ├── workbench-guide.html # ★ 使用说明 + 技术实现介绍页
│   └── design-notes.md      # 设计笔记（数据模型 / 交互要点 / 验收清单）
└── backup/index.html        # v1 历史版本存档
```

## 技术实现一句话

三层架构：**分发层**（静态服务器只递文件）→ **运行层**（浏览器内渲染 / 打卡 / 拖拽）→ **存储层**（IndexedDB 本地 + Gist 云端）；所有写操作走 `Store.enqueue` 串行队列，双时间戳（`lastLocalWrite` vs `csSyncedAt`）对账防覆盖。详见 [workbench-guide.html](docs/workbench-guide.html)。
