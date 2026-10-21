# 数据清洗与 EDA 流程（第 3 步）

> 说明：本文所有数字都只给公式和代码，不给结果值。真实数字你在 Jupyter 里跑出来自己填进对照表。
> 运行顺序就是本文顺序，每个 `# ===== 步骤N =====` 是一格，按 Ctrl+Enter 逐格跑。

---

## 0. 回填第 2 步的字段清单

清洗后统一为这 10 列，顺序固定：

| 列名 | 含义 | 清洗后类型 |
|---|---|---|
| `job_title` | 岗位名称 | str |
| `company` | 公司名称 | str |
| `city` | 城市 | str |
| `salary_min` | 薪资下限（元/月） | float |
| `salary_max` | 薪资上限（元/月） | float |
| `salary_median` | 薪资中位数（计算列） | float |
| `education` | 学历要求 | 有序分类 |
| `experience` | 经验要求 | 有序分类 |
| `skills` | 技能要求（词表匹配，`;` 分隔） | str |
| `post_date` | 发布时间 | datetime64 |

辅助列（`job_id / direction / match_level / salary_text / industry / source_url / crawl_time`）不进主表，另存 `job_aux.csv`，两份文件行序一一对应，用 `pd.concat(..., axis=1)` 合并。

---

## 1. 清洗流程表

| 步骤号 | 目标 | 判定规则（什么情况算脏） | pandas 代码要点 | 业务含义 |
|---|---|---|---|---|
| S0 | 读入并备份 | 读进来就算脏的起点，不改原文件 | `pd.read_csv(path, encoding="utf-8-sig", dtype=str)` 全列先读成字符串 | 让 pandas 别自己猜类型——它会把 `"4000.0"` 猜成 float，把 `"001"` 的前导零吃掉。原始文件绝不覆盖 |
| S1 | 去重 | ① `job_id` 相同（跨天快照重复）② `job_title`+`company`+`post_date` 三项全同（同一天重复投放） | `df.duplicated(subset=[...], keep="last")` + `drop_duplicates` | 一条岗位重复计数，Q1 的岗位供给量直接虚高。保留 `last`（最新抓取）是因为后抓的那条字段通常更全 |
| S2 | 缺失率体检 | 先算不清洗：每个字段缺多少，缺的形态是什么 | `df.isna().mean().sort_values(ascending=False)` | 缺失率决定策略，不看就先填，一定填错。薪资缺失率高不是数据坏，是「面议」这个业务事实 |
| S3 | 缺失值处理（分字段定策略） | 见下方 3.1 的策略表：有的删行、有的补、有的留空但打标 | `dropna` / `fillna` / 新增标记列 | 同一张表里不同字段的缺失含义完全不同：`job_title` 缺=这条记录废了；`salary` 缺=面议，是有效信息 |
| S4 | 异常值：薪资过低 | `salary_min` < 【当地最低工资标准，如成都主城区 2100 元/月，以四川省人社厅最新文件为准】 | `df.loc[条件, "salary_flag"] = "below_min_wage"` | 低于最低工资的岗位多半是兼职、日结、提成制或录入错误，**打标不删**，留证据给人工复核 |
| S5 | 异常值：薪资过高 | `salary_max` > `salary_max.quantile(0.99)`，或超过【行业上限，如 80000 元/月】 | 分位数法兜底，避免被个别极端值拉偏 | 上限用分位数比自己拍一个数更稳，且换城市换行业都不用改代码 |
| S6 | 异常值：薪资区间反了 | `salary_max` < `salary_min` | 交换或两列同时置空，计数留痕 | 录入错误。置空比交换安全，因为不知道哪个是对的 |
| S7 | 异常值：city 带括号备注 | `成都市（高新区）`、`成都市(温江)` 这类带括注的写法 | `str.replace(r"[（(].*?[)）]", "", regex=True)` | 不清理的话 `groupby("city")` 会把同一个成都拆成七八个城市，Q1 的柱状图直接散架 |
| S8 | 异常值：skills 格式不统一 | 分隔符混用 `、` `,` `/` `；`，或首尾带空格、重复项 | 统一替换成 `;` 后 `split(";")` 再 `set` 去重 | 不统一就统计不出词频——`Excel` 和 `excel ` 会被算成两个技能，Q2 的排名全乱 |
| S9 | 类型转换：薪资三列 | 文本 `4000-6000元/月`、`面议`、`5000元/月以上` → 数值 | 正则 `re.findall(r"\d+(?:\.\d+)?", t)` 取数，单位换算，再算中位数 | 字符串没法算中位数和箱线图。这一步决定 Q3 能不能做 |
| S10 | 类型转换：post_date | 字符串 → `datetime64`；含时分秒的截断到日 | `pd.to_datetime(errors="coerce")` | 有 datetime 才能做月度趋势；`errors="coerce"` 让解析失败变 NaT 而不是整行报错 |
| S11 | 类型转换：有序分类 | `education` / `experience` 是文本，无法按学历高低排序 | `pd.Categorical(categories=[...], ordered=True)` | 转成有序分类后，箱线图的横轴才会按「不限→大专→本科」排，而不是按拼音排 |
| S12 | 输出与质检 | 主表 10 列 + 辅助表 + 质检表 | `to_csv(index=False, encoding="utf-8-sig")` | 交付口径和简历上写的「10 字段」必须一致 |

