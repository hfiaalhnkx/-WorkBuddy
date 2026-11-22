# Python 实战：从脚本到工程化，代码可维护性改造全流程.md
## 前言
很多Python开发者的起步，都是从写一段独立脚本开始。
写脚本阶段，目标很简单：**功能跑通就行**。几十行、几百行代码堆在单个`demo.py`文件里，全局变量随处定义，硬编码写死文件路径、接口密钥，没有异常捕获，没有日志，更没有单元测试。一次性任务、临时数据处理脚本这么写无可厚非。

但随着业务迭代，脚本会不断膨胀：从几百行涨到几千行，新增功能不断往同一个文件追加代码。到最后就会出现典型“屎山脚本”现象：改一处逻辑，多处连带报错；换一台机器运行，直接环境缺失；半年后回看自己写的代码，完全看不懂当初的思路；交付给同事维护时，需要花大量时间读懂杂乱的逻辑。

这就是**脚本和工程化项目的核心差距**：脚本优先满足“一次性运行”；工程化项目优先满足**可阅读、可复用、可测试、易部署、方便多人长期迭代**。

本文将基于真实改造案例，完整演示如何把一段简陋Python脚本，一步步重构为规范的工程化项目，覆盖目录结构重构、代码解耦、配置分离、日志、异常处理、依赖管理、单元测试等全套实操。

## 一、改造前：原始脚本现状
下面是典型的原始单文件脚本，功能：读取csv用户数据，过滤无效行，调用接口清洗数据，输出结果文件。
```python
# data_clean.py
import csv
import requests

# 硬编码配置
file_path = "user.csv"
out_path = "result.csv"
api_url = "https://xxx/api/clean"

def main():
    res_list = []
    with open(file_path, "r", encoding="utf-8") as f:
        reader = csv.DictReader(f)
        for row in reader:
            # 没有异常捕获，接口失败程序直接崩溃
            r = requests.post(api_url, json=row)
            if r.status_code == 200:
                res_list.append(r.json())
    # 写入结果
    with open(out_path, "w", encoding="utf-8", newline="") as fw:
        writer = csv.DictWriter(fw, fieldnames=res_list[0].keys())
        writer.writeheader()
        writer.writerows(res_list)

if __name__ == "__main__":
    main()
```

### 原始脚本存在的问题
1. **配置硬编码**：文件路径、接口地址全部写死在代码内，更换环境需要修改源码，容易误提交。
2. **缺少异常捕获**：文件不存在、网络超时、接口报错、空数据都会直接抛出异常，程序中断，无任何记录。
3. **无日志输出**：运行过程看不到进度，出错不知道是哪一行、哪一条数据失败，排查困难。
4. **代码耦合严重**：文件读取、网络请求、数据处理、结果写入全部写在同一流程，无法单独复用函数。
5. **没有依赖声明**：别人拿到代码，不知道需要安装哪个版本的requests，环境版本混乱。
6. **无单元测试**：修改过滤逻辑后，无法快速验证是否破坏原有功能。
7. **缺少类型标注**：函数入参、返回值类型不明确，IDE提示弱，多人协作容易传错参数。

> 小脚本看不出问题，一旦业务迭代，这些缺陷会持续放大，后期重构成本极高。

## 二、第一步：搭建标准化工程目录结构
工程化改造第一件事，不是直接改代码，而是**拆分目录，按职责划分文件**。
改造后的项目结构：
```
data_clean_project/
├── src/                      # 源代码目录
│   ├── data_reader.py        # 文件读取模块
│   ├── data_processor.py     # 数据清洗逻辑
│   ├── api_client.py         # 接口请求封装
│   ├── data_writer.py        # 结果写入模块
│   └── main.py               # 程序入口
├── config/
│   └── settings.yaml         # 配置文件（路径、接口地址）
├── logs/                     # 日志存放目录
├── tests/                    # 单元测试
│   ├── test_reader.py
│   └── test_processor.py
├── .gitignore                # git忽略文件（日志、缓存、环境变量）
├── requirements.txt          # 依赖清单
└── README.md                 # 项目说明文档
```

目录设计思路：**单一职责原则**，一个模块只负责一件事。
- `api_client.py`：只负责网络请求，超时、重试封装；数据逻辑不放在这里
- `data_reader.py`：只做文件读取，支持csv，后续可扩展excel，上层业务不用关心底层读取细节
- `data_processor.py`：纯数据业务逻辑，过滤、字段转换，**不操作文件、不调用网络**，方便单元测试
- `main.py`：组装各个模块，控制整体流程

