# 腾讯小龙虾部署实战：接入大模型，让 AI 自动处理文档任务.md
## 前言
很多人初次上手腾讯小龙虾（OpenClaw/QClaw），只把它当成普通对话AI，忽略了它最核心的价值：**把大模型的文本理解能力，和本地/云端文件操作能力结合，全自动完成文档流水线任务**。

普通Chat类大模型只能上传文件进行问答；而小龙虾作为MCP协议驱动的AI智能体，在接入大模型之后，可以直接读取本地Word、PDF、Excel、Markdown，自动拆分文档、提取摘要、批量整理表格、生成报告、重命名归档，甚至跨文档做对比汇总。

本文为完整实战教程，以最常用的QClaw本地版为主，同时补充Lighthouse云端OpenClaw配置方案。从部署环境准备、大模型API接入、文档MCP技能安装、任务实战演示，到排错、安全风控，完整落地一套自动化文档处理流程。

## 一、原理简述：小龙虾+大模型，文档自动化是怎么工作的
小龙虾OpenClaw分为两层：**大模型大脑 + MCP工具执行层**。
1. **大模型（大脑）**：负责理解你的自然语言指令、阅读文档内容、做总结、提取数据、生成文本，支持混元、GLM、DeepSeek等所有兼容OpenAI协议的模型；
2. **MCP技能（手脚）**：文档处理依赖MCP服务，提供文件读取、PDF解析、Office读写、文件夹遍历、批量导出、重命名等操作系统能力；
3. 工作链路：你下发自然语言指令 → 大模型理解任务，规划执行步骤 → 调用文档MCP工具读取本机文件 → 大模型解析文档内容、执行文本处理 → 调用MCP保存结果、归档、生成新文档。

> 重点区别：普通网页AI，文件要上传到远端；QClaw本地部署模式，文档全程在你的电脑本地读写，文件原始数据不对外传输，适合处理合同、报表、内部资料。

## 二、前期环境准备
### 2.1 部署版本选择（文档场景选型）
- **QClaw本地版（推荐个人文档自动化）**：Windows/macOS一键安装，可以直接读取本机硬盘文档，手机微信远程下发指令。缺点：电脑关机则任务中断，适合按需手动触发文档任务。
- **Lighthouse云端OpenClaw**：腾讯云轻量服务器模板部署，7×24小时在线。无法直接读取本地PC文件，文档需要上传到云服务器对象存储，适合定时批量文档处理、自动报表生成。
> 本次实战以QClaw本地部署为例。

### 2.2 准备材料清单
1. QClaw安装包，完成基础安装，扫码绑定微信IM通道；
2. 大模型API密钥，支持OpenAI兼容协议（腾讯混元TokenHub、GLM、DeepSeek等均可）；
3. 文档MCP技能包（文档解析、Office工具、PDF读取，ClawHub社区一键安装）；
4. 测试文件：PDF报告、Word文档、Excel数据表，放在电脑一个单独文件夹（建议路径**不要带中文空格**，减少报错）。

## 三、实战第一步：部署并接入大模型（核心配置）

打开QClaw客户端，在对话窗口下方找到模型下拉选择框：
1. 点击下拉菜单，选择**自定义大模型**，进入模型API配置面板；
2. 选择【模型API】标签，点击「+自定义配置」；
3. 填写表单（所有支持OpenAI兼容协议的模型都按这个模板填写）：
    - API协议类型：**OpenAI兼容协议**（固定）
    - 接口地址(Base URL)：模型厂商提供的API地址，如腾讯TokenHub：`[https://tokenhub.tencentmaas.com/plan/v3](https://tokenhub.tencentmaas.com/plan/v3)`
    - API Key：从模型服务商控制台复制密钥，妥善保管，不要泄露
    - 模型名称：填写对应模型Model ID，例如hy3、glm-4等
4. 保存配置，回到下拉列表，切换到刚刚新增的自定义模型；
5. 验证连通性：发送一句话测试：`测试连接，请回复“模型连接成功”`。

> 云端Lighthouse OpenClaw操作类似：登录OpenClaw后台，进入Models模型管理页面，新增自定义模型，填入同样API参数，保存并启用。

### 模型选型建议（文档任务专用）
文档处理优先选**长上下文窗口大模型**，适合PDF、长报告、多文档阅读：
- 腾讯混元长文本：合同、政策、长文档精读、提取条款；
- GLM-4 / DeepSeek：多文档对比、数据分析、复杂表格提取。

> 小坑提醒：短上下文模型处理长PDF会截断内容，出现幻觉、漏读段落，做文档任务尽量选择128k及以上上下文窗口。