### 3.1 缺失值策略表（S3 展开）

| 字段 | 缺失含义 | 策略 | 理由 |
|---|---|---|---|
| `job_title` | 岗位名都没有，这条记录无法归类 | **删行** | 没有岗位名的记录答不了任何子问题 |
| `company` | 企业未公开名称 | 填 `未公开`，保留行 | 公司名只影响 Q5 的行业对齐，不该因为缺公司名丢掉一条有效岗位 |
| `city` | 地址没写或格式异常 | 先从 `address` 补，补不到填 `未知`，Q1 统计时排除 | 未知城市不能硬塞进成都 |
| `salary_min` / `salary_max` | 「面议」或未公布 | **留空，不填 0，新增 `salary_flag="negotiable"`** | 填 0 会把中位数拉到地板；填均值等于凭空造数 |
| `salary_median` | 由上下限计算，任一为空则为空 | 计算得到，不插补 | 见上一条 |
| `education` | 平台没填 | 填 `未知`，作为独立一档 | **不能猜成「不限」**——没写≠不限，这是两回事 |
| `experience` | 平台没填 | 填 `未知`，作为独立一档 | 同上 |
| `skills` | 职位描述为空，或一个词表词都没命中 | 留空字符串 | Q2 的分母只能用「有描述的岗位数」，分母要在图注里写清 |
| `post_date` | 发布时间解析失败 | 保留行，趋势分析时排除 | 单条缺日期不影响 Q1–Q4 |

---

## 2. 清洗代码（按步骤逐格运行）

### 步骤 0：读入并备份

```python
# ===== 步骤0 读入 =====
import re
import pandas as pd
import numpy as np

# 原始文件路径（换成你自己的日期）
raw_path = "data/job_snapshot_20261021.csv"

# 关键：dtype=str 让所有列先按字符串读进来
# 为什么这样做：pandas 默认会猜类型，"4000.0" 会被猜成 float，
# 但我们在清洗阶段需要看到原始写法（比如 "4000-6000元/月" 这种混在一起的）
# 所以统一先当文本，后面再逐列转成需要的类型
df_raw = pd.read_csv(raw_path, encoding="utf-8-sig", dtype=str)

print("原始行数：", len(df_raw))          # 用途：记下清洗起点，后面每步对比
print("原始列名：", list(df_raw.columns))
print(df_raw.head(3))                      # 用途：肉眼看一遍，确认列读对了没有
```

```python
# ===== 步骤0.1 建立工作副本 =====
# 为什么不在 df_raw 上直接改：清洗过程中要反复对比"清洗前 vs 清洗后"，
# 原始数据必须留着做参照，否则对照表的数字填不出来
df = df_raw.copy()
```

### 步骤 1：去重

```python
# ===== 步骤1 去重 =====
rows_before = len(df)

# 第一层：按 job_id 去重（跨天快照最容易重复）
if "job_id" in df.columns:
    dup_by_id = df.duplicated(subset=["job_id"], keep="last").sum()
    print("按 job_id 判定重复：", dup_by_id, "行")
    df = df.drop_duplicates(subset=["job_id"], keep="last")

# 第二层：按"岗位名+公司+发布日期"去重（同一天重复投放，job_id 不同但其实是同一条）
key_cols = ["job_title", "company", "post_date"]
dup_by_content = df.duplicated(subset=key_cols, keep="last").sum()
print("按 岗位名+公司+日期 判定重复：", dup_by_content, "行")

df = df.drop_duplicates(subset=key_cols, keep="last")

rows_after = len(df)
print(f"去重：{rows_before} → {rows_after} 行，去掉 {rows_before - rows_after} 行")
print("重复率：", round((rows_before - rows_after) / rows_before * 100, 2), "%")
```