## 三、第二步：配置分离，解除硬编码
不要把配置写进代码，推荐使用yaml或者pydantic-settings管理配置。
安装依赖：`pip install pydantic pydantic-settings`

`config/settings.yaml`
```yaml
file:
  input_path: "user.csv"
  output_path: "result.csv"
api:
  url: "https://xxx/api/clean"
  timeout: 10
```

读取配置 `src/settings.py`
```python
from pydantic_settings import BaseSettings, SettingsConfigDict
import yaml
from pathlib import Path

class AppSettings(BaseSettings):
    model_config = SettingsConfigDict(yaml_file="config/settings.yaml")
    file: dict
    api: dict

def load_config() -> AppSettings:
    cfg_path = Path("config/settings.yaml")
    with open(cfg_path, "r", encoding="utf-8") as f:
        raw = yaml.safe_load(f)
    return AppSettings(**raw)
```

> 生产环境敏感信息（密钥、token）不要写在yaml明文配置，改用环境变量读取，避免提交到代码仓库。

## 四、第三步：代码解耦 + 类型注解 + 异常封装
按照单一职责拆分模块，增加类型注解，自定义异常，区分业务错误和系统错误。

### src/api_client.py
```python
import requests
from typing import Dict
from requests.exceptions import RequestException

class ApiRequestError(Exception):
    """自定义接口请求异常"""
    pass

class DataCleanApiClient:
    def __init__(self, api_url: str, timeout: int):
        self.api_url = api_url
        self.timeout = timeout

    def clean_data(self, row: Dict) -> Dict:
        try:
            resp = requests.post(self.api_url, json=row, timeout=self.timeout)
            resp.raise_for_status()
            return resp.json()
        except RequestException as e:
            raise ApiRequestError(f"接口调用失败：{str(e)}") from e
```

### src/data_reader.py
```python
import csv
from pathlib import Path
from typing import List, Dict

def read_csv(file_path: str) -> List[Dict]:
    path = Path(file_path)
    if not path.exists():
        raise FileNotFoundError(f"输入文件不存在：{file_path}")
    result = []
    with open(path, "r", encoding="utf-8") as f:
        reader = csv.DictReader(f)
        for line in reader:
            result.append(line)
    return result
```

### src/data_writer.py
```python
import csv
from pathlib import Path
from typing import List, Dict

def write_csv(file_path: str, data: List[Dict]):
    if not data:
        return
    path = Path(file_path)
    with open(path, "w", encoding="utf-8", newline="") as f:
        writer = csv.DictWriter(f, fieldnames=data[0].keys())
        writer.writeheader()
        writer.writerows(data)
```

### src/main.py（程序入口，增加日志）
```python
import logging
from settings import load_config
from data_reader import read_csv
from data_writer import write_csv
from api_client import DataCleanApiClient, ApiRequestError

# 日志配置
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(levelname)s - %(message)s",
    handlers=[
        logging.FileHandler("logs/run.log", encoding="utf-8"),
        logging.StreamHandler()
    ]
)
logger = logging.getLogger(__name__)

def main():
    cfg = load_config()
    client = DataCleanApiClient(
        api_url=cfg.api["url"],
        timeout=cfg.api["timeout"]
    )
    try:
        logger.info("开始读取原始数据")
        rows = read_csv(cfg.file["input_path"])
        logger.info(f"读取到 {len(rows)} 条原始数据")
        output_data = []
        for idx, item in enumerate(rows):
            try:
                cleaned = client.clean_data(item)
                output_data.append(cleaned)
                logger.debug(f"第{idx+1}条数据处理成功")
            except ApiRequestError as e:
                logger.error(f"第{idx+1}条数据接口异常：{e}")
                continue
        write_csv(cfg.file["output_path"], output_data)
        logger.info(f"任务完成，共输出 {len(output_data)} 条清洗后数据")
    except Exception as e:
        logger.exception("程序运行发生致命错误")

if __name__ == "__main__":
    main()
```

改造后代码优势：
1. 模块独立，函数职责清晰，方便单独调用；
2. 自定义异常分层捕获，单条数据失败不会中断整个任务；
3. 完整日志：控制台+日志文件，记录进度与报错堆栈；
4. 类型注解清晰，IDE自动提示，降低协作出错概率。

