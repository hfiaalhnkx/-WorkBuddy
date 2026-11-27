# 新手避坑：学习 Python 最容易踩的 5 个运行报错与解决方案.md
很多刚入门Python的同学，语法看懂了，照着教程复制代码，一运行就报红。明明代码看着和示例一模一样，却一直报错，卡在原地反复调试，学习热情直接被消耗。绝大多数新手报错，并不是算法逻辑太难，而是一些细节书写、数据类型、路径问题。本文整理学习Python时最高频的5类运行报错，附带原因、复现代码和完整解决方案，帮你快速定位问题。

## 一、SyntaxError：语法错误
### 报错现象
```
SyntaxError: invalid syntax
```
这是新手遇到最多的报错，直译就是无效语法，Python解释器看不懂你写的代码。
### 常见原因
1. 括号、引号**成对缺失**：少半个`(`、`)`，单双引号只写一边；
2. 中英文符号混用：用中文逗号`，`、中文冒号`：`、中文括号`（）`；
3. if/for/while语句末尾**忘记写冒号 `:`**；
4. 缩进乱删，或者语句换行没有处理。

错误示例：
```python
# 错误：中文冒号 + 缺少右括号
if 10>5：
    print("测试"
```

正确写法：
```python
if 10>5:
    print("测试")
```

✅ 解决办法
1. 优先检查红色箭头指向的那一行，看引号、括号是否成对；
2. 所有代码符号**全部使用英文半角符号**；
3. if、for、while、def 末尾必须加英文冒号；
4. 复制网上代码时，不要连带网页里的中文符号一起粘贴。

## 二、IndentationError：缩进错误
### 报错现象
```
IndentationError: expected an indented block
```
Python不像Java/C++依靠大括号划分代码块，**依靠缩进**。if、for、函数下面的代码必须缩进。

错误示例：
```python
for i in range(3):
print(i) # 没有缩进，直接报错
```

正确示例：
```python
for i in range(3):
    print(i) # 4个空格（推荐）或者Tab缩进，全文统一
```

✅ 解决办法
1. if、for、while、def后面的代码块，必须缩进；
2. 项目内统一缩进风格：**全部4个空格，不要空格和Tab混用**；
3. 复制代码出现缩进错乱，删掉整段缩进，手动重新对齐。

> 小提示：编辑器VS Code可以设置把Tab自动替换为4个空格，避免混用。

## 三、NameError: 变量未定义
### 报错现象
```
NameError: name 'num' is not defined
```
含义：你使用了一个Python找不到名字的变量。

### 常见原因
1. 变量名拼写错误，大小写不一致，Python区分大小写；
2. 变量定义在if/for内部，外部无法访问；
3. 使用变量的代码，写在变量定义代码前面。

错误示例：
```python
print(num)
num = 100
# 代码从上往下执行，print先执行，此时num还没创建
```
另一个坑：大小写区分
```python
Name = "张三"
print(name) # Name 和 name 不是同一个变量！
```

✅ 解决办法
1. 检查变量名拼写，注意大小写；
2. 保证**变量先定义，后使用**；
3. 如果变量写在分支里面，提前初始化变量。
```python
num = 0
if True:
    num = 100
print(num)
```

## 四、TypeError：类型错误
### 报错现象
```
TypeError: can only concatenate str (not "int") to str
```
最经典场景：字符串和数字直接相加。

错误示例：
```python
age = 18
print("年龄是：" + age)
# 字符串不能直接 + 数字
```

解决方案三种写法任选其一
```python
age = 18
# 方案1：数字转为字符串
print("年龄是：" + str(age))
# 方案2：f-string（推荐，简洁好用）
print(f"年龄是：{age}")
# 方案3：format格式化
print("年龄是：{}".format(age))
```

其他TypeError场景：
- 调用函数时传入参数数量不对；
- 试图修改不可变类型，例如字符串`"abc"[0] = "x"`。

✅ 解决办法
看报错提示，分清变量类型，用`print(type(变量))`打印类型排查；不同类型不能直接运算，按需转换。

## 五、FileNotFoundError：文件找不到
### 报错现象
```
FileNotFoundError: [Errno 2] No such file or directory: 'data.xlsx'
```
做文件自动化、Pandas读取表格时高频报错，新手以为文件在文件夹里就一定能读到。

### 常见原因
1. 使用相对路径，脚本运行目录和文件存放目录不一致；
2. 文件名写错，大小写、后缀名遗漏（`data`和`data.xlsx`不是同一个文件）；
3. Windows路径反斜杠转义问题。

错误示例
```python
# 直接写文件名，脚本不在这个文件夹就找不到
df = pd.read_excel("data.xlsx")
```