**为什么保留 `keep="last"`**：多次采集时后抓到的记录通常字段更全（岗位被编辑过），而且发布时间更新。

### 步骤 2：缺失率体检（先看再动手）

```python
# ===== 步骤2 缺失率 =====
# 为什么先算缺失率：不同字段的缺失处理方式完全相反，
# 不看清楚就 fillna，很可能把"面议薪资"填成 0，直接毁掉整个薪资分析
missing = df.isna().mean().mul(100).round(2).sort_values(ascending=False)
print("各字段缺失率（%）：")
print(missing)
```

```python
# ===== 步骤2.1 看缺失的形态（不只看比例，还要看长什么样）=====
# 薪资字段缺失时，原始文本写的是"面议"还是真的空白？这决定后面怎么打标
print("薪资原文取值分布（前10）：")
print(df["salary_text"].value_counts(dropna=False).head(10))
```

### 步骤 3：缺失值处理

```python
# ===== 步骤3 缺失值 =====
# 3.1 job_title 缺失 → 删行（没有岗位名的记录答不了任何问题）
before = len(df)
df = df[df["job_title"].notna() & (df["job_title"].str.strip() != "")]
print(f"job_title 空值删除：{before} → {len(df)} 行")

# 3.2 company 缺失 → 填"未公开"，保留行
# 为什么不删：公司名只影响 Q5 的行业对齐，不该因为缺公司名丢掉一条有效岗位
df["company"] = df["company"].fillna("未公开").replace("", "未公开")

# 3.3 city 缺失 → 先尝试从 address 里补（address 形如 "四川省-成都市-武侯区"）
if "address" in df.columns:
    city_from_addr = df["address"].fillna("").str.split("-").str[1]
    df["city"] = df["city"].fillna(city_from_addr)
df["city"] = df["city"].fillna("未知").replace("", "未知")

# 3.4 薪资缺失 → 留空 + 打标，绝不填 0
# 这是整份清洗里最重要的一条规则：
# 填 0 会让中位数掉到地板，填均值等于凭空造数，两种都是错的
df["salary_flag"] = np.where(
    df["salary_min"].isna() | df["salary_max"].isna(),
    "negotiable",     # 面议 / 未公布
    "ok"
)
print("薪资状态分布：")
print(df["salary_flag"].value_counts())

# 3.5 education / experience 缺失 → 填"未知"，单独一档
# 为什么不填"不限"：平台没写 不等于 学历不限，这是两回事，混在一起会低估门槛
df["education"] = df["education"].fillna("未知").replace("", "未知")
df["experience"] = df["experience"].fillna("未知").replace("", "未知")

# 3.6 skills 缺失 → 留空字符串（Q2 的分母只算有描述的岗位）
df["skills"] = df["skills"].fillna("").astype(str)

print("处理后剩余缺失：")
print(df.isna().mean().mul(100).round(2).sort_values(ascending=False).head(6))
```

### 步骤 4：异常值——薪资

```python
# ===== 步骤4 薪资异常值 =====
# 先把薪资转成数字（如果读进来是字符串）
df["salary_min"] = pd.to_numeric(df["salary_min"], errors="coerce")
df["salary_max"] = pd.to_numeric(df["salary_max"], errors="coerce")

# 4.1 上下限反了 → 两列同时置空并计数
# 为什么置空而不是交换：我们不知道哪个数才是对的，交换等于替数据做主
wrong_order = df["salary_max"] < df["salary_min"]
print("上下限反转的条数：", wrong_order.sum())
df.loc[wrong_order, ["salary_min", "salary_max"]] = np.nan

# 4.2 低于当地最低工资
# 【待填】min_wage 请填四川省人社厅最新公布的最低工资标准（成都主城区档）
min_wage = 2100          # 单位：元/月，【以最新官方文件为准】
df.loc[df["salary_min"] < min_wage, "salary_flag"] = "below_min_wage"

# 4.3 高于 P99（用分位数兜底，比手拍一个上限更稳）
high_cap = df["salary_max"].quantile(0.99)
print("P99 上限：", round(high_cap, 0), "元/月")
df.loc[df["salary_max"] > high_cap, "salary_flag"] = "above_p99"

# 4.4 打标后看分布，不删除
# 为什么不删：低于最低工资的岗位可能真是兼职/日结，是有信息量的，删了就等于假装它不存在
print("异常标记分布：")
print(df["salary_flag"].value_counts())
```