## 五、第四步：依赖管理与环境隔离
原始脚本最大痛点之一：环境不一致。
`requirements.txt`，锁定依赖包名称：
```txt
requests>=2.31.0
pyyaml>=6.0
pydantic>=2.0
pydantic-settings>=2.0
```
> 进阶工程推荐使用`poetry`或者`pipenv`管理，自动锁定版本`lock`文件，保证开发、测试、生产环境依赖版本完全一致。

`.gitignore`
```
# 日志
logs/*.log
# python缓存
__pycache__/
*.pyc
# 环境文件夹
venv/
.venv/
# 本地配置
config/local.yaml
# 输出文件
*.csv
```

README.md 是工程化必不可少的文档，示例：
```
# 数据清洗工具
功能：读取csv用户数据，调用清洗接口，输出清洗结果
## 环境安装
python >=3.10
pip install -r requirements.txt
## 使用方法
1. 修改config/settings.yaml配置
2. python src/main.py
## 日志
日志输出目录 logs/run.log
```

## 六、第五步：增加单元测试，保障迭代稳定性
工程化项目，核心业务逻辑必须写单元测试。
安装 pytest：`pip install pytest`

`tests/test_processor.py`
```python
import pytest
from src.data_reader import read_csv

def test_read_csv():
    # 可以准备mock测试文件，验证读取逻辑
    pass
```
执行测试命令：`pytest tests/ -v`。
每次修改数据逻辑之后，执行单元测试，快速验证是否引入bug，避免回归问题。

> 针对接口调用，单元测试阶段不要请求真实线上接口，使用`requests-mock`做mock模拟，保证测试稳定、不消耗线上资源。

## 七、进阶优化方向（中大型项目推荐）
完成上面基础改造后，项目已经脱离“脚本”范畴，如果后续持续迭代，可以继续升级：
1. **命令行参数支持**：使用`argparse` / `typer`，支持运行时传入输入输出文件，不用修改配置文件；
2. **异步改造**：数据量大、接口多的场景，将同步requests替换为aiohttp，并发请求提升处理速度；
3. **数据校验**：使用pydantic模型校验每行数据字段格式，提前拦截脏数据；
4. **进度条**：tqdm增加控制台进度展示；
5. **Docker容器化**：编写Dockerfile，打包项目，实现一键部署，彻底解决环境问题；
6. **CI自动化**：提交代码自动运行pytest，代码规范检查（ruff）。

## 八、改造总结：脚本转工程化的核心原则
很多人误以为工程化就是写很多复杂框架，其实并不是。
**工程化不是炫技，而是为了降低长期维护成本。**
改造核心要点：
1. 职责拆分：不同能力放到不同模块，拒绝超大单文件；
2. 配置和代码分离，敏感信息保护；
3. 日志+异常，让程序“会说话”，方便排障；
4. 依赖版本管理，消除环境差异；
5. 单元测试，保障迭代不踩回归坑；
6. 补充文档，方便后续接手维护。

不是所有Python代码都需要工程化。一次性临时脚本，几十行代码，完全没必要这么复杂。**判断标准：代码是否需要多次运行、长期迭代、多人协作。**
只要脚本需要反复使用，迟早要重构，越早按照工程化规范编写，后期节省的时间越多。







