# 川渝大数据岗位招聘信息分析

> 用官方公共就业平台的真实岗位数据，回答一个具体问题：以「专科 + Python/pandas 入门」为条件，川渝的数据类岗位要什么、给多少，我差在哪。

## 一句话简介

采集四川人才网官方接口的岗位数据，用 pandas 清洗成 10 个标准字段的数据集，按技能、城市薪资、学历溢价、经验门槛四个维度出图分析，产出一份能支撑「投哪个方向、补哪项技能、薪资怎么报」三个决策的报告。

## 技术栈

| 类别 | 选型 | 说明 |
|---|---|---|
| 语言 | Python 3.10+ | 全程单文件脚本，可逐格在 Jupyter 运行 |
| 采集 | requests | 直连平台 JSON 接口，不用 bs4 解析 HTML |
| 清洗 | pandas | 去重、缺失、异常、类型转换、双表输出 |
| 可视化 | matplotlib | 4 张图，中文字体统一设置 |
| 文档 | Markdown | 报告与 README，GitHub Pages 可直接渲染 |

## 项目结构树

```text
job-data-cy/
├── README.md                  # 本文件
├── LICENSE
├── 01_项目立项_双版本简介.md      # 业务问题、子问题拆解、简历版/作品集版
├── 02_数据采集方案.md            # 源评估、字段映射、反爬应对
├── 03_数据清洗与EDA流程.md       # 清洗流程、代码、分析维度
├── 04_可视化与业务结论.md        # 4 张图方案与业务结论
├── 05_项目报告.md               # 报告底稿
├── 06_发布材料与面试准备.md      # 口述稿、16 题问答
├── 07_图表插入与部署.md          # 配图插入与 Pages 部署
├── assets/img/                # 结构类配图（流程图、ER 图），由 mermaid 导出
│   ├── dataflow.png
│   └── er-diagram.png
├── docs/                      # GitHub Pages 发布根目录（只发布这一层）
│   ├── index.md               # Pages 首页
│   ├── report.md              # 项目报告正文（图 1–图 6）
│   ├── diagrams/              # 配图的 mermaid 源文件，改图从这里改
│   │   ├── dataflow.mmd
│   │   └── er-diagram.mmd
│   ├── assets/img/            # 结构图（发布用，与根目录的 assets 同一批图）
│   │   ├── dataflow.png
│   │   └── er-diagram.png
│   └── figs/                  # 分析图（make_figs.py 直接输出到这里）
│       ├── fig1_skills_top20.png
│       ├── fig2_city_salary_box.png
│       ├── fig3_edu_salary.png
│       └── fig4_exp_donut.png
├── data/
│   ├── raw/                   # 原始快照，按日期命名，只增不改
│   │   └── job_snapshot_YYYYMMDD.csv
│   ├── job_clean.csv          # 清洗后主表，严格 10 列
│   └── job_aux.csv            # 辅助列（job_id / direction / match_level …）
└── src/
    ├── fetch_jobs.py          # 请求 + 主循环
    ├── clean_jobs.py          # 解析 + 存储 + 清洗
    └── make_figs.py           # 4 张分析图
```

> 数据处理流程图和数据表结构图在报告里，见 [报告第一章](docs/report.md#一项目概述)（图 1）和 [第二章](docs/report.md#二数据源与采集方案)（图 2）。两张图只放一处，README 不重复贴。

## 复现步骤

### 1. 数据

```bash
# 每个自然日跑一次，单次最多 6 页 × 50 条 = 300 条（平台硬上限）
python src/fetch_jobs.py
# 输出：data/raw/job_snapshot_YYYYMMDD.csv
```

> 数据类岗位在平台总量里占比很低，单次快照不足以支撑统计，需要连续累积 10 天以上。

### 2. 代码

```bash
# 清洗：去重 → 缺失 → 异常 → 类型转换 → 输出双表
python src/clean_jobs.py
# 输出：data/job_clean.csv（10 列）、data/job_aux.csv、质检表

# 出图：4 张 PNG
python src/make_figs.py
# 输出：docs/figs/*.png
```

### 3. 结果

- 数据集：`data/job_clean.csv`
- 图表：`docs/figs/` 下 4 张 PNG + `docs/assets/img/` 下 2 张结构图
- 报告：`docs/report.md`

## 图表预览说明

| 图号 | 文件 | 说明 | 回答的问题 |
|---|---|---|---|
| 图 1 | `docs/assets/img/dataflow.png` | 数据处理全流程，清洗环节展开四个子步骤 | 整条链路怎么走 |
| 图 2 | `docs/assets/img/er-diagram.png` | 主表 10 列与辅助表的结构关系 | 字段怎么存、怎么关联 |
| 图 3 | `docs/figs/fig1_skills_top20.png` | 技能提及率 TOP20 横向条形图，已掌握/待补双色 | 市场最认哪几个技能，我还差哪些 |
| 图 4 | `docs/figs/fig2_city_salary_box.png` | 分城市薪资箱线图，按有效样本量排序，n<5 不画 | 城市之间薪资差多少 |
| 图 5 | `docs/figs/fig3_edu_salary.png` | 学历 × 薪资中位数柱状图，柱顶标 n | 学历溢价有多大 |
| 图 6 | `docs/figs/fig4_exp_donut.png` | 经验要求占比环形图，「未知」不入环 | 应届生能投的池子多大 |

> 所有薪资相关图的样本量都标在轴标题或柱顶，因为中位数只在「公开薪资」的子集上计算，样本少时中位数不可信。

## 合规声明

- 只采集官方与公共就业服务平台的公开岗位信息，不采商业招聘平台
- 不识别验证码、不伪造登录态、不使用代理池绕过限制
- 请求间隔 1.2–2 秒随机，单线程，每日不超过 1 次
- 数据仅用于个人求职研究与学习，不转售、不批量公开原始数据
- 不采集任何求职者个人信息

## License

MIT License，详见 [LICENSE](LICENSE)。数据本身不随仓库分发，需自行按上述步骤采集。