### 步骤 5：异常值——city 与 skills

```python
# ===== 步骤5 文本类异常 =====
# 5.1 city 去掉括号备注，如 "成都市（高新区）" → "成都市"
# 不清理的后果：groupby("city") 会把一个成都拆成七八个城市，Q1 柱状图直接散架
df["city"] = (
    df["city"]
    .astype(str)
    .str.replace(r"[（(].*?[)）]", "", regex=True)   # 去掉中英文括号及内容
    .str.replace(" ", "", regex=False)             # 去空格
    .str.strip()
)

# 5.2 city 统一写法：用白名单映射，不要用"结尾加市"这种规则
# 为什么不用自动加"市"：像"万州""涪陵"这类区名会被加成"万州市"，反而错得更离谱。
# 只处理自己数据里真实出现过的写法，跑完 5.1 先 print 一遍取值再补这张表
city_fix = {
    "成都": "成都市", "绵阳": "绵阳市", "德阳": "德阳市",
    "宜宾": "宜宾市", "泸州": "泸州市", "南充": "南充市",
    "重庆": "重庆市", "渝北": "重庆市", "渝中": "重庆市",
}
df["city"] = df["city"].replace(city_fix)

print("city 取值分布（前10）：")
print(df["city"].value_counts().head(10))
# 用途：看有没有"成都市""成都 "这种同城异写，有就继续往 city_fix 里加
```

```python
# ===== 步骤5.3 skills 格式统一 =====
# 分隔符混用（、 , / ；）统一成 ";"，再去掉首尾空格和重复项
def clean_skills(text):
    """把一条 skills 字符串整理成 'A;B;C' 的形式"""
    if not isinstance(text, str) or text.strip() == "":
        return ""
    # 把所有可能的分隔符换成统一的 ;
    t = re.sub(r"[、,，/／;；|]", ";", text)
    # 拆开、去空格、去空值
    items = [x.strip() for x in t.split(";") if x.strip() != ""]
    # 去重但保留原顺序（dict.fromkeys 是去重保序的常用写法）
    items = list(dict.fromkeys(items))
    return ";".join(items)

df["skills"] = df["skills"].apply(clean_skills)

print("skills 整理后示例（前5条非空）：")
print(df.loc[df["skills"] != "", "skills"].head(5).tolist())
```

### 步骤 6：薪资文本解析（备用源或文本格式才用）

```python
# ===== 步骤6 从文本拆薪资（只有当薪资是 "4000-6000元/月" 这种文本时才需要）=====
# 首选源给的是数值字段，这一步可以跳过；换源或字段变了才用
def parse_salary_text(text):
    """'4000-6000元/月' → (4000, 6000)；'面议' → (None, None)"""
    if not isinstance(text, str):
        return None, None
    t = text.strip()
    # 统一各种连接符和标点
    t = t.replace("，", ",").replace("～", "-").replace("—", "-").replace("－", "-")
    # 面议 / 空 → 都返回空，不返回 0
    if t == "" or "面议" in t:
        return None, None

    nums = re.findall(r"\d+(?:\.\d+)?", t)      # 找出所有数字
    if len(nums) == 0:
        return None, None

    nums = [float(n) for n in nums]
    # 单位换算：带"万"的乘 10000；按"年"算的除 12 换成月薪
    if "万" in t:
        nums = [n * 10000 for n in nums]
    if "年" in t and "月" not in t:
        nums = [n / 12 for n in nums]

    if len(nums) == 1:
        return nums[0], nums[0]                 # 只写一个数，上下限相同
    return nums[0], nums[1]

# 用法示例（自己按实际列名调整）
# df[["salary_min", "salary_max"]] = df["salary_text"].apply(
#     lambda x: pd.Series(parse_salary_text(x))
# )
```