## 四、实战第二步：安装文档处理MCP技能包
大模型接入完成后，只是拥有“思考能力”，还没有读写本地文件的权限，必须启用文档相关MCP技能。

### 方式1：ClawHub可视化一键安装（推荐新手）
1. 打开QClaw技能市场ClawHub；
2. 搜索并启用这几个核心文档MCP：
    - `file-manager` 文件管理器：遍历文件夹、读取文本、新建/重命名文件；
    - `pdf-reader` PDF解析：读取PDF文字内容，支持扫描件以外的电子PDF；
    - `office-parser` Office文档解析：读取Word、Excel内容，提取表格；
    - `doc-writer` 文档写入：让AI生成新Word/Markdown/CSV报告并保存到本地；
3. 启用技能后，重启QClaw的MCP服务，执行`MCP重载`；
4. 校验技能：发送指令：`列出D:\doc_test文件夹下面所有文件`，AI能返回文件列表，说明文件工具正常加载。

### 方式2：手动配置openclaw.json（进阶开发者）
硬核玩家可直接编辑配置文件，添加MCP服务配置，配置完成执行`openclaw mcp reload`重载，适合自定义本地脚本、自建文档解析服务。

> 安全提醒：文件MCP拥有读写本机文件能力。建议开启**高危操作人工确认**，删除、覆盖文件这类动作，小龙虾会先询问你确认，防止误删重要文档。

## 五、实战第三步：文档自动化任务实操演示
环境、模型、技能全部就绪，我们直接执行几类高频文档任务。你可以直接复制指令发给小龙虾（微信远程或者客户端对话都可以）。

### 案例1：单篇PDF报告摘要提取
> 指令：
> 读取D:\doc_test\季度报告.pdf，提炼300字核心摘要，提取报告里3个关键数据指标，将结果保存为summary.md，放在同一个目录下。

执行流程：
1. MCP文件工具定位并读取PDF；
2. 大模型解析全文，提炼摘要和指标；
3. MCP写入本地新建Markdown文档保存结果；
4. 完成后自动回复：任务已完成，文件路径xxx。

### 案例2：批量多文档汇总（多Word+Excel）
> 指令：
> 遍历D:\doc_test目录，读取全部Word和Excel，汇总所有文档里的项目预算数据，生成汇总表格，输出为budget_summary.csv。

适合批量合同、多份报表合并，一次性汇总数十份文档的数据。

### 案例3：文档校对与问题标注
> 指令：
> 读取合同.docx，检查合同里风险条款，标记潜在漏洞，生成一份修改建议清单，另存为合同修订建议.docx。

### 案例4：定时云端文档任务（Lighthouse版本）
云端部署OpenClaw可以设置定时任务：每天早上8点，读取对象存储内昨日上传的报表，自动汇总并生成日报，保存日志。

## 六、常见踩坑排查（文档任务高频问题）
1. **AI提示找不到文件**
    - 原因：文件路径包含中文、空格、特殊符号；或者没有给MCP授予该文件夹访问权限。
    - 解决：文件夹重命名为英文，在MCP配置设置允许访问目录白名单。
2. **PDF读取空白，提取不出文字**
    - 原因：图片扫描版PDF，纯图片没有可复制文本；
    - 解决：搭配OCR MCP技能，增加图片文字识别能力。
3. **大模型返回内容截断，文档读取不全**
    - 原因：模型上下文窗口不足；或者MCP单次读取文件大小限制。
    - 解决：换长上下文模型；或者配置MCP分片读取长文档。
4. **API调用失败、报401鉴权错误**
    - 检查API Key是否复制多余空格，确认接口地址、模型ID填写无误，检查API密钥余额。
5. **AI直接覆盖原有文档**
    - 解决：在Agent设置开启「高危操作人工确认」，修改、覆盖、删除文件必须人工确认后执行。

## 七、安全与最佳实践
1. **权限最小化原则**：不要给MCP开放全盘访问，只允许访问存放文档的指定文件夹，不开放系统目录；
2. **敏感文档**：涉密、隐私文档优先本地QClaw，避免上传云端Lighthouse；
3. **模型幻觉防控**：自动生成文档后，重要内容务必人工复核，AI提取数据可能出错；
4. **密钥管理**：API Key不要分享、不要截图发聊天，一旦泄露会被盗刷Token；
5. **文件备份**：自动化批量处理文档前，先备份源文件，防止意外覆盖。

## 八、进阶玩法扩展
1. 搭配知识库MCP：处理完文档自动存入向量知识库，后续可基于这批文档问答；
2. 增加邮件MCP：文档处理完成，自动把生成的报告作为附件发送邮件；
3. 自定义提示词模板：预设文档提取模板，每次执行任务直接调用模板，减少重复写指令；
4. 多Agent分工：一个Agent负责读取文档，一个负责数据分析，一个负责生成报告。

