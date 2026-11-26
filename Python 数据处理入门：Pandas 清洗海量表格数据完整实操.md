# Python 数据处理入门：Pandas 清洗海量表格数据完整实操.md
在日常业务中，Excel、CSV 表格是最常见的数据载体。但原始数据往往存在大量问题：空值、重复行、格式错乱、多余空格、异常字符、列名不统一。当表格行数达到几万甚至几十万行时，手动在 Excel 里筛选、删除、修改，不仅卡顿，还很容易漏改、改错。Pandas 作为 Python 数据分析核心库，可以高效完成海量表格的读取、清洗、转换与导出，把重复性的数据治理工作自动化。

本文从环境准备开始，完整演示一套通用的数据清洗流程，覆盖读取、探查、空值处理、去重、格式修正、筛选、导出，全部代码可直接复制运行。

## 一、环境准备
### 安装依赖
```bash
pip install pandas openpyxl
```
- pandas：核心数据处理库
- openpyxl：用于读写 xlsx 格式文件（读取Excel必须）

> 提示：如果是超大 CSV 文件，不需要 openpyxl，读取速度会比 Excel 快很多。

## 二、读取数据与数据探查（清洗第一步）
拿到原始表格，**不要直接修改**，第一步先查看数据全貌，了解数据质量。
```python
import pandas as pd

# 读取Excel文件
df = pd.read_excel("raw_data.xlsx")
# 读取CSV文件（编码异常可尝试 encoding="gbk"）
# df = pd.read_csv("raw_data.csv", encoding="utf-8")

# 查看前5行
print(df.head())

# 查看数据集行列数量
print(f"数据形状：{df.shape}")

# 查看每一列的数据类型
print(df.dtypes)

# 查看统计信息（数值列最大、最小、均值等）
print(df.describe())

# 查看每一列缺失值数量
print(df.isnull().sum())

# 查看重复行数量
print("重复行数：", df.duplicated().sum())
```
常用探查命令说明：
1. `head()`：预览样本数据，快速发现明显脏数据；
2. `shape`：返回(行数,列数)，知道数据体量；
3. `isnull().sum()`：统计每一列空值，空值是表格最常见问题；
4. `duplicated().sum()`：统计完全重复的记录。

## 三、核心清洗实操
### 1. 重复数据处理
业务表经常因为多次导入产生完全重复的行，直接删除重复行：
```python
# 查看重复数据
print(df[df.duplicated(keep=False)])

# 删除完全重复行，保留第一次出现的数据
df = df.drop_duplicates(keep="first")

# 按指定字段去重，例如只根据【手机号】去重，保留第一条
# df = df.drop_duplicates(subset=["手机号"], keep="first")
```
参数说明：
- `keep="first"`：保留第一条，删除后面重复；
- `keep="last"`：保留最后一条；
- `keep=False`：所有重复项全部删除。

### 2. 缺失值（空值）处理
空值处理没有统一标准，要根据业务场景选择：删除、填充默认值、填充均值。
```python
# 方案1：整行存在空值，直接删除（适合少量空数据）
# df = df.dropna()

# 方案2：指定列存在空值才删除该行
# df = df.dropna(subset=["姓名","订单号"])

# 方案3：填充空值
# 文本列填充未知
df["姓名"] = df["姓名"].fillna("未知")
# 数值列填充0
df["金额"] = df["金额"].fillna(0)
# 数值列填充该列平均值
df["销售额"] = df["销售额"].fillna(df["销售额"].mean())
```
> 业务提醒：**不要无脑 dropna**，海量数据中随意删除行可能丢失有效业务记录。

### 3. 清除文本多余空格、特殊字符
从业务系统导出的数据，单元格经常自带首尾空格，导致匹配失败（`"张三 "` 和 `"张三"` 被判定为两个不同的值）。
```python
# 对文本列去除首尾空格
df["姓名"] = df["姓名"].str.strip()

# 清除全部空格（包含中间）
# df["地址"] = df["地址"].str.replace(" ", "")

# 清除换行符、制表符
df["备注"] = df["备注"].str.replace("\n", "").str.replace("\t", "")
```

### 4. 数据类型转换
很多时候数字被识别为字符串，导致无法求和、排序，需要类型转换。
```python
# 转为整数
df["订单数量"] = df["订单数量"].astype(int)

# 转为浮点数
df["单价"] = df["单价"].astype(float)

# 日期字段转换（非常常用）
df["下单时间"] = pd.to_datetime(df["下单时间"], errors="coerce")
```
`errors="coerce"`：遇到无法转换的脏数据，自动转为 NaT（日期空值），避免脚本直接崩溃。

### 5. 数据筛选，过滤异常数据
过滤超出合理范围的脏数据，例如订单金额不能为负数：
```python
# 筛选金额大于0的有效订单
df_clean = df[df["金额"] > 0]

# 多条件组合：金额>0 并且 地区=广西
df_clean = df[(df["金额"] > 0) & (df["地区"] == "广西")]

# 取反：排除无效状态
df_clean = df[~(df["状态"] == "作废")]
```
> 多条件注意：每个条件必须用括号包裹，条件之间用 `&`（且）、`|`（或）。

### 6. 修改列名、删除不需要的列
原始表格列名经常中英文混杂、带空格，统一简化列名方便后续处理：
```python
# 重命名列
df = df.rename(columns={"用户名称":"姓名","订单总额":"金额"})

# 删除不需要的列
df = df.drop(columns=["临时备注","废弃字段"])
```

## 四、清洗完成，导出结果
清洗完成后，将干净数据输出到新文件，**不要覆盖原始文件**，保留原始数据用于回溯。
```python
# 导出Excel
df_clean.to_excel("clean_data.xlsx", index=False)

# 导出CSV
# df_clean.to_csv("clean_data.csv", index=False, encoding="utf-8-sig")
```
`index=False`：不导出 pandas 自动生成的行索引；
`encoding="utf-8-sig"`：CSV在Windows打开不乱码。

## 五、完整整合脚本（一键清洗模板）
把上面流程整合，形成可复用的完整脚本，每次换表格只需要修改文件名：
```python
import pandas as pd

# 1.读取原始数据
df = pd.read_excel("raw_data.xlsx")

# 2.数据探查
print("原始数据行数：", df.shape[0])
print("缺失值统计：")
print(df.isnull().sum())

# 3.清洗流程
df = df.drop_duplicates()
df["姓名"] = df["姓名"].str.strip().fillna("未知")
df["金额"] = df["金额"].fillna(0)
df["下单时间"] = pd.to_datetime(df["下单时间"], errors="coerce")

# 过滤异常数据
df_clean = df[df["金额"] >= 0]

print("清洗后行数：", df_clean.shape[0])

# 4.导出清洗结果
df_clean.to_excel("clean_data.xlsx", index=False)
print("数据清洗完成，已输出 clean_data.xlsx")
```

## 六、处理海量数据的优化技巧
当数据量达到几十万行，Excel打开卡顿，Pandas也有优化手段：
1. **优先使用CSV**：CSV读写速度远高于xlsx；
2. **分块读取**：超大数据使用 `chunksize` 分批读取，降低内存占用
```python
chunk_iter = pd.read_csv("big_data.csv", chunksize=10000)
for chunk in chunk_iter:
    # 对每一块数据单独清洗处理
    pass
```
3. 指定读取列：只读取需要的字段，减少内存消耗
```python
df = pd.read_excel("raw_data.xlsx", usecols=["姓名","金额","下单时间"])
```