两种稳妥解决方式：
1. 使用原始字符串r""写绝对路径
```python
import pandas as pd
df = pd.read_excel(r"D:\python_project\data.xlsx")
```
2. pathlib自动处理路径（推荐，跨平台）
```python
from pathlib import Path
file = Path(r"D:/python_project/data.xlsx")
df = pd.read_excel(file)
```

✅ 排查小技巧：打印当前工作目录，确认程序在哪里查找文件
```python
import os
print(os.getcwd())
```

## 六、通用排错思路（所有报错都适用）
1. **优先看报错第一行**，定位出错行号；
2. 阅读报错英文提示，看懂关键词，比如NameError就是名字问题，FileNotFound就是找不到文件；
3. 最小化测试：注释大部分代码，只保留报错部分单独运行；
4. 增加print打印中间变量，看变量的值和类型，判断是否和预期一致；
5. 不要一次性复制几十行代码运行，分段写，分段测试。

## 七、拓展避坑建议
除了上面5个高频报错，新手还会遇到：
- KeyError：字典访问不存在的key；
- IndexError：列表下标超出范围；
- ModuleNotFoundError：忘记pip安装库，比如直接import pandas但没安装。

## 结语
学习Python，遇到报错不是写得差，而是必经过程。大部分报错都有固定规律，掌握这5类高频错误，看懂报错信息，就能大幅减少调试时间。遇到程序报红不要直接复制整段报错丢给搜索引擎，先看行号、读懂错误类型，自己动手排查，慢慢就能建立排错思维，提升编码能力。