## 总结
腾讯小龙虾做文档自动化的核心逻辑，就是**大模型负责“看懂文档、思考、写作”，MCP工具负责“读写本地文件”**。
整套落地流程分为四步：部署小龙虾 → 接入兼容OpenAI协议的大模型 → 安装文档类MCP技能 → 下发自然语言指令自动执行文档任务。

个人办公、批量报表、合同审阅、文献整理场景，用本地QClaw部署，文档保存在本机，隐私可控；需要无人值守、定时自动跑报表，则用Lighthouse云端OpenClaw。
只要掌握这套方案，就可以把大量重复的文档阅读、汇总、整理工作交给AI智能体自动完成。



https://my.oschina.net/u/9769155/blog/19802373
https://my.oschina.net/u/9769155/blog/19802375
https://my.oschina.net/u/9769155/blog/19802376
https://my.oschina.net/u/9769155/blog/19802377
https://my.oschina.net/u/9769156/blog/19802378
https://my.oschina.net/u/9769156/blog/19802379
https://my.oschina.net/u/9769156/blog/19802380
https://my.oschina.net/u/9769156/blog/19802381
https://my.oschina.net/u/9769157/blog/19802382
https://my.oschina.net/u/9769157/blog/19802383
https://my.oschina.net/u/9769157/blog/19802384
https://my.oschina.net/u/9769157/blog/19802385
https://my.oschina.net/u/9769158/blog/19802386
https://my.oschina.net/u/9769158/blog/19802387
https://my.oschina.net/u/9769158/blog/19802388
https://my.oschina.net/u/9769158/blog/19802389
https://my.oschina.net/u/9769159/blog/19802391
https://my.oschina.net/u/9769159/blog/19802392
https://my.oschina.net/u/9769159/blog/19802393
https://my.oschina.net/u/9769159/blog/19802394
https://my.oschina.net/u/9769160/blog/19802395
https://my.oschina.net/u/9769160/blog/19802396
https://my.oschina.net/u/9769160/blog/19802397
https://my.oschina.net/u/9769160/blog/19802398
https://my.oschina.net/u/9769161/blog/19802399
https://my.oschina.net/u/9769161/blog/19802400
https://my.oschina.net/u/9769161/blog/19802401
https://my.oschina.net/u/9769161/blog/19802402
https://my.oschina.net/u/9769162/blog/19802403
https://my.oschina.net/u/9769162/blog/19802404
https://my.oschina.net/u/9769162/blog/19802405
https://my.oschina.net/u/9769162/blog/19802406
https://my.oschina.net/u/9769163/blog/19802407
https://my.oschina.net/u/9769163/blog/19802408
https://my.oschina.net/u/9769163/blog/19802409
https://my.oschina.net/u/9769163/blog/19802410
https://my.oschina.net/u/9769164/blog/19802411
https://my.oschina.net/u/9769164/blog/19802412
https://my.oschina.net/u/9769164/blog/19802413
https://my.oschina.net/u/9769164/blog/19802414
https://my.oschina.net/u/9769165/blog/19802415
https://my.oschina.net/u/9769165/blog/19802416
https://my.oschina.net/u/9769165/blog/19802417
https://my.oschina.net/u/9769165/blog/19802418
https://my.oschina.net/u/9767665/blog/19802330
https://my.oschina.net/u/9767665/blog/19802348
https://my.oschina.net/u/9767665/blog/19802349
https://my.oschina.net/u/9769149/blog/19802351	
https://my.oschina.net/u/9769149/blog/19802350
https://my.oschina.net/u/9769149/blog/19802352
https://my.oschina.net/u/9769149/blog/19802353
https://my.oschina.net/u/9769150/blog/19802356	
https://my.oschina.net/u/9769150/blog/19802355
https://my.oschina.net/u/9769150/blog/19802357
https://my.oschina.net/u/9769150/blog/19802358
https://my.oschina.net/u/9769151/blog/19802360	
https://my.oschina.net/u/9769151/blog/19802359
https://my.oschina.net/u/9769151/blog/19802361
https://my.oschina.net/u/9769151/blog/19802362
https://my.oschina.net/u/9769152/blog/19802364
https://my.oschina.net/u/9769152/blog/19802365
https://my.oschina.net/u/9769152/blog/19802366
https://my.oschina.net/u/9769153/blog/19802369	
https://my.oschina.net/u/9769153/blog/19802368	
https://my.oschina.net/u/9769152/blog/19802367
https://my.oschina.net/u/9769153/blog/19802370
https://my.oschina.net/u/9769153/blog/19802371






