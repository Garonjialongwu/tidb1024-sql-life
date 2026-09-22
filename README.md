# 一条 SQL 的一生 · 1024 TiDB AIGC 黑客松

交互式单文件 Web 应用：把 `SELECT name FROM users WHERE id = 42;` 在 TiDB 里的一次完整旅程
做成可点击、可播放的动画。

- 形式：**可运行站点**（程序类门槛之一，因此**无需运行视频**）
- 方向：① 产品特性可视化 / ③ 平凯 Loop 实践
- 零依赖：单个 `index.html`，无外部资源、无需构建、**不依赖真集群**

## 本地运行
```bash
open index.html        # macOS
# 或
python3 -m http.server # 然后访问 http://localhost:8000
```

## 部署（GitHub Pages）
仓库根目录即站点根，`index.html` 为入口。`.github/workflows/pages.yml` 已配好：
push 到 `main` 后自动发布到 `https://Garonjialongwu.github.io/tidb1024-sql-life/`。
首次需在仓库 Settings → Pages → Source 选 **GitHub Actions**。

## 内容准确性
技术解读稿见 [`sql-life-technical-notes.md`](./sql-life-technical-notes.md)，
关键论断逐条附 **PingCAP 官方文档**来源，并已抓原文核对（含一处易错点：Region 默认 256 MiB）。

## AIGC 说明
文案/技术解读由 **PingCAP Loop** 的 Agent 协作生成并抓官方原文核对；
拓扑与动画为**代码生成的 SVG/Canvas**（未使用图像生成模型）；代码由 Agent 编写、人工复核。