### 步骤 7：类型转换与派生列

```python
# ===== 步骤7 类型转换 =====
# 7.1 薪资三列
df["salary_min"] = pd.to_numeric(df["salary_min"], errors="coerce").round(0)
df["salary_max"] = pd.to_numeric(df["salary_max"], errors="coerce").round(0)

# 中位数是计算列，页面上没有；任一为空则结果为空（不插补）
df["salary_median"] = ((df["salary_min"] + df["salary_max"]) / 2).round(0)

print("薪资三列抽样：")
print(df[["salary_min", "salary_max", "salary_median"]].dropna().head(5))

# 7.2 post_date 转 datetime
# errors="coerce" 让解析失败的变成 NaT，而不是让整行报错中断
df["post_date"] = pd.to_datetime(df["post_date"], errors="coerce")
print("post_date 解析失败条数：", df["post_date"].isna().sum())
print("时间范围：", df["post_date"].min(), "→", df["post_date"].max())
```

```python
# ===== 步骤7.3 学历、经验转有序分类 =====
# 为什么要转：文本无法排序，箱线图的横轴会按拼音排成"大专、本科、不限"这样乱序
edu_map = {
    "学历不限": "不限", "无要求": "不限", "不限": "不限",
    "初中及以下": "初中", "初中": "初中",
    "高中": "高中", "中专": "中专", "中专/中技": "中专",
    "大专": "大专", "专科": "大专", "大学专科": "大专",
    "本科": "本科", "大学本科": "本科",
    "硕士": "硕士", "研究生": "硕士", "博士": "博士",
}
df["education"] = df["education"].map(edu_map).fillna("未知")
edu_order = ["不限", "初中", "高中", "中专", "大专", "本科", "硕士", "博士", "未知"]
df["education"] = pd.Categorical(df["education"], categories=edu_order, ordered=True)

exp_map = {
    "无要求": "不限", "不限": "不限", "经验不限": "不限",
    "1年以下": "1年以下", "一年以下": "1年以下",
    "1-3年": "1-3年", "1~3年": "1-3年", "1－3年": "1-3年",
    "3-5年": "3-5年", "3~5年": "3-5年",
    "5年以上": "5年以上", "5-10年": "5年以上", "10年以上": "5年以上",
}
df["experience"] = df["experience"].map(exp_map).fillna("未知")
exp_order = ["不限", "1年以下", "1-3年", "3-5年", "5年以上", "未知"]
df["experience"] = pd.Categorical(df["experience"], categories=exp_order, ordered=True)

print("学历分布：")
print(df["education"].value_counts().sort_index())
print("经验分布：")
print(df["experience"].value_counts().sort_index())
```

### 步骤 8：输出与质检

```python
# ===== 步骤8 输出 =====
# 交付主表：严格 10 列，顺序固定（简历上写"10 字段"就必须真的是 10 列）
clean_cols = ["job_title", "company", "city", "salary_min", "salary_max",
              "salary_median", "education", "experience", "skills", "post_date"]
df_clean = df[clean_cols].copy()

# 辅助表：分析要用但不算"标准字段"的列
aux_cols = ["job_id", "direction", "match_level", "salary_text",
            "industry", "source_url", "crawl_time", "salary_flag"]
aux_cols = [c for c in aux_cols if c in df.columns]     # 缺列不报错
df_aux = df[aux_cols].copy()

# 两张表行序完全一致（同一次切片出来的），合并时按位置拼
df_clean.to_csv("data/job_clean.csv", index=False, encoding="utf-8-sig")
df_aux.to_csv("data/job_aux.csv", index=False, encoding="utf-8-sig")

print("已输出：", len(df_clean), "行 ×", len(clean_cols), "列")
print("合并验证：")
print(pd.concat([df_clean.head(2), df_aux.head(2)], axis=1))
```