https://yuanzhuo.bnu.edu.cn/user/e4985f6326f929a33b5be50c6a43ff884d2c5427/about
https://yuanzhuo.bnu.edu.cn/user/0a16ebabc75390fe30c2f92b4406de6a42638a8d/learn
https://yuanzhuo.bnu.edu.cn/user/5688bb970401536f26657b93cdd31107befe4cfa/favorited
https://yuanzhuo.bnu.edu.cn/user/dde357012c626d368c9e50613ec672eeb0454c93/group
https://yuanzhuo.bnu.edu.cn/user/28d2ca89ac5fac5053c735ba7060db41429769a7/following
https://yuanzhuo.bnu.edu.cn/user/74b1338b8a7d225e88f84fa30f1506032b7d70de/about
https://yuanzhuo.bnu.edu.cn/user/eeee3019c044bf99e3a3c556333c110e8c4d2a3c/learn
https://yuanzhuo.bnu.edu.cn/user/689cd59809eb1dbd6c7fb57ddf48013cbc063969/favorited
https://yuanzhuo.bnu.edu.cn/user/2d1616cb10144b043699947fc14a9ce2643a1cc9/group
https://yuanzhuo.bnu.edu.cn/user/c854036f8126130fb86d1e2f70e9b1d0264459b6/following
https://yuanzhuo.bnu.edu.cn/user/13ccc0fc1959b9ad0b712fcd39aca6c120b15f86/about
https://yuanzhuo.bnu.edu.cn/user/df4193f4088a98191830f0be52827c3b012eaaef/learn
https://yuanzhuo.bnu.edu.cn/user/dd05d4950e2182c8b1f505b561eecab5cd8b9c2d/favorited
https://yuanzhuo.bnu.edu.cn/user/15d7059a04497270bcea87e51494f48123718caa/group
https://yuanzhuo.bnu.edu.cn/user/293f0fdd1b607f62cefd2a3602f421bc35e65aa9/following
https://yuanzhuo.bnu.edu.cn/user/9809e9615291079ed820a95a5203414c870ea2b7/about
https://yuanzhuo.bnu.edu.cn/user/fc6559c9d2b1af0e15d5c17243e067d0c040ee82/learn
https://yuanzhuo.bnu.edu.cn/user/20692105c449fcfe74378d57b9c190ebe7d1bbc4/favorited
https://yuanzhuo.bnu.edu.cn/user/8a7130e49a31358fab760e3e2f6124102000ed38/group
https://yuanzhuo.bnu.edu.cn/user/e4985f6326f929a33b5be50c6a43ff884d2c5427/following
https://yuanzhuo.bnu.edu.cn/user/0a16ebabc75390fe30c2f92b4406de6a42638a8d/about
https://yuanzhuo.bnu.edu.cn/user/5688bb970401536f26657b93cdd31107befe4cfa/learn
https://yuanzhuo.bnu.edu.cn/user/dde357012c626d368c9e50613ec672eeb0454c93/favorited
https://yuanzhuo.bnu.edu.cn/user/28d2ca89ac5fac5053c735ba7060db41429769a7/group
https://yuanzhuo.bnu.edu.cn/user/74b1338b8a7d225e88f84fa30f1506032b7d70de/following
https://yuanzhuo.bnu.edu.cn/user/eeee3019c044bf99e3a3c556333c110e8c4d2a3c/about
https://yuanzhuo.bnu.edu.cn/user/689cd59809eb1dbd6c7fb57ddf48013cbc063969/learn
https://yuanzhuo.bnu.edu.cn/user/2d1616cb10144b043699947fc14a9ce2643a1cc9/favorited
https://yuanzhuo.bnu.edu.cn/user/c854036f8126130fb86d1e2f70e9b1d0264459b6/group
https://yuanzhuo.bnu.edu.cn/user/13ccc0fc1959b9ad0b712fcd39aca6c120b15f86/following
https://yuanzhuo.bnu.edu.cn/user/df4193f4088a98191830f0be52827c3b012eaaef/about
https://yuanzhuo.bnu.edu.cn/user/dd05d4950e2182c8b1f505b561eecab5cd8b9c2d/learn
https://yuanzhuo.bnu.edu.cn/user/15d7059a04497270bcea87e51494f48123718caa/favorited
https://yuanzhuo.bnu.edu.cn/user/293f0fdd1b607f62cefd2a3602f421bc35e65aa9/group
https://yuanzhuo.bnu.edu.cn/user/9809e9615291079ed820a95a5203414c870ea2b7/following
https://yuanzhuo.bnu.edu.cn/user/fc6559c9d2b1af0e15d5c17243e067d0c040ee82/about
https://yuanzhuo.bnu.edu.cn/user/20692105c449fcfe74378d57b9c190ebe7d1bbc4/learn
https://yuanzhuo.bnu.edu.cn/user/8a7130e49a31358fab760e3e2f6124102000ed38/favorited
https://yuanzhuo.bnu.edu.cn/user/e4985f6326f929a33b5be50c6a43ff884d2c5427/group
https://yuanzhuo.bnu.edu.cn/user/0a16ebabc75390fe30c2f92b4406de6a42638a8d/following
https://yuanzhuo.bnu.edu.cn/user/5688bb970401536f26657b93cdd31107befe4cfa/about
https://yuanzhuo.bnu.edu.cn/user/dde357012c626d368c9e50613ec672eeb0454c93/learn
https://yuanzhuo.bnu.edu.cn/user/28d2ca89ac5fac5053c735ba7060db41429769a7/favorited
https://yuanzhuo.bnu.edu.cn/user/74b1338b8a7d225e88f84fa30f1506032b7d70de/group
https://yuanzhuo.bnu.edu.cn/user/eeee3019c044bf99e3a3c556333c110e8c4d2a3c/following
https://yuanzhuo.bnu.edu.cn/user/689cd59809eb1dbd6c7fb57ddf48013cbc063969/about
https://yuanzhuo.bnu.edu.cn/user/2d1616cb10144b043699947fc14a9ce2643a1cc9/learn
https://yuanzhuo.bnu.edu.cn/user/c854036f8126130fb86d1e2f70e9b1d0264459b6/favorited
https://yuanzhuo.bnu.edu.cn/user/13ccc0fc1959b9ad0b712fcd39aca6c120b15f86/group
https://yuanzhuo.bnu.edu.cn/user/df4193f4088a98191830f0be52827c3b012eaaef/following
https://yuanzhuo.bnu.edu.cn/user/dd05d4950e2182c8b1f505b561eecab5cd8b9c2d/about
https://yuanzhuo.bnu.edu.cn/user/15d7059a04497270bcea87e51494f48123718caa/learn
https://yuanzhuo.bnu.edu.cn/user/293f0fdd1b607f62cefd2a3602f421bc35e65aa9/favorited
https://yuanzhuo.bnu.edu.cn/user/9809e9615291079ed820a95a5203414c870ea2b7/group
https://yuanzhuo.bnu.edu.cn/user/fc6559c9d2b1af0e15d5c17243e067d0c040ee82/following
https://yuanzhuo.bnu.edu.cn/user/20692105c449fcfe74378d57b9c190ebe7d1bbc4/about
https://yuanzhuo.bnu.edu.cn/user/8a7130e49a31358fab760e3e2f6124102000ed38/learn
https://yuanzhuo.bnu.edu.cn/user/e4985f6326f929a33b5be50c6a43ff884d2c5427/favorited
https://yuanzhuo.bnu.edu.cn/user/0a16ebabc75390fe30c2f92b4406de6a42638a8d/group
https://yuanzhuo.bnu.edu.cn/user/5688bb970401536f26657b93cdd31107befe4cfa/following
https://yuanzhuo.bnu.edu.cn/user/dde357012c626d368c9e50613ec672eeb0454c93/about
https://yuanzhuo.bnu.edu.cn/user/28d2ca89ac5fac5053c735ba7060db41429769a7/learn
https://yuanzhuo.bnu.edu.cn/user/74b1338b8a7d225e88f84fa30f1506032b7d70de/favorited
https://yuanzhuo.bnu.edu.cn/user/eeee3019c044bf99e3a3c556333c110e8c4d2a3c/group
https://yuanzhuo.bnu.edu.cn/user/689cd59809eb1dbd6c7fb57ddf48013cbc063969/following
https://yuanzhuo.bnu.edu.cn/user/2d1616cb10144b043699947fc14a9ce2643a1cc9/about
https://yuanzhuo.bnu.edu.cn/user/c854036f8126130fb86d1e2f70e9b1d0264459b6/learn
https://yuanzhuo.bnu.edu.cn/user/13ccc0fc1959b9ad0b712fcd39aca6c120b15f86/favorited
https://yuanzhuo.bnu.edu.cn/user/df4193f4088a98191830f0be52827c3b012eaaef/group
https://yuanzhuo.bnu.edu.cn/user/dd05d4950e2182c8b1f505b561eecab5cd8b9c2d/following
https://yuanzhuo.bnu.edu.cn/user/15d7059a04497270bcea87e51494f48123718caa/about
https://yuanzhuo.bnu.edu.cn/user/293f0fdd1b607f62cefd2a3602f421bc35e65aa9/learn
https://yuanzhuo.bnu.edu.cn/user/9809e9615291079ed820a95a5203414c870ea2b7/favorited
https://yuanzhuo.bnu.edu.cn/user/fc6559c9d2b1af0e15d5c17243e067d0c040ee82/group
https://yuanzhuo.bnu.edu.cn/user/20692105c449fcfe74378d57b9c190ebe7d1bbc4/following
https://yuanzhuo.bnu.edu.cn/user/8a7130e49a31358fab760e3e2f6124102000ed38/about
https://yuanzhuo.bnu.edu.cn/user/e4985f6326f929a33b5be50c6a43ff884d2c5427/learn
https://yuanzhuo.bnu.edu.cn/user/0a16ebabc75390fe30c2f92b4406de6a42638a8d/favorited
https://yuanzhuo.bnu.edu.cn/user/5688bb970401536f26657b93cdd31107befe4cfa/group
https://yuanzhuo.bnu.edu.cn/user/dde357012c626d368c9e50613ec672eeb0454c93/following
https://yuanzhuo.bnu.edu.cn/user/28d2ca89ac5fac5053c735ba7060db41429769a7/about
https://yuanzhuo.bnu.edu.cn/user/74b1338b8a7d225e88f84fa30f1506032b7d70de/learn
https://yuanzhuo.bnu.edu.cn/user/eeee3019c044bf99e3a3c556333c110e8c4d2a3c/favorited
https://yuanzhuo.bnu.edu.cn/user/689cd59809eb1dbd6c7fb57ddf48013cbc063969/group
https://yuanzhuo.bnu.edu.cn/user/2d1616cb10144b043699947fc14a9ce2643a1cc9/following
https://yuanzhuo.bnu.edu.cn/user/c854036f8126130fb86d1e2f70e9b1d0264459b6/about
https://yuanzhuo.bnu.edu.cn/user/13ccc0fc1959b9ad0b712fcd39aca6c120b15f86/learn
https://yuanzhuo.bnu.edu.cn/user/df4193f4088a98191830f0be52827c3b012eaaef/favorited
https://yuanzhuo.bnu.edu.cn/user/dd05d4950e2182c8b1f505b561eecab5cd8b9c2d/group
https://yuanzhuo.bnu.edu.cn/user/15d7059a04497270bcea87e51494f48123718caa/following
https://yuanzhuo.bnu.edu.cn/user/293f0fdd1b607f62cefd2a3602f421bc35e65aa9/about
https://yuanzhuo.bnu.edu.cn/user/9809e9615291079ed820a95a5203414c870ea2b7/learn
https://yuanzhuo.bnu.edu.cn/user/fc6559c9d2b1af0e15d5c17243e067d0c040ee82/favorited
https://yuanzhuo.bnu.edu.cn/user/20692105c449fcfe74378d57b9c190ebe7d1bbc4/group
https://yuanzhuo.bnu.edu.cn/user/8a7130e49a31358fab760e3e2f6124102000ed38/following
