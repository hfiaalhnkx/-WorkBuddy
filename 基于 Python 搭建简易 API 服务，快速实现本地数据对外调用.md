# 基于 Python 搭建简易 API 服务，快速实现本地数据对外调用.md

很多本地业务场景中，我们手里有存储在本地的文本、CSV、数据库等数据，其他程序、前端页面、第三方工具想要读取这些数据，传统方案要么复制文件，要么手动导出，十分麻烦。利用 Python 可以快速搭建轻量 API 服务，把本地数据封装成 HTTP 接口，通过 GET/POST 请求就能完成数据读取与交互，无需部署复杂的 Nginx、K8s 集群，开发调试几分钟就能跑通。

本文选用两个主流轻量框架：**Flask**（简单上手，适合快速原型），以及 **FastAPI**（高性能、自动接口文档，推荐新项目），两套示例都可直接复制运行，实现本地数据对外调用。

## 一、技术选型说明

1. Flask：老牌轻量 Web 框架，代码简短，学习门槛低，适合小型内部接口、临时调试服务。
2. FastAPI：现代高性能框架，基于 Starlette，自带 OpenAPI 接口文档，支持类型提示，开发效率更高，适合长期使用的内部API。

> 
> 注意：本文搭建的简易API仅用于**内网本地调试、内部业务调用**，不建议直接暴露到公网，缺少鉴权、限流、HTTPS等安全能力，公网使用需要额外加固。

## 二、方案1：Flask 快速搭建本地数据API

### 1. 安装依赖

```
pip install flask
```

### 2. 完整示例代码

```
from flask import Flask, jsonify, request
import json

app = Flask(__name__)

# 模拟本地数据：实际可替换为读取csv、sqlite、本地json文件
local_data = {
    "dept_list": [
        {"id": 1, "name": "研发部", "person_count": 28},
        {"id": 2, "name": "产品部", "person_count": 12},
        {"id": 3, "name": "运营部", "person_count": 16}
    ],
    "update_time": "2026-11-27"
}

# GET接口：获取全部本地数据
@app.route("/api/data/all", methods=["GET"])
def get_all_data():
    return jsonify(code=200, msg="success", data=local_data)

# GET接口：按ID查询部门信息
@app.route("/api/data/dept/<int:dept_id>", methods=["GET"])
def get_dept(dept_id):
    res = next((item for item in local_data["dept_list"] if item["id"] == dept_id), None)
    if res:
        return jsonify(code=200, msg="success", data=res)
    else:
        return jsonify(code=404, msg="未找到对应部门数据"), 404

# POST接口：接收外部提交数据
@app.route("/api/data/add", methods=["POST"])
def add_data():
    req_data = request.get_json()
    if not req_data:
        return jsonify(code=400, msg="请求体JSON格式错误"), 400
    local_data["dept_list"].append(req_data)
    return jsonify(code=200, msg="新增成功", data=req_data)

if __name__ == "__main__":
    # host="0.0.0.0" 允许局域网其他设备访问；host="127.0.0.1"仅本机访问
    app.run(host="0.0.0.0", port=5000, debug=True)
```

### 3. 启动与调用测试

1. 保存为 `flask_api.py`，执行：`python flask_api.py`
2. 服务启动后访问地址：

- 获取全部数据：`http://127.0.0.1:5000/api/data/all`
- 查询id=1部门：`http://127.0.0.1:5000/api/data/dept/1`

3. POST请求可以使用 Postman / curl 提交JSON新增数据。

> 
> debug=True 仅开发调试使用，正式环境务必关闭，防止代码泄露与安全风险。

## 三、方案2：FastAPI 高性能简易API（推荐）

FastAPI最大优势：自动生成交互式接口文档，支持数据校验，异步性能更好。

### 1. 安装依赖

```
pip install fastapi uvicorn
```

### 2. 完整示例代码