## 七、常见踩坑点
1. **中文编码报错**：CSV读取报错，尝试 `encoding="gbk"`、`encoding="utf-8"`；
2. **空字符串不等于NaN**：单元格是空字符串`""`，`isnull()`识别不出来，需要额外处理：`df.replace("", None)`；
3. **日期解析失败**：脏文本会导致转换报错，务必加上 `errors="coerce"`；
4. **覆盖原始文件**：清洗结果一定要输出新文件，防止原始业务数据丢失。

## 八、拓展方向
基础清洗掌握后，可以继续拓展：
- 数据透视表，按地区、月份汇总统计；
- 多表合并（merge、concat），实现多业务表关联；
- 自动生成数据清洗报告，统计删除行数、空值数量；
- 和前面Python自动化脚本结合：批量读取文件夹所有表格，批量清洗。

## 结语
Pandas数据清洗核心不是复杂代码，而是一套标准化流程：探查数据质量 → 处理重复/空值 → 修正文本与类型 → 过滤异常数据 → 导出干净数据集。几万、几十万行表格，用Pandas只需要几秒完成清洗，告别Excel卡顿、手动处理低效易错的问题。学会这套基础模板，绝大多数日常业务表格治理需求都可以自动化落地。







https://www.720yuntu.com/tour/6169448e316a620d
https://www.720yuntu.com/tour/ec701dc660e8bbb7
https://www.720yuntu.com/tour/21b352ab3fba7739
https://www.720yuntu.com/tour/a837abbfe7a97798
https://www.720yuntu.com/tour/47f07d290af7015b
https://www.720yuntu.com/tour/ee379522f66c1f9b
https://www.720yuntu.com/tour/3173bd745296a5dc
https://www.720yuntu.com/tour/fb254ace763ba6aa
https://www.720yuntu.com/tour/3f76391f66adeffb
https://www.720yuntu.com/tour/276027280bafbdbb
https://www.720yuntu.com/tour/1c99dbd0163cd31f
https://www.720yuntu.com/tour/7b7c13c3c035a378
https://www.720yuntu.com/tour/6279c8362d1ed6f2
https://www.720yuntu.com/tour/80b51b302f40d3ca
https://www.720yuntu.com/tour/d26974a13effed71
https://www.720yuntu.com/tour/17c622917e0832f9
https://www.720yuntu.com/tour/86d83ccda92a56fe
https://www.720yuntu.com/tour/4e0718d68a6916e4
https://www.720yuntu.com/tour/753d2606119be3ec
https://www.720yuntu.com/tour/e5d7fd422e24778c
https://www.720yuntu.com/tour/66c715c88be46afb
https://www.720yuntu.com/tour/7bd7ad7c5568ff71
https://www.720yuntu.com/tour/5e3c60c3898ad2ab
https://www.720yuntu.com/tour/bccb2230c7853c8f
https://www.720yuntu.com/tour/c64272957d0eaa3e
https://www.720yuntu.com/author?pk=131791
https://www.720yuntu.com/tour/f89ab9431afb3687
https://www.720yuntu.com/tour/6c58ac2821ca975e
https://www.720yuntu.com/tour/7521dc86ef4b7496
https://www.720yuntu.com/tour/ff223dda9ea87e14
https://www.720yuntu.com/tour/5a3eacf154e9d6b6
https://www.720yuntu.com/tour/e39e61658a068126
https://www.720yuntu.com/tour/0edb13f146fe9d0f
https://www.720yuntu.com/tour/6eb0dc8915f13351
https://www.720yuntu.com/tour/1facec8f7eee1cd7
https://www.720yuntu.com/tour/8a19996401126a08
https://www.720yuntu.com/tour/cefedf2ab94e68cf
https://www.720yuntu.com/tour/2726e005fc67c6fb
https://www.720yuntu.com/tour/2e9683ec6c6a18fa
https://www.720yuntu.com/tour/b0f97e1a9170a4f0
https://www.720yuntu.com/tour/1692cb42058b2f38
https://www.720yuntu.com/tour/45adc708d7cb21d3
https://www.720yuntu.com/tour/9eefff4b68b18e61
https://www.720yuntu.com/tour/fdf1ed65a638c493
https://www.720yuntu.com/tour/108bd65f9e29b0a5
https://www.720yuntu.com/tour/d61ec4484dc0450a
https://www.720yuntu.com/tour/4d10eeafeb568fe1
https://www.720yuntu.com/tour/8ef5d59306b4ca9a
https://www.720yuntu.com/tour/beaa9f9031937e27
https://www.720yuntu.com/tour/beeec545021da2b7
https://www.720yuntu.com/tour/16392ff0b234c378
https://www.720yuntu.com/tour/fc24246c82c59005
https://www.720yuntu.com/tour/9bcb4ac3f107dbce
https://www.720yuntu.com/tour/e085d6fc6596b997
https://www.720yuntu.com/tour/76f06dc517e47c83
https://www.720yuntu.com/tour/c416bb23d8fb664c
https://www.720yuntu.com/tour/9dd8e6c5d3f7d293
https://www.720yuntu.com/tour/20bb7a1cae2cfa4c
https://www.720yuntu.com/tour/1f96f657a0340431
https://www.720yuntu.com/tour/2cf70f5f02cd1abd
https://www.720yuntu.com/tour/90060b9f05abb7e8
https://www.720yuntu.com/tour/96e46a3705c096ec
https://www.720yuntu.com/tour/c8eaefc660c2d595
https://www.720yuntu.com/tour/0a400ac9dba7bfd3
https://www.720yuntu.com/tour/eb520636f3e34754
https://www.720yuntu.com/tour/1fb783cfdc9214a6
https://www.720yuntu.com/tour/3314e9ddcaea332b
https://www.720yuntu.com/tour/f3276901ee627a0a
https://www.720yuntu.com/tour/0562705021ca0a0e
https://www.720yuntu.com/tour/c37c230e7f69ab12
https://www.720yuntu.com/tour/c128de93df06dcdf
https://www.720yuntu.com/tour/c9239c3ddb3fade6
https://www.720yuntu.com/tour/b8ea52e0979cdeca
https://www.720yuntu.com/tour/64b0225dc5f63ce2
https://www.720yuntu.com/tour/ddba8ec9082354d4
https://www.720yuntu.com/tour/15a5ea89f83beee6
https://www.720yuntu.com/tour/45a6f938d0179845
https://www.720yuntu.com/tour/a4a579cd1a36ba15
https://www.720yuntu.com/tour/f0219327e607077f
https://www.720yuntu.com/tour/f2285498347d1382
https://www.720yuntu.com/tour/aff4c439cca4831c
https://www.720yuntu.com/tour/cccda6de90da1e76
https://www.720yuntu.com/tour/8e2d049590a94da9
https://www.720yuntu.com/tour/544e9a048b3e0bbf
https://www.720yuntu.com/tour/bbce110cbaccf5a3
https://www.720yuntu.com/tour/2e2dfeec876f6fde
https://www.720yuntu.com/tour/fe708545940538e6
https://www.720yuntu.com/tour/9eb0ffda03ee541e
https://www.720yuntu.com/tour/b9541b3fa04111c5
https://www.720yuntu.com/tour/9e0e792126a96397
https://www.720yuntu.com/tour/4238e5698064b9f8
https://www.720yuntu.com/tour/b52053ffdd6f3dce
https://www.720yuntu.com/tour/9173031d8099a3ca
https://www.720yuntu.com/tour/b34bcbaec0cfef7e
https://www.720yuntu.com/tour/9936d308e59a2af3
https://www.720yuntu.com/tour/8b1aba7544c8d530
https://www.720yuntu.com/tour/96c5c3a9da38cd5b
https://www.720yuntu.com/tour/05a090b58a208335
https://www.720yuntu.com/tour/3f94971c91a0a5d8
https://www.720yuntu.com/tour/e3938b9cf91f1fa3
https://www.720yuntu.com/tour/972ed5b225005a72
https://www.720yuntu.com/tour/59ea78decb3697d7
https://www.720yuntu.com/tour/9460363234156875
https://www.720yuntu.com/tour/ad3383d068e5983c
https://www.720yuntu.com/tour/31d3967831b7431a
https://www.720yuntu.com/tour/14c5928080387e04
https://www.720yuntu.com/tour/e8e88d1e391ec45f
https://www.720yuntu.com/tour/2dcee2dc07fe3572
https://www.720yuntu.com/tour/542dcc3583d2ba61
https://www.720yuntu.com/tour/070b074da2f1d0f2
https://www.720yuntu.com/tour/b38d947b7ed9c2dd
https://www.720yuntu.com/tour/19b5f8035de6293f
https://www.720yuntu.com/tour/09bbf5d9c63ac5c7
https://www.720yuntu.com/tour/5ce02b19795f50fd
https://www.720yuntu.com/tour/698c183812c1d5eb
https://www.720yuntu.com/tour/0bcf1e33ecb85c5f
https://www.720yuntu.com/tour/09b2f585661d82d6
https://www.720yuntu.com/tour/ae294bbdb45ae7cc
https://www.720yuntu.com/tour/19e3b51179f4cd82
https://www.720yuntu.com/tour/160a6db2b666f399
https://www.720yuntu.com/tour/0289c1ed71a4cc86
https://vr.justeasy.cn/view/i1p7yg95b6898642-1795689874.html
https://vr.justeasy.cn/view/179tz56m8990ei36-1795689918.html
https://vr.justeasy.cn/view/t1p97t95r689i951-1795689962.html
https://vr.justeasy.cn/view/1r7p9y5m689999o4-1795690006.html
https://vr.justeasy.cn/view/179546j20p90v038-1795690050.html
https://vr.justeasy.cn/view/13h7925669m0t081-1795690093.html
https://vr.justeasy.cn/view/gj187956d9012gr5-1795690137.html
https://vr.justeasy.cn/view/17956s90ye1g6rx9-1795690181.html
https://vr.justeasy.cn/view/177x9591690s2193-1795690225.html
https://vr.justeasy.cn/view/1795p3693k508256-1795690268.html
https://vr.justeasy.cn/view/l179p562903u0ew0-1795690312.html
https://vr.justeasy.cn/view/137n9cfb56903n44-1795690355.html
https://vr.justeasy.cn/view/v192795690384ym7-1795690399.html
https://vr.justeasy.cn/view/17g9k75e6904kz31-1795690442.html
https://vr.justeasy.cn/view/617i950x65490474-1795690486.html
https://vr.justeasy.cn/view/15jl795x69j05n18-1795690529.html
https://vr.justeasy.cn/view/h1sr7ro9569056o1-1795690573.html
https://vr.justeasy.cn/view/1ms57956n9h060t5-1795690616.html
https://vr.justeasy.cn/view/17e9n5yj690kh648-1795690660.html
https://vr.justeasy.cn/view/wd1w795690w62c92-1795690704.html
https://vr.justeasy.cn/view/15795bi6i9073226-1795690748.html
https://vr.justeasy.cn/view/1m79l56y90717v89-1795690791.html
https://vr.justeasy.cn/view/17hu0965690t82t3-1795690835.html
https://vr.justeasy.cn/view/179i5q6p950z86w7-1795690878.html
https://vr.justeasy.cn/view/1vx795e6v90x9i10-1795690922.html
https://vr.justeasy.cn/view/1yj79o56f90m9m54-1795690966.html
https://vr.justeasy.cn/view/1s979569di069397-1795691009.html
https://vr.justeasy.cn/view/1xl676950h691041-1795691053.html
https://vr.justeasy.cn/view/179vd5yh2691w084-1795691096.html
https://vr.justeasy.cn/view/17wm9564ae981128-1795691140.html
https://vr.justeasy.cn/view/on1p79fp56911c72-1795691184.html
https://vr.justeasy.cn/view/z1799c56cg9121y6-1795691228.html
https://vr.justeasy.cn/view/q179lu56j91o2960-1795691272.html
https://vr.justeasy.cn/view/1u795691c30ks784-1795691316.html
https://vr.justeasy.cn/view/d17935l569y1f348-1795691360.html
https://vr.justeasy.cn/view/41ff7s2905691392-1795691404.html
https://vr.justeasy.cn/view/v17f95691j44r736-1795691448.html
https://vr.justeasy.cn/view/17nw956p9k14rv80-1795691492.html
https://vr.justeasy.cn/view/17a956y9155l2oa4-1795691535.html
https://vr.justeasy.cn/view/1b795dc69156bkl8-1795691579.html
https://vr.justeasy.cn/view/179r45xy69pr1612-1795691623.html
https://vr.justeasy.cn/view/1795uj692165cbe5-1795691667.html
https://vr.justeasy.cn/view/n1qrt679569169d8-1795691710.html
https://vr.justeasy.cn/view/t123t79v15691743-1795691754.html
https://vr.justeasy.cn/view/q1p179x5d6991786-1795691798.html
https://vr.justeasy.cn/view/17e95z69108l2fs9-1795691841.html
https://vr.justeasy.cn/view/17h958zi76918793-1795691885.html
https://vr.justeasy.cn/view/179e536f9yq191j7-1795691929.html
https://vr.justeasy.cn/view/bw179569pv1926l0-1795691972.html
https://vr.justeasy.cn/view/s17sz95i694n2004-1795692016.html
https://vr.justeasy.cn/view/1y7956h92w041kn8-1795692062.html
https://vr.justeasy.cn/view/0179ms5692e0l9g4-1795692106.html
https://vr.justeasy.cn/view/179y56ddg90213p8-1795692150.html
https://vr.justeasy.cn/view/17695o6924iw1812-1795692194.html
https://vr.justeasy.cn/view/p1795y689j2n22h5-1795692237.html
https://vr.justeasy.cn/view/4w179569p7xe2269-1795692281.html
https://vr.justeasy.cn/view/1g7956lpne923s12-1795692324.html
https://vr.justeasy.cn/view/1v7c09506923lc56-1795692367.html
https://vr.justeasy.cn/view/17b9569ny0oj2400-1795692412.html
https://vr.justeasy.cn/view/17o69v5ak692o444-1795692456.html
https://vr.justeasy.cn/view/917et95ly692d925-1795692937.html
https://vr.justeasy.cn/view/n17985v568922968-1795692980.html
https://vr.justeasy.cn/view/1m719u5b693e5012-1795693023.html
https://vr.justeasy.cn/view/0179aq56930wo565-1795693067.html
https://vr.justeasy.cn/view/ac179r569h31y060-1795693111.html
https://vr.justeasy.cn/view/8ne17951a6m93143-1795693155.html
https://vr.justeasy.cn/view/jm1l79s5tw693188-1795693199.html
https://vr.justeasy.cn/view/1z7vvk956g932831-1795693243.html
https://vr.justeasy.cn/view/ev1hb7956l932745-1795693287.html
https://vr.justeasy.cn/view/1i7729l556933y19-1795693331.html
https://vr.justeasy.cn/view/1t7ila95cs693363-1795693375.html
https://vr.justeasy.cn/view/z61795bc6dp93407-1795693419.html
https://vr.justeasy.cn/view/l17ra9556934ba51-1795693463.html
https://vr.justeasy.cn/view/1795q6xe9q34dk95-1795693507.html
https://vr.justeasy.cn/view/d1i7r8956935er39-1795693551.html
https://vr.justeasy.cn/view/17gxs95t56935v82-1795693594.html
https://vr.justeasy.cn/view/h1795y69mtx36j26-1795693638.html
https://vr.justeasy.cn/view/17o956b9c3r66lt9-1795693681.html
https://vr.justeasy.cn/view/6179k54g169371g3-1795693724.html
https://vr.justeasy.cn/view/17295m6i69i37536-1795693768.html
https://vr.justeasy.cn/view/17d9569g9u3580x0-1795693812.html
https://vr.justeasy.cn/view/u1795pb6k93l84g4-1795693855.html
https://vr.justeasy.cn/view/1w7h9t569a3d88m8-1795693899.html
https://vr.justeasy.cn/view/179r569h4rnz3931-1795693942.html
https://vr.justeasy.cn/view/179n561az8n93974-1795693986.html
https://vr.justeasy.cn/view/1m7395c6940xb1x8-1795694030.html
https://vr.justeasy.cn/view/1795569vdjt4q061-1795694072.html
https://vr.justeasy.cn/view/1lnt7ul95694y104-1795694116.html
https://vr.justeasy.cn/view/179ixu56f94nm147-1795694159.html
https://vr.justeasy.cn/view/179536t2f9491q90-1795694202.html
https://vr.justeasy.cn/view/1uo795e69ws42234-1795694245.html
https://vr.justeasy.cn/view/17957j69v9942h77-1795694289.html
https://vr.justeasy.cn/view/e179y596b94y32b0-1795694333.html
https://vr.justeasy.cn/view/179tv56r9843z6z4-1795694376.html
https://vr.justeasy.cn/view/1tf14e795694d408-1795694419.html
https://vr.justeasy.cn/view/jws17d956i944b52-1795694464.html
https://vr.justeasy.cn/view/917e956q97ui4495-1795694507.html
https://vr.justeasy.cn/view/17fq99564945hw40-1795694551.html
https://vr.justeasy.cn/view/17e92q5x6945vr83-1795694595.html
https://vr.justeasy.cn/view/17e956kgn7m94627-1795694639.html
https://vr.justeasy.cn/view/1ju79506941f67p1-1795694682.html
https://vr.justeasy.cn/view/17x3wrl954694714-1795694726.html
https://vr.justeasy.cn/view/bh1q7uk956x94758-1795694770.html
https://vr.justeasy.cn/view/1b795o639w4q8y02-1795694814.html
https://vr.justeasy.cn/view/5u13gr7956m94845-1795694857.html
https://vr.justeasy.cn/view/1n79c56i194e88j9-1795694901.html
https://vr.justeasy.cn/view/127wn9569x493v53-1795694945.html
https://vr.justeasy.cn/view/91i795695m49hj76-1795694988.html
https://vr.justeasy.cn/view/179gn5w6z905e019-1795695031.html
https://vr.justeasy.cn/view/1087n956y950dw63-1795695075.html
https://vr.justeasy.cn/view/13v7795e6951ca07-1795695121.html
https://vr.justeasy.cn/view/1m7b9560j95r1453-1795695165.html
https://vr.justeasy.cn/view/pi179l0a56795198-1795695209.html
https://vr.justeasy.cn/view/17956m595u2s4iy1-1795695253.html
https://vr.justeasy.cn/view/1795760n095d28i5-1795695297.html
https://vr.justeasy.cn/view/d179b5569t53m2m9-1795695341.html
https://vr.justeasy.cn/view/17p95we6h9n5s373-1795695385.html
https://vr.justeasy.cn/view/n51j795695g4p1a6-1795695428.html
https://vr.justeasy.cn/view/a179h569l5u446u0-1795695472.html
https://vr.justeasy.cn/view/w1g7k9586v95g504-1795695516.html
https://vr.justeasy.cn/view/1t5795u69c5731u0-1795695742.html
https://vr.justeasy.cn/view/lm1795k689y57754-1795695786.html
https://vr.justeasy.cn/view/179985g6av9581p8-1795695830.html
https://vr.justeasy.cn/view/6127956d9i9v5905-1795695916.html
https://vr.justeasy.cn/view/14795dyc68c95992-1795696004.html
https://vr.justeasy.cn/view/1795690y69ih07a9-1795696092.html
https://vr.justeasy.cn/view/17956v9661682s73-1795696135.html
https://vr.justeasy.cn/view/179rb5691ef61h67-1795696179.html
https://vr.justeasy.cn/view/1w7985a696kd2q10-1795696222.html
https://vr.justeasy.cn/view/170e95z696kds254-1795696265.html
https://vr.justeasy.cn/view/17p95iq7696p2k97-1795696309.html
https://vr.justeasy.cn/view/s1b7956e9b634o41-1795696353.html
https://vr.justeasy.cn/view/11lks79156096429-1795696441.html
https://vr.justeasy.cn/view/17895w6d9k647023-1795696485.html
https://vr.justeasy.cn/view/1079uz50n6c96517-1795696529.html
https://vr.justeasy.cn/view/0179d56fup96u561-1795696573.html
https://vr.justeasy.cn/view/179506896gc6w0a5-1795696616.html
https://vr.justeasy.cn/view/g1l879uh569664g8-1795696660.html
https://vr.justeasy.cn/view/9n1795869e669xn2-1795696704.html
https://vr.justeasy.cn/view/6e2179p5069673a6-1795696748.html
https://vr.justeasy.cn/view/1v7i956996vq78n0-1795696792.html
https://vr.justeasy.cn/view/17f97s57c696l824-1795696836.html
https://vr.justeasy.cn/view/179569686vs4qsd9-1795696880.html
https://vr.justeasy.cn/view/179wv56a96691p52-1795696924.html
https://vr.justeasy.cn/view/1s791d569q6q95y7-1795696968.html
https://vr.justeasy.cn/view/ao1795669700b4x0-1795697012.html
https://vr.justeasy.cn/view/1795n6q59p704zh4-1795697056.html
https://vr.justeasy.cn/view/1769569d7o0d82w8-1795697100.html
https://vr.justeasy.cn/view/0179y5k6m9v7o132-1795697144.html
https://vr.justeasy.cn/view/xi17v19586971e75-1795697187.html
https://vr.justeasy.cn/view/31798456r9i7c218-1795697230.html
https://vr.justeasy.cn/view/31u7p9x569r67262-1795697274.html
https://vr.justeasy.cn/view/91795669a7ko30a7-1795697318.html
https://vr.justeasy.cn/view/1n78j95k6z97j350-1795697362.html
https://vr.justeasy.cn/view/91795m56p9w7a394-1795697407.html
https://vr.justeasy.cn/view/c179h56j97ps43k8-1795697450.html
https://vr.justeasy.cn/view/6179j5s69g74nh82-1795697493.html
https://vr.justeasy.cn/view/1e79d5691s7k5h25-1795697537.html
https://vr.justeasy.cn/view/14p7z8953d697569-1795697581.html
https://vr.justeasy.cn/view/0l17h95697v6s1o3-1795697625.html
https://vr.justeasy.cn/view/1789566s97jr6357-1795697669.html
https://vr.justeasy.cn/view/17g956977k010ry1-1795697713.html
https://vr.justeasy.cn/view/i179u569u7789745-1795697757.html
https://vr.justeasy.cn/view/17o945069c7788c8-1795697800.html
https://vr.justeasy.cn/view/hp179b5ys6978231-1795697843.html
https://vr.justeasy.cn/view/1y87vg9152697875-1795697887.html
https://vr.justeasy.cn/view/179l5n69s7f914h8-1795697930.html
https://vr.justeasy.cn/view/179zw5697961dt72-1795697973.html
https://vr.justeasy.cn/view/1i79h56a9i87w007-1795698018.html
https://vr.justeasy.cn/view/1v795s690td805j0-1795698062.html
https://vr.justeasy.cn/view/1r7b9g56l98e09g3-1795698108.html
https://vr.justeasy.cn/view/p17y95h6a9w81y39-1795698151.html
https://vr.justeasy.cn/view/m17695669z8n18m3-1795698195.html
https://vr.justeasy.cn/view/hy17u9o56982j2i6-1795698237.html
https://vr.justeasy.cn/view/17i39e57698i26l9-1795698281.html
https://vr.justeasy.cn/view/t176m956983n1cl3-1795698324.html
https://vr.justeasy.cn/view/17igc9v569i83596-1795698368.html
https://vr.justeasy.cn/view/17b95698c246wo00-1795698412.html
https://vr.justeasy.cn/view/1u709e5bl6v98444-1795698455.html
https://vr.justeasy.cn/view/1795698hcp48xh57-1795698499.html
https://vr.justeasy.cn/view/2d179569l9u414q6-1795699428.html
https://vr.justeasy.cn/view/1g7o9526m97w9460-1795699472.html
https://vr.justeasy.cn/view/19g795dfz6995c03-1795699515.html
https://vr.justeasy.cn/view/1795du369995w4n7-1795699558.html
https://vr.justeasy.cn/view/o17195pi6995p9b0-1795699602.html
https://vr.justeasy.cn/view/1x79r56t996c3wx4-1795699646.html
https://vr.justeasy.cn/view/179856994u906o77-1795699689.html
https://vr.justeasy.cn/view/1pc7095zf6997h21-1795699732.html
https://vr.justeasy.cn/view/1779vs56l699f764-1795699776.html
https://vr.justeasy.cn/view/l179kij56c99e808-1795699820.html
https://vr.justeasy.cn/view/15m795c6a9c985g2-1795699863.html
https://vr.justeasy.cn/view/1k73oe956p9o9895-1795699906.html
https://vr.justeasy.cn/view/26179r56n99933e8-1795699950.html
https://vr.justeasy.cn/view/17p9n5d369969822-1795699994.html
https://vr.justeasy.cn/view/1437w99570000256-1795700037.html
https://vr.justeasy.cn/view/107m957m0l006yh9-1795700081.html
https://vr.justeasy.cn/view/179a5a37cj00z112-1795700124.html
https://vr.justeasy.cn/view/17957sq0010r45j5-1795700167.html
https://vr.justeasy.cn/view/17o95e70e0w19ns8-1795700210.html
https://vr.justeasy.cn/view/ds41779537004242-1795700253.html
https://vr.justeasy.cn/view/v17952py270x0285-1795700297.html
https://vr.justeasy.cn/view/179b5760hvk032p9-1795700340.html
https://vr.justeasy.cn/view/1k759t57003sc7e2-1795700384.html
https://vr.justeasy.cn/view/u7179587q00041y5-1795700427.html
https://vr.justeasy.cn/view/w1798u57004ayj59-1795700470.html
https://vr.justeasy.cn/view/176d2957p0e050r2-1795700514.html
https://vr.justeasy.cn/view/1bhn7nu95s700546-1795700557.html
https://vr.justeasy.cn/view/1279l5700uh5y8n9-1795700601.html
https://vr.justeasy.cn/view/j17o99f57p006n32-1795700644.html
https://vr.justeasy.cn/view/217b957200h6h7y6-1795700687.html
https://vr.justeasy.cn/view/17957q00x7y01az9-1795700731.html
https://vr.justeasy.cn/view/1zm79570w07gs653-1795700775.html
https://vr.justeasy.cn/view/17r9d57xoq001807-1795700818.html
https://vr.justeasy.cn/view/1pcz795rb7008550-1795700862.html
https://vr.justeasy.cn/view/179s570bk08m29e4-1795700905.html
https://vr.justeasy.cn/view/1h7s9v5700493ez7-1795700948.html
https://vr.justeasy.cn/view/1x7lc9570092hx80-1795700992.html
https://vr.justeasy.cn/view/1sa7gt957la01023-1795701035.html
https://vr.justeasy.cn/view/17d95n70dz10zl67-1795701079.html
https://vr.justeasy.cn/view/21795d730wi11191-1795701123.html
https://vr.justeasy.cn/view/1n5m79e5701i1154-1795701166.html
https://vr.justeasy.cn/view/1795m70111p9ibw9-1795701210.html
https://vr.justeasy.cn/view/17xw9570124tfm22-1795701254.html
https://vr.justeasy.cn/view/179e5a7uu4012r85-1795701297.html
https://vr.justeasy.cn/view/11779570y1032di9-1795701341.html
https://vr.justeasy.cn/view/17957r0ft13u7jn2-1795701384.html
https://vr.justeasy.cn/view/179x57l01416i9c6-1795701428.html
https://vr.justeasy.cn/view/i1n795701lhm4k61-1795701471.html
https://vr.justeasy.cn/view/17bi985702150ci3-1795701515.html
https://vr.justeasy.cn/view/1i7jx957mq01w547-1795701559.html
https://vr.justeasy.cn/view/d15795t70g159b21-1795701605.html
https://vr.justeasy.cn/view/157e9570v16x23q7-1795701649.html
https://vr.justeasy.cn/view/1as7bwq9570e1681-1795701693.html
https://vr.justeasy.cn/view/17950d707173mc24-1795701736.html
https://vr.justeasy.cn/view/rq137957e0z17d68-1795701779.html
https://vr.justeasy.cn/view/17955586c7018111-1795701823.html
https://vr.justeasy.cn/view/x1rx6y7957018v55-1795701866.html
https://vr.justeasy.cn/view/kp1e739a57101898-1795701909.html
https://yuanzhuo.bnu.edu.cn/user/5e0ec5a1b5e0b938043c6465b98e72c2c59436d4/about
https://yuanzhuo.bnu.edu.cn/user/25fbb62d177cd93b9a0cd136261dabc024352123/learn
https://yuanzhuo.bnu.edu.cn/user/8e37857dec3df9b42467aec2c6c3ace1e559f9cf/favorited
https://yuanzhuo.bnu.edu.cn/user/fb95bbd10e60abe166a7b4d9b354fc6181e58eef/group
https://yuanzhuo.bnu.edu.cn/user/bec72e07c74324f31ae41ed6c694b4159f3027f3/following
https://yuanzhuo.bnu.edu.cn/user/7006ff5a695f42cd4cf42690f91195801f21088a/about
https://yuanzhuo.bnu.edu.cn/user/28a8c406da0cf38a805dd1b079949404f900087e/learn
https://yuanzhuo.bnu.edu.cn/user/514f64868baa90b4942973334087e948ba69b28f/favorited
https://yuanzhuo.bnu.edu.cn/user/2f93315695c1e054024e1fd67e9552e5195871d0/group
https://yuanzhuo.bnu.edu.cn/user/e1ac0afe77c9c76930cca3ebf93470598552b332/following
https://yuanzhuo.bnu.edu.cn/user/3c9aebaf3df6d0e3fb1e404d181deae0de511d79/about
https://yuanzhuo.bnu.edu.cn/user/0b203719dbcac33f4591b6425134b1dac9b7922c/learn
https://yuanzhuo.bnu.edu.cn/user/10c513c880c29142b4c3b7e8e200fecc135ef809/favorited
https://yuanzhuo.bnu.edu.cn/user/67ec0a475763298549f33cea5b8896ffc2e436d8/group
https://yuanzhuo.bnu.edu.cn/user/287880694514c552a69fca9f8e8f53960eff362b/following
https://yuanzhuo.bnu.edu.cn/user/7762672daa9b4a0dd479f73c4bd4dccc6317378f/about
https://yuanzhuo.bnu.edu.cn/user/995077e4096c0a651c673e24be23c8b5e686eef1/learn
https://yuanzhuo.bnu.edu.cn/user/895086f66682fbf25e933b52d81c0aa4494bbdb0/favorited
https://yuanzhuo.bnu.edu.cn/user/d1fdbb6e089832a488848ab898168fc95c04a149/group
https://yuanzhuo.bnu.edu.cn/user/0d8ff19c5cb2f0ed14b663f8ef495072195c28c4/following
https://yuanzhuo.bnu.edu.cn/user/9cb57f8d5f3d433a1b0e991f0e4d1c817f5adc48/about
https://yuanzhuo.bnu.edu.cn/user/8f69c0feaa897e87e695877213cddb910bb6d751/learn
https://yuanzhuo.bnu.edu.cn/user/8bd268d7c70118a81f998580a15dc00cc7f02a4b/favorited
https://yuanzhuo.bnu.edu.cn/user/7cc21e3e85d6fed764f263abfe1a8185d565e696/group
https://yuanzhuo.bnu.edu.cn/user/37a6a9d018bd590d327bb94ca5ca423140b7b41f/following
https://yuanzhuo.bnu.edu.cn/user/8831ca23955a336a19fbd4cee3f6346ffbd38b60/about
https://yuanzhuo.bnu.edu.cn/user/a16af7fb3d8f32e70873004a5d1753b611bf118f/learn
https://yuanzhuo.bnu.edu.cn/user/426de889ace1667aac6314081e65c9e4a348b455/favorited
https://yuanzhuo.bnu.edu.cn/user/f8a70ddbe19d8de00a7fcb6290d0c7a36ad1cf5f/group
https://yuanzhuo.bnu.edu.cn/user/86fc2564d797450de5efacb7502f58317b9bb53b/following
https://yuanzhuo.bnu.edu.cn/user/bc54ce6c752aeb32560dc6d8698c69d83011b229/about
https://yuanzhuo.bnu.edu.cn/user/02b30dd9695474a843335465a988498ab3d85102/learn
https://yuanzhuo.bnu.edu.cn/user/6c1db08daf5f5c5787366ec894cca8f348230418/favorited
https://yuanzhuo.bnu.edu.cn/user/5e0ec5a1b5e0b938043c6465b98e72c2c59436d4/group
https://yuanzhuo.bnu.edu.cn/user/25fbb62d177cd93b9a0cd136261dabc024352123/following
https://yuanzhuo.bnu.edu.cn/user/8e37857dec3df9b42467aec2c6c3ace1e559f9cf/about
https://yuanzhuo.bnu.edu.cn/user/fb95bbd10e60abe166a7b4d9b354fc6181e58eef/learn
https://yuanzhuo.bnu.edu.cn/user/bec72e07c74324f31ae41ed6c694b4159f3027f3/favorited
https://yuanzhuo.bnu.edu.cn/user/7006ff5a695f42cd4cf42690f91195801f21088a/group
https://yuanzhuo.bnu.edu.cn/user/28a8c406da0cf38a805dd1b079949404f900087e/following
https://yuanzhuo.bnu.edu.cn/user/514f64868baa90b4942973334087e948ba69b28f/about
https://yuanzhuo.bnu.edu.cn/user/2f93315695c1e054024e1fd67e9552e5195871d0/learn
https://yuanzhuo.bnu.edu.cn/user/e1ac0afe77c9c76930cca3ebf93470598552b332/favorited
https://yuanzhuo.bnu.edu.cn/user/3c9aebaf3df6d0e3fb1e404d181deae0de511d79/group
https://yuanzhuo.bnu.edu.cn/user/0b203719dbcac33f4591b6425134b1dac9b7922c/following
https://yuanzhuo.bnu.edu.cn/user/10c513c880c29142b4c3b7e8e200fecc135ef809/about
https://yuanzhuo.bnu.edu.cn/user/67ec0a475763298549f33cea5b8896ffc2e436d8/learn
https://yuanzhuo.bnu.edu.cn/user/287880694514c552a69fca9f8e8f53960eff362b/favorited
https://yuanzhuo.bnu.edu.cn/user/7762672daa9b4a0dd479f73c4bd4dccc6317378f/group
https://yuanzhuo.bnu.edu.cn/user/995077e4096c0a651c673e24be23c8b5e686eef1/following
https://yuanzhuo.bnu.edu.cn/user/895086f66682fbf25e933b52d81c0aa4494bbdb0/about
https://yuanzhuo.bnu.edu.cn/user/d1fdbb6e089832a488848ab898168fc95c04a149/learn
https://yuanzhuo.bnu.edu.cn/user/0d8ff19c5cb2f0ed14b663f8ef495072195c28c4/favorited
https://yuanzhuo.bnu.edu.cn/user/9cb57f8d5f3d433a1b0e991f0e4d1c817f5adc48/group
https://yuanzhuo.bnu.edu.cn/user/8f69c0feaa897e87e695877213cddb910bb6d751/following
https://yuanzhuo.bnu.edu.cn/user/8bd268d7c70118a81f998580a15dc00cc7f02a4b/about
https://yuanzhuo.bnu.edu.cn/user/7cc21e3e85d6fed764f263abfe1a8185d565e696/learn
https://yuanzhuo.bnu.edu.cn/user/37a6a9d018bd590d327bb94ca5ca423140b7b41f/favorited
https://yuanzhuo.bnu.edu.cn/user/8831ca23955a336a19fbd4cee3f6346ffbd38b60/group
https://yuanzhuo.bnu.edu.cn/user/a16af7fb3d8f32e70873004a5d1753b611bf118f/following
https://yuanzhuo.bnu.edu.cn/user/426de889ace1667aac6314081e65c9e4a348b455/about
https://yuanzhuo.bnu.edu.cn/user/f8a70ddbe19d8de00a7fcb6290d0c7a36ad1cf5f/learn
https://yuanzhuo.bnu.edu.cn/user/86fc2564d797450de5efacb7502f58317b9bb53b/favorited
https://yuanzhuo.bnu.edu.cn/user/bc54ce6c752aeb32560dc6d8698c69d83011b229/group
https://yuanzhuo.bnu.edu.cn/user/02b30dd9695474a843335465a988498ab3d85102/following
https://yuanzhuo.bnu.edu.cn/user/6c1db08daf5f5c5787366ec894cca8f348230418/about
https://yuanzhuo.bnu.edu.cn/user/5e0ec5a1b5e0b938043c6465b98e72c2c59436d4/learn
https://yuanzhuo.bnu.edu.cn/user/25fbb62d177cd93b9a0cd136261dabc024352123/favorited
https://yuanzhuo.bnu.edu.cn/user/8e37857dec3df9b42467aec2c6c3ace1e559f9cf/group
https://yuanzhuo.bnu.edu.cn/user/fb95bbd10e60abe166a7b4d9b354fc6181e58eef/following
https://yuanzhuo.bnu.edu.cn/user/bec72e07c74324f31ae41ed6c694b4159f3027f3/about
https://yuanzhuo.bnu.edu.cn/user/7006ff5a695f42cd4cf42690f91195801f21088a/learn
https://yuanzhuo.bnu.edu.cn/user/28a8c406da0cf38a805dd1b079949404f900087e/favorited
https://yuanzhuo.bnu.edu.cn/user/514f64868baa90b4942973334087e948ba69b28f/group
https://yuanzhuo.bnu.edu.cn/user/2f93315695c1e054024e1fd67e9552e5195871d0/following
https://yuanzhuo.bnu.edu.cn/user/e1ac0afe77c9c76930cca3ebf93470598552b332/about
https://yuanzhuo.bnu.edu.cn/user/3c9aebaf3df6d0e3fb1e404d181deae0de511d79/learn
https://yuanzhuo.bnu.edu.cn/user/0b203719dbcac33f4591b6425134b1dac9b7922c/favorited
https://yuanzhuo.bnu.edu.cn/user/10c513c880c29142b4c3b7e8e200fecc135ef809/group
https://yuanzhuo.bnu.edu.cn/user/67ec0a475763298549f33cea5b8896ffc2e436d8/following
https://yuanzhuo.bnu.edu.cn/user/287880694514c552a69fca9f8e8f53960eff362b/about
https://yuanzhuo.bnu.edu.cn/user/7762672daa9b4a0dd479f73c4bd4dccc6317378f/learn
https://yuanzhuo.bnu.edu.cn/user/995077e4096c0a651c673e24be23c8b5e686eef1/favorited
https://yuanzhuo.bnu.edu.cn/user/895086f66682fbf25e933b52d81c0aa4494bbdb0/group
https://yuanzhuo.bnu.edu.cn/user/d1fdbb6e089832a488848ab898168fc95c04a149/following
https://yuanzhuo.bnu.edu.cn/user/0d8ff19c5cb2f0ed14b663f8ef495072195c28c4/about
https://yuanzhuo.bnu.edu.cn/user/9cb57f8d5f3d433a1b0e991f0e4d1c817f5adc48/learn
https://yuanzhuo.bnu.edu.cn/user/8f69c0feaa897e87e695877213cddb910bb6d751/favorited
https://yuanzhuo.bnu.edu.cn/user/8bd268d7c70118a81f998580a15dc00cc7f02a4b/group
https://yuanzhuo.bnu.edu.cn/user/7cc21e3e85d6fed764f263abfe1a8185d565e696/following
https://yuanzhuo.bnu.edu.cn/user/37a6a9d018bd590d327bb94ca5ca423140b7b41f/about
https://yuanzhuo.bnu.edu.cn/user/8831ca23955a336a19fbd4cee3f6346ffbd38b60/learn
https://yuanzhuo.bnu.edu.cn/user/a16af7fb3d8f32e70873004a5d1753b611bf118f/favorited
https://yuanzhuo.bnu.edu.cn/user/426de889ace1667aac6314081e65c9e4a348b455/group
https://yuanzhuo.bnu.edu.cn/user/f8a70ddbe19d8de00a7fcb6290d0c7a36ad1cf5f/following
https://yuanzhuo.bnu.edu.cn/user/86fc2564d797450de5efacb7502f58317b9bb53b/about
https://yuanzhuo.bnu.edu.cn/user/bc54ce6c752aeb32560dc6d8698c69d83011b229/learn
https://yuanzhuo.bnu.edu.cn/user/02b30dd9695474a843335465a988498ab3d85102/favorited
https://yuanzhuo.bnu.edu.cn/user/6c1db08daf5f5c5787366ec894cca8f348230418/group
https://yuanzhuo.bnu.edu.cn/user/5e0ec5a1b5e0b938043c6465b98e72c2c59436d4/following
https://yuanzhuo.bnu.edu.cn/user/25fbb62d177cd93b9a0cd136261dabc024352123/about
https://yuanzhuo.bnu.edu.cn/user/8e37857dec3df9b42467aec2c6c3ace1e559f9cf/learn
https://yuanzhuo.bnu.edu.cn/user/fb95bbd10e60abe166a7b4d9b354fc6181e58eef/favorited
https://yuanzhuo.bnu.edu.cn/user/bec72e07c74324f31ae41ed6c694b4159f3027f3/group
https://yuanzhuo.bnu.edu.cn/user/7006ff5a695f42cd4cf42690f91195801f21088a/following
https://yuanzhuo.bnu.edu.cn/user/28a8c406da0cf38a805dd1b079949404f900087e/about
https://yuanzhuo.bnu.edu.cn/user/514f64868baa90b4942973334087e948ba69b28f/learn
https://yuanzhuo.bnu.edu.cn/user/2f93315695c1e054024e1fd67e9552e5195871d0/favorited
https://yuanzhuo.bnu.edu.cn/user/e1ac0afe77c9c76930cca3ebf93470598552b332/group
https://yuanzhuo.bnu.edu.cn/user/3c9aebaf3df6d0e3fb1e404d181deae0de511d79/following
https://yuanzhuo.bnu.edu.cn/user/0b203719dbcac33f4591b6425134b1dac9b7922c/about
https://yuanzhuo.bnu.edu.cn/user/10c513c880c29142b4c3b7e8e200fecc135ef809/learn
https://yuanzhuo.bnu.edu.cn/user/67ec0a475763298549f33cea5b8896ffc2e436d8/favorited
https://yuanzhuo.bnu.edu.cn/user/287880694514c552a69fca9f8e8f53960eff362b/group
https://yuanzhuo.bnu.edu.cn/user/7762672daa9b4a0dd479f73c4bd4dccc6317378f/following
https://yuanzhuo.bnu.edu.cn/user/995077e4096c0a651c673e24be23c8b5e686eef1/about
https://yuanzhuo.bnu.edu.cn/user/895086f66682fbf25e933b52d81c0aa4494bbdb0/learn
https://yuanzhuo.bnu.edu.cn/user/d1fdbb6e089832a488848ab898168fc95c04a149/favorited
https://yuanzhuo.bnu.edu.cn/user/0d8ff19c5cb2f0ed14b663f8ef495072195c28c4/group
https://yuanzhuo.bnu.edu.cn/user/9cb57f8d5f3d433a1b0e991f0e4d1c817f5adc48/following
https://yuanzhuo.bnu.edu.cn/user/8f69c0feaa897e87e695877213cddb910bb6d751/about
https://yuanzhuo.bnu.edu.cn/user/8bd268d7c70118a81f998580a15dc00cc7f02a4b/learn
https://yuanzhuo.bnu.edu.cn/user/7cc21e3e85d6fed764f263abfe1a8185d565e696/favorited
https://yuanzhuo.bnu.edu.cn/user/37a6a9d018bd590d327bb94ca5ca423140b7b41f/group
https://yuanzhuo.bnu.edu.cn/user/8831ca23955a336a19fbd4cee3f6346ffbd38b60/following
https://yuanzhuo.bnu.edu.cn/user/a16af7fb3d8f32e70873004a5d1753b611bf118f/about
https://yuanzhuo.bnu.edu.cn/user/426de889ace1667aac6314081e65c9e4a348b455/learn
https://yuanzhuo.bnu.edu.cn/user/f8a70ddbe19d8de00a7fcb6290d0c7a36ad1cf5f/favorited
https://yuanzhuo.bnu.edu.cn/user/86fc2564d797450de5efacb7502f58317b9bb53b/group
https://yuanzhuo.bnu.edu.cn/user/bc54ce6c752aeb32560dc6d8698c69d83011b229/following
https://yuanzhuo.bnu.edu.cn/user/02b30dd9695474a843335465a988498ab3d85102/about
https://yuanzhuo.bnu.edu.cn/user/6c1db08daf5f5c5787366ec894cca8f348230418/learn
https://yuanzhuo.bnu.edu.cn/user/5e0ec5a1b5e0b938043c6465b98e72c2c59436d4/favorited
https://yuanzhuo.bnu.edu.cn/user/25fbb62d177cd93b9a0cd136261dabc024352123/group
https://yuanzhuo.bnu.edu.cn/user/8e37857dec3df9b42467aec2c6c3ace1e559f9cf/following
https://yuanzhuo.bnu.edu.cn/user/fb95bbd10e60abe166a7b4d9b354fc6181e58eef/about
https://yuanzhuo.bnu.edu.cn/user/bec72e07c74324f31ae41ed6c694b4159f3027f3/learn
https://yuanzhuo.bnu.edu.cn/user/7006ff5a695f42cd4cf42690f91195801f21088a/favorited
https://yuanzhuo.bnu.edu.cn/user/28a8c406da0cf38a805dd1b079949404f900087e/group
https://yuanzhuo.bnu.edu.cn/user/514f64868baa90b4942973334087e948ba69b28f/following
https://yuanzhuo.bnu.edu.cn/user/2f93315695c1e054024e1fd67e9552e5195871d0/about
https://yuanzhuo.bnu.edu.cn/user/e1ac0afe77c9c76930cca3ebf93470598552b332/learn
https://yuanzhuo.bnu.edu.cn/user/3c9aebaf3df6d0e3fb1e404d181deae0de511d79/favorited
https://yuanzhuo.bnu.edu.cn/user/0b203719dbcac33f4591b6425134b1dac9b7922c/group
https://yuanzhuo.bnu.edu.cn/user/10c513c880c29142b4c3b7e8e200fecc135ef809/following
https://yuanzhuo.bnu.edu.cn/user/67ec0a475763298549f33cea5b8896ffc2e436d8/about
https://yuanzhuo.bnu.edu.cn/user/287880694514c552a69fca9f8e8f53960eff362b/learn
https://yuanzhuo.bnu.edu.cn/user/7762672daa9b4a0dd479f73c4bd4dccc6317378f/favorited
https://yuanzhuo.bnu.edu.cn/user/995077e4096c0a651c673e24be23c8b5e686eef1/group
https://yuanzhuo.bnu.edu.cn/user/895086f66682fbf25e933b52d81c0aa4494bbdb0/following
https://yuanzhuo.bnu.edu.cn/user/d1fdbb6e089832a488848ab898168fc95c04a149/about
https://yuanzhuo.bnu.edu.cn/user/0d8ff19c5cb2f0ed14b663f8ef495072195c28c4/learn
https://yuanzhuo.bnu.edu.cn/user/9cb57f8d5f3d433a1b0e991f0e4d1c817f5adc48/favorited
https://yuanzhuo.bnu.edu.cn/user/8f69c0feaa897e87e695877213cddb910bb6d751/group
https://yuanzhuo.bnu.edu.cn/user/8bd268d7c70118a81f998580a15dc00cc7f02a4b/following
https://yuanzhuo.bnu.edu.cn/user/7cc21e3e85d6fed764f263abfe1a8185d565e696/about
https://yuanzhuo.bnu.edu.cn/user/37a6a9d018bd590d327bb94ca5ca423140b7b41f/learn
https://yuanzhuo.bnu.edu.cn/user/8831ca23955a336a19fbd4cee3f6346ffbd38b60/favorited
https://yuanzhuo.bnu.edu.cn/user/a16af7fb3d8f32e70873004a5d1753b611bf118f/group
https://yuanzhuo.bnu.edu.cn/user/426de889ace1667aac6314081e65c9e4a348b455/following
https://yuanzhuo.bnu.edu.cn/user/f8a70ddbe19d8de00a7fcb6290d0c7a36ad1cf5f/about
https://yuanzhuo.bnu.edu.cn/user/86fc2564d797450de5efacb7502f58317b9bb53b/learn
https://yuanzhuo.bnu.edu.cn/user/bc54ce6c752aeb32560dc6d8698c69d83011b229/favorited
https://yuanzhuo.bnu.edu.cn/user/02b30dd9695474a843335465a988498ab3d85102/group
https://yuanzhuo.bnu.edu.cn/user/6c1db08daf5f5c5787366ec894cca8f348230418/following
