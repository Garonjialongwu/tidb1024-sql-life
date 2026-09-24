# 数据库的前世今生：从一张表格到 TiDB

滚动叙事长页（scrollytelling）单文件 Web 应用：同一批等距数据立方体，随滚动在十幕间真实重排，
讲清数据如何从文件、表、关系模型、NoSQL 一路走到 NewSQL 与 TiDB。

- 形式：**可运行站点**（单个 `index.html`，零依赖、无外部资源、无需构建）
- 方向：① 产品特性解读 / 技术概念可视化
- 长图：在地址后加 `?long=1` 可得到同源竖版信息长图（同一套代码生成 SVG）

## 本地运行
```bash
open index.html        # macOS
# 或
python3 -m http.server # 然后访问 http://localhost:8000
```

## 部署（GitHub Pages）
仓库根目录即站点根，`index.html` 为入口，`.nojekyll` 已置。
Settings → Pages → Source = **Deploy from a branch** → Branch `main` → `/ (root)` → Save。

## 内容准确性
史实（Scene 2–6）与产品能力（Scene 8）分栏，能力逐条对照 **PingCAP 官方文档**（`docs.pingcap.com`）；
产品主张不含绝对化用语。

## AIGC 说明
文案/技术解读由 **PingCAP Loop** 的多 Agent 协作生成并对照官方原文；
可视化全部为**代码生成的 SVG**（未使用图像生成模型），代码由 Agent 编写、人工复核。

> 图注：代码生成、未使用图像生成模型。