```
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI(title="本地数据API服务")

# 本地模拟数据
local_data = {
    "dept_list": [
        {"id": 1, "name": "研发部", "person_count": 28},
        {"id": 2, "name": "产品部", "person_count": 12},
        {"id": 3, "name": "运营部", "person_count": 16}
    ],
    "update_time": "2026-11-27"
}

# 定义POST请求数据模型，自动校验字段类型
class DeptItem(BaseModel):
    id: int
    name: str
    person_count: int

# GET 获取全部数据
@app.get("/api/data/all")
def get_all():
    return {"code":200, "msg":"success", "data": local_data}

# GET 按ID查询
@app.get("/api/data/dept/{dept_id}")
def get_dept(dept_id:int):
    res = next((i for i in local_data["dept_list"] if i["id"] == dept_id), None)
    if res:
        return {"code":200, "msg":"success", "data": res}
    return {"code":404, "msg":"未找到部门"}

# POST新增数据
@app.post("/api/data/add")
def add_dept(item: DeptItem):
    local_data["dept_list"].append(item.model_dump())
    return {"code":200, "msg":"新增成功", "data": item}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run("main:app", host="0.0.0.0", port=8000, reload=True)
```

### 3. 启动与接口文档

1. 文件命名 `main.py`，运行：`python main.py`
2. 接口地址：`http://127.0.0.1:8000/api/data/all`
3. **自动交互式文档**

- Swagger UI：`http://127.0.0.1:8000/docs` 可视化页面，直接在网页调试GET/POST请求
- ReDoc文档：`http://127.0.0.1:8000/redoc`，用于导出离线接口说明文档

## 四、对接真实本地数据（替换模拟数据）

上面示例用内存模拟数据，下面改成读取本地 CSV / JSON 文件作为数据源。

### 示例：读取本地JSON文件作为API数据源

```
import json
from pathlib import Path

def load_local_json():
    file_path = Path("./local_data.json")
    with open(file_path, "r", encoding="utf-8") as f:
        return json.load(f)

# 在接口函数中调用
local_data = load_local_json()
```

### 示例：读取本地CSV文件（pandas）

```
pip install pandas
```

```
import pandas as pd
def load_csv_data():
    df = pd.read_csv("./business.csv")
    return df.to_dict("records")
```

> 
> 注意：内存数据会在服务重启后丢失。如果需要持久化，写入本地文件或SQLite数据库。

## 五、局域网其他设备访问API

服务启动参数 `host="0.0.0.0"` 代表监听本机所有网卡地址。
同一局域网内其他电脑/手机，可以用**本机内网IP + 端口**访问。
例：本机IP是 `192.168.1.100`，FastAPI服务端口8000，则访问地址：
`http://192.168.1.100:8000/api/data/all`

排查访问失败要点：

1. 关闭电脑防火墙或者放行端口；
2. 确认两台设备在同一个WiFi/局域网；
3. 不要用127.0.0.1，这个地址只能本机访问。

## 六、生产与安全注意事项

1. **公网暴露风险**：简易服务没有身份认证，任何人拿到地址都能调用，禁止直接映射公网。公网部署需要增加token鉴权、IP白名单、HTTPS。
2. 调试模式：Flask `debug=True` / FastAPI `reload=True` 仅开发使用，正式环境关闭。
3. 并发性能：内置开发服务器性能弱，高并发场景需要使用gunicorn+uvicorn组合部署。
4. 文件读取异常：读取本地文件时增加try except捕获文件不存在、编码错误。

```
try:
    with open(file_path, "r", encoding="utf-8") as f:
        return json.load(f)
except Exception as e:
    print(f"读取本地数据失败：{e}")
    return []
```

## 七、拓展实战方向

- 增加接口Token鉴权，只有携带合法token才能获取数据；
- 接入SQLite本地数据库，实现数据持久化增删改查；
- 增加分页接口，本地数据量大时分批返回；
- 结合定时脚本更新本地数据源，API自动返回最新业务数据；
- 配合之前Python文件自动化脚本，实现：文件自动汇总 → 写入本地文件 → API对外提供查询。

## 结语

使用Python搭建简易API，核心价值就是打通本地数据和外部程序之间的数据通道。Flask适合快速验证原型，FastAPI适合长期维护的内部接口。不需要复杂的中间件，十几行代码就能把本地文件、表格数据封装成HTTP接口，让前端、自动化脚本、其他业务系统直接调用本地数据，极大简化多系统之间的数据对接工作。




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
