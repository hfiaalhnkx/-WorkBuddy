# Python 脚本自动化实战：从基础语法到批量处理业务文件.md
日常办公与业务场景里，大量重复文件操作常常消耗大量人力：批量重命名、读取表格、汇总数据、导出报表、分类归档文件。手动点击复制粘贴不仅效率低下，还极易出现人为失误。而 Python 凭借简单易上手的语法，搭配内置文件操作库，不需要复杂的开发环境，就能编写脚本完成各类文件自动化任务。本文从基础语法入手，一步步落地业务文件批量处理实战，适合有少量编程基础或者零基础想实现办公自动化的开发者。

## 一、为什么选择Python做文件自动化
很多人会疑惑，Excel VBA、Power Query同样可以处理文件，为什么还要学习Python脚本？
1. **跨平台兼容**：Windows、Mac、Linux系统都可以直接运行脚本，不受Office版本限制；
2. **能力上限更高**：小到文件重命名、移动归档，大到多表格合并、PDF提取、图片批量处理，都能实现；
3. **学习成本低**：文件操作依靠Python内置模块，不需要复杂框架，少量代码就能实现自动化；
4. **可复用性强**：脚本一次编写，后续只需要修改文件路径，反复执行处理同类业务。

Python 内置 `os`、`pathlib` 模块专门用于文件和目录操作，不需要额外安装依赖，开箱即用，是批量文件处理的核心工具。

## 二、必备基础语法：文件与路径操作
### 1. pathlib 路径操作（推荐）
`pathlib` 是Python3.4+引入的面向对象路径库，相比传统os.path写法更简洁，可读性更强。
```python
from pathlib import Path

# 定义目标文件夹路径
folder = Path(r"D:/business_files")

# 判断文件夹是否存在，不存在则创建
if not folder.exists():
    folder.mkdir()

# 遍历文件夹下所有文件
for file in folder.iterdir():
    if file.is_file():
        print(file.name)
```
简单说明：
- `Path()`：定义文件/文件夹路径，Windows路径使用r字符串避免转义符报错；
- `.exists()`：判断路径是否存在；
- `.mkdir()`：创建文件夹；
- `.is_file()`：判断是否为文件，过滤掉子文件夹。

### 2. os模块基础用法
os模块是老牌文件操作库，大量旧自动化脚本都在使用：
```python
import os

path = r"D:/business_files"
# 列出目录内所有内容
print(os.listdir(path))
```

> 实操建议：新项目优先使用`pathlib`，代码更简洁，减少路径拼接错误。

## 三、实战案例1：批量重命名业务文件
业务场景：文件夹内几十份报表，文件名杂乱，需要统一重命名为`业务报表_序号.xlsx`。
```python
from pathlib import Path

# 目标目录
target_dir = Path(r"D:/business_files/report")
# 筛选xlsx文件
file_list = list(target_dir.glob("*.xlsx"))

for index, file in enumerate(file_list, start=1):
    new_name = f"业务报表_{index}.xlsx"
    new_path = target_dir / new_name
    file.rename(new_path)
print("批量重命名完成")
```
代码解析：
1. `glob("*.xlsx")`：匹配目录下所有后缀为xlsx的文件，支持`*.pdf`、`*.txt`等；
2. `enumerate(start=1)`：从1开始生成序号；
3. `.rename()`：完成文件重命名。

⚠️ 重要提醒：运行脚本前，**先备份原始文件**，防止命名逻辑错误导致文件丢失。

## 四、实战案例2：批量分类归档文件
业务场景：一个文件夹混合存放PDF、Excel、图片，自动按后缀名创建子文件夹，把文件分类移动进去。
```python
from pathlib import Path

source = Path(r"D:/business_files/all_files")

for file in source.iterdir():
    if file.is_file():
        # 获取文件后缀，去掉.
        suffix = file.suffix.replace(".", "")
        # 建立分类文件夹
        dest_folder = source / suffix
        dest_folder.mkdir(exist_ok=True)
        # 移动文件
        file.rename(dest_folder / file.name)
print("文件分类归档完成")
```
`mkdir(exist_ok=True)`：文件夹已存在时不会抛出报错，避免重复运行脚本异常。

## 五、实战案例3：批量读取多个Excel并汇总数据
很多业务场景需要把数十张Excel表格数据合并汇总，借助`pandas`库一行行读取合并。
> 先安装依赖：`pip install pandas openpyxl`
```python
from pathlib import Path
import pandas as pd

source = Path(r"D:/business_files/month_data")
all_data = []

# 遍历所有xlsx
for excel_file in source.glob("*.xlsx"):
    df = pd.read_excel(excel_file)
    all_data.append(df)

# 合并全部数据
result = pd.concat(all_data, ignore_index=True)
# 输出汇总结果
result.to_excel(source / "汇总业务数据.xlsx", index=False)
print("多表格数据汇总完成")
```
这个脚本可以替代手动复制粘贴表格，几百份表格也能在几秒内完成汇总。

## 六、脚本调试与业务落地注意事项
1. **路径问题是最高发错误**
Windows路径反斜杠容易转义报错，推荐两种写法：`r"D:/xxx"` 或者 `D:\\xxx`；尽量不要使用中文路径，部分老旧环境会出现编码异常。

2. **增加日志输出，方便排错**
正式业务脚本不要直接执行操作，可以先打印预览，确认无误再启用文件修改代码。示例：
```python
# 预览模式，只打印，不修改文件
print(f"将要重命名：{file.name} -> {new_name}")
```
确认打印结果正确，再打开`.rename()`执行。

3. **做好文件备份**
凡是涉及文件删除、重命名、移动的脚本，**务必先备份原始文件**，避免脚本逻辑错误造成业务资料丢失。

4. **异常捕获**
业务文件可能存在损坏、被其他软件占用的情况，可以增加try except捕获异常，防止脚本直接中断。
```python
try:
    file.rename(new_path)
except Exception as e:
    print(f"文件 {file.name} 处理失败：{e}")
```

## 七、拓展方向
掌握基础批量文件处理之后，可以继续拓展更多自动化场景：
- 批量读取PDF文档文字内容，提取业务信息；
- 自动发送邮件，把汇总报表定时推送给相关人员；
- 监控文件夹，新增文件自动触发处理脚本；
- 搭配定时任务，实现每日凌晨自动跑批生成业务报表。

## 结语
Python脚本自动化不需要高深的算法知识，核心是用好文件操作相关模块，把重复人工流程转化为代码。从简单的批量改名、文件分类，再到多表格汇总，循序渐进练习，就能把大量机械性业务工作交给脚本执行，大幅减少重复劳动，把时间留给更有价值的业务分析工作。





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