```python
# ===== 步骤8.1 自动生成质检表（对照表的数字直接从这里抄）=====
def missing_rate(data, cols=None):
    """返回各字段缺失率（%）"""
    s = data.isna().mean().mul(100).round(2)
    return s[cols] if cols else s

qc = pd.DataFrame({
    # 只看原始文件里真实存在的列——salary_median 是计算列，清洗前根本不存在，
    # 硬塞进来会显示 100% 缺失，看着像数据坏了，其实是我自己造的列
    "清洗前缺失率(%)": missing_rate(df_raw, [c for c in clean_cols if c in df_raw.columns]).reindex(clean_cols),
    "清洗后缺失率(%)": missing_rate(df_clean),
})
print("行数：清洗前", len(df_raw), "→ 清洗后", len(df_clean))
print(qc)
```

---

## 3. 清洗前后对照表模板

跑完上面代码后，把数字填进【】。

| 指标 | 清洗前 | 清洗后 | 变化说明 |
|---|---|---|---|
| 行数 | 【】 | 【】 | 去掉重复【】行、缺岗位名【】行 |
| 重复率 | 【】% | 0% | 按 `job_id` + 岗位名/公司/日期 双层去重 |
| `job_title` 缺失率 | 【】% | 【】% | 缺的整行删除 |
| `company` 缺失率 | 【】% | 【】% | 填「未公开」，比例为填充量 |
| `city` 缺失率 | 【】% | 【】% | 从 address 补 + 填「未知」 |
| `salary_min` 缺失率 | 【】% | 【】% | **不降低属正常**：面议岗位保留空值 |
| `salary_max` 缺失率 | 【】% | 【】% | 同上 |
| `salary_median` 缺失率 | 【】% | 【】% | 与上下限缺失一致 |
| `education` 缺失率 | 【】% | 0% | 填「未知」独立一档 |
| `experience` 缺失率 | 【】% | 0% | 填「未知」独立一档 |
| `skills` 缺失率 | 【】% | 【】% | 词表命中率决定，越低说明词表要补 |
| `post_date` 缺失率 | 【】% | 【】% | 解析失败保留行、趋势分析排除 |
| 薪资异常标记条数 | — | 【】 | below_min_wage 【】、above_p99 【】、negotiable 【】 |

**填法**：行数看 `len(df)`；缺失率看步骤 8.1 的 `qc` 表；异常条数看 `df["salary_flag"].value_counts()`。

> 薪资缺失率清洗后没下降是对的，不是代码写错了。这几个字段缺失代表「企业没公开」，处理方式是打标，不是消灭空值。面试被问到就这么答。

---

## 4. EDA 分析维度

### 4.1 描述性统计

```python
# ===== EDA-1 数值列描述性统计 =====
# 只看有效样本（salary_median 非空），样本量必须一起打印出来
salary_valid = df_clean["salary_median"].dropna()
print("有效样本量 N =", len(salary_valid))
print(salary_valid.describe())                    # 均值/标准差/四分位/极值
print("偏度 skew =", round(salary_valid.skew(), 2))
# 偏度 > 1 说明右偏严重（少数高薪岗位拉高均值），
# 这种情况报告里必须用中位数而不是均值，否则会被一个 5 万的岗位带跑
```

```python
# ===== EDA-2 分位数 =====
# 箱线图的箱体位置就是这几个数
for q in [0.1, 0.25, 0.5, 0.75, 0.9]:
    print(f"P{int(q*100)} = {salary_valid.quantile(q):.0f} 元/月")
```

```python
# ===== EDA-3 分类列频次 =====
print("学历频次：")
print(df_clean["education"].value_counts().sort_index())
print("经验频次：")
print(df_clean["experience"].value_counts().sort_index())
# 注意：频次的分母是全部行，还是"有明确要求"的行，图注里要写清
```

### 4.2 分组统计

```python
# ===== EDA-4 city × 薪资 =====
# count 是有效样本数（有薪资的），不是岗位总数，两个数要分开看
city_salary = (
    df_clean.groupby("city")["salary_median"]
    .agg(有效样本数="count", 中位数="median", 均值="mean")
    .sort_values("中位数", ascending=False)
)
print(city_salary)
```

```python
# ===== EDA-5 education × 薪资 =====
edu_salary = (
    df_clean.groupby("education", observed=False)["salary_median"]
    .agg(有效样本数="count", 中位数="median")
)
print(edu_salary)
# 样本数小于 5 的档位不要单独下结论，容易是噪声
```

```python
# ===== EDA-6 experience × 岗位量 =====
exp_count = df_clean["experience"].value_counts().sort_index()
print(exp_count)
print("占比：")
print((exp_count / len(df_clean) * 100).round(2))
```