https://vr.justeasy.cn/view/179cp5111p9m6771-1795196782.html
https://vr.justeasy.cn/view/17e95199b5684q99-1795196911.html
https://vr.justeasy.cn/view/m017951k9h69yu41-1795196953.html
https://vr.justeasy.cn/view/17eh951y96hev983-1795196995.html
https://vr.justeasy.cn/view/11t7895i19u70726-1795197037.html
https://vr.justeasy.cn/view/1t7w9510970vq368-1795197080.html
https://vr.justeasy.cn/view/179th5r1987s1m10-1795197122.html
https://vr.justeasy.cn/view/j1j7oi95194715w3-1795197165.html
https://vr.justeasy.cn/view/179519a271wl4i95-1795197207.html
https://vr.justeasy.cn/view/171kz9n5197j23s8-1795197250.html
https://vr.justeasy.cn/view/17c951797229u381-1795197292.html
https://vr.justeasy.cn/view/1769j50x19t7e323-1795197335.html
https://vr.justeasy.cn/view/h21kq72951973c66-1795197377.html
https://vr.justeasy.cn/view/i1795l194724x0q8-1795197420.html
https://vr.justeasy.cn/view/1769519nrn790451-1795197462.html
https://vr.justeasy.cn/view/l17951g97154kp93-1795197505.html
https://vr.justeasy.cn/view/17l9954197hw5w36-1795197547.html
https://vr.justeasy.cn/view/11795d1n9752zq78-1795197589.html
https://vr.justeasy.cn/view/16879k5t19762ju0-1795197632.html
https://vr.justeasy.cn/view/lt11795h19t76y63-1795197674.html
https://vr.justeasy.cn/view/179ww51977p9p605-1795197717.html
https://vr.justeasy.cn/view/17z951g9974hp748-1795197760.html
https://vr.justeasy.cn/view/17p95e3197k79241-1795197802.html
https://vr.justeasy.cn/view/aco1j79g51b97833-1795197845.html
https://vr.justeasy.cn/view/17p9054d1967n876-1795197887.html
https://vr.justeasy.cn/view/1m795ndxj9197918-1795197929.html
https://vr.justeasy.cn/view/hw1795197u960fi0-1795197972.html
https://vr.justeasy.cn/view/21795ds631998003-1795198014.html
https://vr.justeasy.cn/view/179oc5h198ml04q5-1795198056.html
https://vr.justeasy.cn/view/179b251fn94808j8-1795198099.html
https://vr.justeasy.cn/view/17l958o1968am131-1795198142.html
https://vr.justeasy.cn/view/187709p51968w173-1795198185.html
https://vr.justeasy.cn/view/413u7956129e8216-1795198227.html
https://vr.justeasy.cn/view/l1wn779051984258-1795198269.html
https://vr.justeasy.cn/view/10z7951997c83i01-1795198312.html
https://vr.justeasy.cn/view/179151zi31998343-1795198355.html
https://vr.justeasy.cn/view/179d25vg19a838w6-1795198397.html
https://vr.justeasy.cn/view/m1lwrp7951d98428-1795198440.html
https://vr.justeasy.cn/view/y17951b9w2u84c70-1795198482.html
https://vr.justeasy.cn/view/1679c51f9o835v13-1795198524.html
https://vr.justeasy.cn/view/v17bim92519x8555-1795198567.html
https://vr.justeasy.cn/view/11779995198al598-1795198609.html
https://vr.justeasy.cn/view/17i9k5n1j9px8641-1795198652.html
https://vr.justeasy.cn/view/r17395ow1mv98683-1795198695.html
https://vr.justeasy.cn/view/127951y98pg7f296-1795198737.html
https://vr.justeasy.cn/view/81a79p519wt87688-1795198779.html
https://vr.justeasy.cn/view/1795c1898s871l70-1795198822.html
https://vr.justeasy.cn/view/61795uk1988pnd53-1795198864.html
https://vr.justeasy.cn/view/r1739mwf519u8896-1795198907.html
https://vr.justeasy.cn/view/179519e86re93p48-1795198949.html
https://vr.justeasy.cn/view/z179519d895582q0-1795198991.html
https://vr.justeasy.cn/view/1hz795oi199m0f22-1795199036.html
https://vr.justeasy.cn/view/1xf7951u9o9t0637-1795199079.html
https://vr.justeasy.cn/view/h17k3g951399m110-1795199122.html
https://vr.justeasy.cn/view/yb117795199ua153-1795199164.html
https://vr.justeasy.cn/view/81795y1sk9919v45-1795199206.html
https://vr.justeasy.cn/view/s17qyc951992l3q6-1795199248.html
https://vr.justeasy.cn/view/1ao779510em99279-1795199290.html
https://vr.justeasy.cn/view/e15t457951x99321-1795199332.html
https://vr.justeasy.cn/view/gh179z5k1u9k9363-1795199375.html
https://vr.justeasy.cn/view/0y1kd79514r99759-1795199770.html
https://vr.justeasy.cn/view/o11f795119978s01-1795199812.html
https://vr.justeasy.cn/view/1b7a951998zpk4w3-1795199854.html
https://vr.justeasy.cn/view/11795s19c9f8u816-1795199897.html
https://vr.justeasy.cn/view/1i79ke51t99892q8-1795199940.html
https://vr.justeasy.cn/view/1795199c8au29741-1795199982.html
https://vr.justeasy.cn/view/17j952yv0001t7s3-1795200025.html
https://vr.justeasy.cn/view/1r79ar5h20q0b056-1795200068.html
https://vr.justeasy.cn/view/1qpz79ix5u200099-1795200110.html
https://vr.justeasy.cn/view/q1379yb5200h1461-1795200152.html
https://vr.justeasy.cn/view/17u9c52n80gl0183-1795200195.html
https://vr.justeasy.cn/view/1y79w54200p22uw6-1795200238.html
https://vr.justeasy.cn/view/51795jh200bv26m9-1795200280.html
https://vr.justeasy.cn/view/117j953z52000311-1795200323.html
https://vr.justeasy.cn/view/179520fn503135t4-1795200365.html
https://vr.justeasy.cn/view/1795f52f0039c0e6-1795200408.html
https://vr.justeasy.cn/view/1td795vyii200439-1795200450.html
https://vr.justeasy.cn/view/17i9o5qmi2004c81-1795200493.html
https://vr.justeasy.cn/view/179785w2b00fv524-1795200535.html
https://vr.justeasy.cn/view/1k07j9c520tp0566-1795200578.html
https://vr.justeasy.cn/view/17951e821006t0a9-1795200620.html
https://vr.justeasy.cn/view/v179s5j2y0066x52-1795200663.html
https://vr.justeasy.cn/view/j1bo795203gl0694-1795200706.html
https://vr.justeasy.cn/view/103x79l520d07d37-1795200748.html
https://vr.justeasy.cn/view/9179cd52tg007r79-1795200790.html
https://vr.justeasy.cn/view/1h7j9y8at5200822-1795200833.html
https://vr.justeasy.cn/view/17q97q5op2f00864-1795200875.html
https://vr.justeasy.cn/view/17e8v95820039906-1795200918.html
https://vr.justeasy.cn/view/17975up20049fs50-1795200961.html
https://vr.justeasy.cn/view/17es952001f959b2-1795201004.html
https://vr.justeasy.cn/view/1cyt798552010335-1795201046.html
https://vr.justeasy.cn/view/179520l17zl07jh8-1795201089.html
https://vr.justeasy.cn/view/le117z95201128x0-1795201132.html
https://vr.justeasy.cn/view/bd17195201d81b64-1795201176.html
https://vr.justeasy.cn/view/17w952012k5n9708-1795201220.html
https://vr.justeasy.cn/view/17xw95620k125dz1-1795201263.html
https://vr.justeasy.cn/view/o1795206pu1ac295-1795201307.html
https://vr.justeasy.cn/view/41xro7954g201338-1795201349.html
https://vr.justeasy.cn/view/17f9r53042f01380-1795201391.html
https://vr.justeasy.cn/view/n17958eh2y014g23-1795201434.html
https://vr.justeasy.cn/view/17l9p5v2cs014u64-1795201476.html
https://vr.justeasy.cn/view/17955e12f01i5j06-1795201518.html
https://vr.justeasy.cn/view/w8p17bp9520155x0-1795201561.html
https://vr.justeasy.cn/view/1ebe7952v015g9g3-1795201605.html
https://vr.justeasy.cn/view/1r795m2zt016km37-1795201648.html
https://vr.justeasy.cn/view/1795270w1yda6m80-1795201691.html
https://vr.justeasy.cn/view/179c5207s175mv24-1795201735.html
https://vr.justeasy.cn/view/f179d4u520e17f67-1795201779.html
https://vr.justeasy.cn/view/179dym5tia201811-1795201822.html
https://vr.justeasy.cn/view/f17991l52201z854-1795201866.html
https://vr.justeasy.cn/view/17952p0o18qunl98-1795201912.html
https://vr.justeasy.cn/view/uj179qyh52019544-1795201956.html
https://vr.justeasy.cn/view/r179t5o20t1p9847-1795201999.html
https://vr.justeasy.cn/view/1fe5j7s9520k2031-1795202043.html
https://vr.justeasy.cn/view/170952202do08t74-1795202085.html
https://vr.justeasy.cn/view/1t7c95202110fgg6-1795202128.html
https://vr.justeasy.cn/view/tg51d79f52n02159-1795202171.html
https://vr.justeasy.cn/view/u17th9b652022302-1795202213.html
https://vr.justeasy.cn/view/1upw7n952l0237e5-1795202387.html
https://vr.justeasy.cn/view/0179vw520v524g18-1795202429.html
https://vr.justeasy.cn/view/8147r9e5l2029460-1795202472.html
https://vr.justeasy.cn/view/r13f79d52g025e02-1795202514.html
https://vr.justeasy.cn/view/17p935r2v0254io5-1795202556.html
https://vr.justeasy.cn/view/ri17952g0c2598s7-1795202599.html
https://vr.justeasy.cn/view/1p7952bz00w26i30-1795202641.html
https://vr.justeasy.cn/view/1579520y2w96sk72-1795202684.html
https://vr.justeasy.cn/view/179l52m02gs7m915-1795202726.html
https://vr.justeasy.cn/view/i31p79s5z2027257-1795202768.html
https://vr.justeasy.cn/view/1d7d9t952u027g99-1795202810.html
https://vr.justeasy.cn/view/1m7t905202784wq1-1795202853.html
https://vr.justeasy.cn/view/10795x20n42ow884-1795202895.html
https://vr.justeasy.cn/view/179ftu5nwj202926-1795202937.html
https://vr.justeasy.cn/view/1pi79520y8g2k968-1795202980.html
https://vr.justeasy.cn/view/svs1795828n03010-1795203021.html
https://vr.justeasy.cn/view/s91795rw20303x52-1795203064.html
https://vr.justeasy.cn/view/107955203y0ji925-1795203106.html
https://vr.justeasy.cn/view/178s9l52h0314437-1795203148.html
https://vr.justeasy.cn/view/1q79562qt0g31c79-1795203191.html
https://vr.justeasy.cn/view/187gcev952013222-1795203233.html
https://vr.justeasy.cn/view/z43179542mp03264-1795203275.html
https://vr.justeasy.cn/view/1a73952byg03p306-1795203318.html
https://vr.justeasy.cn/view/l517gy952b703349-1795203360.html
https://vr.justeasy.cn/view/179er520sf3r3991-1795203402.html
https://vr.justeasy.cn/view/175952d5034dp303-1795203445.html
https://vr.justeasy.cn/view/107p952627034z76-1795203487.html
https://vr.justeasy.cn/view/198bm794520k3518-1795203529.html
https://vr.justeasy.cn/view/177d9952a0o35e60-1795203571.html
https://vr.justeasy.cn/view/1ky7952dr03260t2-1795203613.html
https://vr.justeasy.cn/view/71h7935z2r034644-1795203655.html
https://vr.justeasy.cn/view/17g195k203b1r686-1795203698.html
https://vr.justeasy.cn/view/179oqt520372go19-1795203740.html
https://vr.justeasy.cn/view/1795z29038bu77a1-1795203783.html
https://vr.justeasy.cn/view/es107y91520s3814-1795203825.html
https://vr.justeasy.cn/view/17w95kb203q4v856-1795203867.html
https://vr.justeasy.cn/view/18789q452038jp98-1795203910.html
https://vr.justeasy.cn/view/17f9520vf39d4re1-1795203953.html
https://vr.justeasy.cn/view/1rod7mw952036983-1795203995.html
https://vr.justeasy.cn/view/1x71i952c0240q26-1795204037.html
https://vr.justeasy.cn/view/1f79520f490f6vd8-1795204079.html
https://vr.justeasy.cn/view/1c7wph9520411np0-1795204122.html
https://vr.justeasy.cn/view/481795204k1fh503-1795204164.html
https://vr.justeasy.cn/view/17952j041zznc194-1795204206.html
https://vr.justeasy.cn/view/75147g9520hn4237-1795204248.html
https://vr.justeasy.cn/view/13r7v9x52c0y4279-1795204291.html
https://vr.justeasy.cn/view/147933po45204321-1795204333.html
https://vr.justeasy.cn/view/176925z27043cw63-1795204374.html
https://vr.justeasy.cn/view/17f95wi204471065-1795204416.html
https://vr.justeasy.cn/view/17mx9f5i20444pj7-1795204459.html
https://vr.justeasy.cn/view/d1j795e204483u29-1795204503.html
https://vr.justeasy.cn/view/y17795204vk46535-1795204546.html
https://vr.justeasy.cn/view/m1795x2t0a49c577-1795204589.html
https://vr.justeasy.cn/view/1799z52z0nv46619-1795204630.html
https://vr.justeasy.cn/view/w1y7e95lhr204661-1795204673.html
https://vr.justeasy.cn/view/p1795p2qm0n470q3-1795204715.html
https://vr.justeasy.cn/view/167952i350j47446-1795204757.html
https://vr.justeasy.cn/view/176d9ku5k204w789-1795204800.html
https://vr.justeasy.cn/view/17u95a21dtz04831-1795204842.html