```python
# ===== EDA-7 skills 词频 =====
# 一条岗位有多个技能，要先按 ; 拆开再统计
s = df_clean["skills"].fillna("").str.split(";").explode()
s = s[s.str.strip() != ""]                          # 去掉空技能
skill_count = s.value_counts()

total_with_skills = (df_clean["skills"].fillna("") != "").sum()
print("有技能描述的岗位数：", total_with_skills)
print("Top10 技能：")
print(skill_count.head(10))
# 技能提及率 = 该技能出现岗位数 / 有技能描述的岗位数
print("Top10 提及率（%）：")
print((skill_count.head(10) / total_with_skills * 100).round(1))
```

```python
# ===== EDA-8 三方向 × 城市（Q1 的数据源）=====
# direction 在辅助表里，先合并回来
if "direction" in df_aux.columns:
    df_all = pd.concat([df_clean, df_aux], axis=1)
    q1 = pd.crosstab(df_all["city"], df_all["direction"])
    print(q1)
    print("方向占比（%）：")
    print((df_all["direction"].value_counts(normalize=True) * 100).round(2))
```

### 4.3 维度 → 子问题 → 面试解读

| 分析维度 | 回答哪个子问题 | 输出什么 | 面试时的一句话解读 |
|---|---|---|---|
| `salary_median` 的均值/中位数/极值/偏度 | Q3 薪资分布 | 一段描述 + 直方图 | 「工资分布是右偏的，所以我在报告里用中位数而不是均值，均值会被少数高薪岗位拉高，不能代表大多数人」 |
| 分位数 P25/P50/P75 | Q3 | 箱线图的箱体 | 「P25 到 P75 这个区间，就是市面上大部分这类岗位的真实报价带」 |
| `education` 频次 | Q4 学历门槛 | 堆叠条形图 | 「我把『未知』单独算一档，因为平台没写不等于学历不限，混在一起会低估门槛」 |
| `experience` 频次 | Q4 经验门槛 | 堆叠条形图 | 「无经验要求的岗位占比，直接决定我这类应届生能投的池子有多大」 |
| `city` × 薪资 | Q1 + Q3 交叉 | 分组箱线图 | 「不同城市比的是中位数不是均值，而且要同时报有效样本数，样本太少的中位数不可信」 |
| `education` × 薪资 | Q4 补充 | 分组箱线图 | 「学历每上一档，中位数抬多少，这个差值就是学历的溢价」 |
| `skills` 词频 | Q2 技能要求 | 横向条形图 | 「分母我只用有技能描述的岗位，因为没写描述不等于没要求，用总岗位数当分母会系统性低估提及率」 |
| 三方向 × 城市 | Q1 供给量 | 分组柱状图 | 「供给量只统计高置信命中的岗位，描述里顺便提到『数据』两个字的那种我没算进去，宁可少不可虚」 |
| 行业/产业 × 岗位数（配公报增速） | Q5 政策一致性 | 双组柱状图 | 「我比的是排名是否同向，不是算相关系数——产业增速和岗位数量级不同，直接算相关没有意义」 |

---

## 附：面试官最可能追问

**Q：你这批数据里「面议」的岗位你是怎么处理的？**

> 话术：不填 0 也不填均值，留空并单独打标，中位数只在有薪资的子集上算，图注里写清有效样本量 N；因为面议不是随机缺失，写面议的岗位往往集中在薪资的两头，插补会把结论带偏。

**Q：你怎么判断一个薪资是异常值？**

> 话术：下限用官方最低工资标准卡，上限用 P99 分位而不是自己拍一个数，因为分位数会跟着数据走，换城市换行业都不用改代码；异常值我只打标不删除，先人工看几条确认是兼职还是录入错误，再决定要不要排除。

---

## 自检清单

- [ ] 每一步都有「前 vs 后」的行数或缺失率对比 print，不是闷头改完直接输出
- [ ] 薪资缺失一律留空打标，全流程没有出现 `fillna(0)` 或填充均值
- [ ] `job_clean.csv` 确实是 10 列且列序固定，辅助列没有混进主表
- [ ] EDA 每个维度都能对应上一个子问题，且能说出「为什么用中位数/为什么这个分母」
- [ ] 对照表里所有【】都填了真实数字，没有一个是我替你估的
