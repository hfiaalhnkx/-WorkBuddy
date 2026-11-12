# 腾讯小龙虾 OpenClaw 部署指南：Lighthouse 一键安装，打通 IM 自动化.md
> 导读：AI Agent 落地最大的难点，不是提示词，而是部署与交互。很多技术爱好者尝试自建智能体，卡在环境安装、容器配置、消息通道对接等环节。OpenClaw（圈内昵称小龙虾）作为开源AI智能体网关，依托腾讯云Lighthouse轻量应用服务器预制模板，实现**一键安装**，无需手动编译代码、处理Docker镜像。部署完成后，接入企业微信、微信、飞书等IM通道，直接在聊天窗口下发自然语言指令，让AI自动完成多步骤任务，搭建一套7×24小时在线的自动化智能体。本文为完整实操指南，覆盖服务器选购、初始化、模型配置、IM通道对接、技能插件安装、自动化任务落地、故障排查与运维优化，内容详实，新手与开发者均可参考。

## 一、OpenClaw小龙虾与Lighthouse方案概述
### 1. 什么是OpenClaw
OpenClaw是开源自托管AI智能体网关，它并非大模型本身，而是负责**任务拆解、工具调度、记忆管理、消息路由**的中间层。它接收自然语言指令，将复杂需求拆分为多步执行计划，调用各类技能插件完成网页抓取、文档解析、定时任务、文件读写，再将结果推送至IM聊天窗口。

简单分工：
- 大模型：理解自然语言、规划任务逻辑、生成文本；
- OpenClaw小龙虾：负责任务调度、插件调用、长期记忆、IM消息收发；
- 技能插件：提供各类工具能力，按需开启，模块化扩展。

### 2. 为什么选择腾讯云Lighthouse一键模板部署
传统源码或Docker部署OpenClaw，需要手动安装运行环境、拉取镜像、处理端口映射、解决依赖冲突，网络波动还会导致镜像拉取失败，对新手门槛很高。

腾讯云Lighthouse提供**预制OpenClaw应用模板**，底层运行环境、数据库、网关服务全部预装好，只需要创建实例即可，网页可视化操作，几乎不用命令行，优势如下：
1. **部署速度快**：实例创建3～5分钟自动初始化，开箱即用；
2. **运维简单**：Lighthouse自带控制台、防火墙、快照备份，可视化管理；
3. **国内低延迟**：国内节点，IM消息推送响应快；
4. **成本可控**：轻量实例资费低廉，适合个人、小团队长期7×24小时运行；
5. **原生支持多IM通道**：企微、微信、QQ、飞书通道插件可直接在后台安装配置。

> 适用场景：个人自动化助理、私有知识库问答、资讯定时汇总、网页监控、轻量团队数字员工。

## 二、部署前置准备
### 1. 资源清单
1. 腾讯云账号，完成实名认证；
2. 大模型API密钥：作为智能体推理能力来源，兼容OpenAI协议接口；
3. IM账号：优先企业微信（稳定性最佳，支持群聊推送）；个人体验可选微信；
4. 浏览器，可正常访问腾讯云控制台。

### 2. 服务器配置选型
- 个人体验、轻量问答、少量网页抓取：**2核2G**，系统盘40G；
- RAG知识库、大量文档上传、高频定时任务：**2核4G**；
- 多并发任务、团队多人使用：4核8G。

### 3. 端口规划（关键）
OpenClaw网关通信端口为`18789`，实例创建后，**务必在Lighthouse防火墙放行TCP 18789端口**，否则IM消息无法回调通信。

## 三、Lighthouse一键安装OpenClaw完整步骤
1. 登录腾讯云控制台，进入【轻量应用服务器Lighthouse】，点击【新建实例】。
2. **镜像选择：应用模板**，在AI应用列表选中`OpenClaw`预制镜像。⚠️ 不要选择空白Linux系统，空白系统需要手动部署环境。
3. 选择就近地域，设置实例登录密码，自定义实例名称，确认套餐配置，提交订单。
4. 等待实例状态变为**运行中**，一般3～5分钟，系统自动部署OpenClaw全套服务。
5. 进入实例详情页，切换【应用管理】标签，查看公网IP与OpenClaw后台访问地址：`http://公网IP:18789`。
6. 防火墙配置：切换到【防火墙】页面，添加规则，放行TCP 18789端口，来源选择0.0.0.0/0，保存。

> 注意：实例创建完成后，OpenClaw网关服务默认自动启动，无需手动执行启动命令。

## 四、OpenClaw后台初始化，接入大模型
访问`http://公网IP:18789`进入Web管理面板，完成模型配置，给小龙虾装上AI大脑。
1. 在后台导航栏打开【模型Models】配置页；
2. 填写模型接口地址、API Key，选择对应的模型名称；
3. 设置参数：上下文窗口、温度，个人场景默认值即可；
4. 点击保存并应用，在后台对话窗口发送测试消息，验证模型连通。

**常见连通失败排查**
- API Key复制时携带前后空格，需要清理；
- 接口地址填写错误；
- 模型调用额度耗尽；
- 服务器出口网络无法访问模型接口。

模型连通成功后，OpenClaw网关就具备基础对话能力，接下来接入IM通道，实现微信/企微远程操控。

## 五、IM通道配置，打通消息自动化入口
OpenClaw支持多IM通道，企业微信适合长期稳定自动化任务；微信适合个人私信交互；下面分别讲解配置流程。

### 方案A：企业微信通道（推荐）
企微通道支持私聊、群聊消息推送，回调机制稳定，适合定时任务结果推送。
1. 登录企业微信管理后台`work.weixin.qq.com`；
2. 左侧菜单【管理工具】→【智能机器人】，新建API模式机器人；
3. 获取Bot ID、Secret、Token、EncodingAESKey；Secret仅展示一次，务必复制保存；
4. 回调地址填写：`https://服务器公网IP:18789/wecom-app`，暂不要点保存；
5. 返回OpenClaw后台，打开【通道Channels】，选择企业微信；
6. 依次填入Bot ID、Secret、Token、EncodingAESKey，点击添加并应用，网关自动重启；
7. 返回企微后台保存机器人配置，获取机器人二维码；
8. 企微扫码添加机器人，即可在企微对话框下发指令，接收任务执行结果。

### 方案B：个人微信通道（插件模式）
适合个人快速体验，仅支持私信，不支持群聊，稳定性弱于企微。
1. 在OpenClaw后台ClawHub插件市场，安装微信通道插件`@tencent-weixin/openclaw-weixin`；
2. 通道管理页面选中微信，启动登录流程，页面生成登录二维码；
3. 使用微信扫码授权，等待通道连接成功；
4. 扫码成功后，直接私信小龙虾下达指令。

> 补充：还可以按需安装QQ、飞书通道插件，同一台OpenClaw实例支持同时开启多个IM通道。

## 六、安装技能插件，解锁自动化任务能力
基础对话只是起点，**技能插件是实现自动化的核心**，进入ClawHub技能市场按需安装。
1. **网页抓取插件**：访问网页，提取正文、自动生成资讯摘要；
2. **RAG私有知识库插件**：上传PDF、Markdown、TXT文档，建立私有知识库，支持文档问答；
3. **定时任务插件**：自然语言创建Cron周期性任务，到时间自动执行并推送结果；
4. **长期记忆插件**：跨会话保存你的需求偏好、历史任务，解决大模型短时记忆问题；
5. **文件处理插件**：解析表格、生成Markdown报告、导出文档；
6. **代码执行插件**：简单代码运行、数据计算（按需开启，注意权限安全）。

### 自动化任务实操示例（直接发送给小龙虾）
> 每周一早上9点，检索5篇AI Agent技术文章，提炼核心观点，整理成技术简报，推送到企微技术群。

智能体收到这条自然语言指令，会自动拆解任务：
1. 解析任务周期、检索目标、输出格式、推送渠道；
2. 到触发时间，自动调用网页抓取插件检索文章；
3. 调用大模型总结提炼要点；
4. 生成Markdown简报；
5. 通过企微通道推送至指定群聊，全程无人值守。

其他常用指令示例：
- 每天22点抓取行业新闻，汇总摘要私信发给我；
- 上传这份PDF文档，后续可以直接提问文档内容；
- 监控XX页面，页面内容发生变化时推送提醒。

## 七、全链路功能测试
全部配置完成，执行4轮测试，验证整套自动化链路可用：
1. **基础对话测试**：发送问候消息，确认正常回复；
2. **工具调用测试**：下发网页检索总结任务，验证联网与文本处理；
3. **IM推送测试**：让小龙虾主动推送一条测试消息到企微/微信；
4. **云端常驻测试**：关闭本地电脑，手机发送指令，验证云服务器持续在线执行任务。

全部测试通过，你的IM自动化AI智能体正式上线。

## 八、高频故障排查与性能优化
### 1. 机器人收不到消息
- 防火墙未放行18789端口；
- 回调地址IP、端口填写错误；
- Bot ID、Secret复制错误，存在多余空格；
- 网关服务异常，在后台重启网关。

### 2. 定时任务不执行
- 定时技能插件未启用；
- 服务器时间时区异常；
- 任务指令描述模糊，Agent无法正确解析；
- 查看后台运行日志，定位报错信息。

### 3. 服务器内存占用高
- 同时运行过多并发任务，减少并行任务数量；
- RAG知识库文档数量大，升级服务器内存；
- 定期清理历史对话缓存。

### 4. 模型响应延迟
- 更换国内低延迟模型接口；
- 检查服务器出口网络质量；
- 降低并发工具调用频率。

### 5. 记忆丢失、上下文混乱
安装长期记忆插件；定期做实例快照备份，防止配置丢失。

## 九、落地业务场景
✅ 个人资讯订阅：定时抓取行业技术资讯，自动汇总推送；
✅ 私有知识库：读书笔记、项目文档上传，文档数据保存在自己云服务器，不对外泄露；
✅ 自动化办公：周报、会议纪要整理，定时信息汇总，结果推送IM；
✅ 网页监控：监测技术站点、公告页面更新，变动主动推送提醒；
✅ 研发辅助：阅读技术文档、提取代码要点、整理开发笔记；
✅ 小团队轻量数字员工：内部资料问答，文档处理，群内信息汇总。

## 十、安全与合规注意事项
1. **权限最小化原则**：不要给OpenClaw开放服务器root高危权限，防止AI执行破坏性指令；
2. 服务器安全：定期修改实例登录密码，开启自动快照备份；
3. IM使用规范：微信、企微通道遵守平台规则，禁止营销、批量骚扰；
4. 爬虫限制：网页抓取仅限个人学习研究，禁止大规模违规采集网站数据；
5. 合规红线：不得利用智能体执行违法违规任务，遵守网络安全相关法律法规；
6. 数据隐私：涉及企业敏感资料，做好访问权限管控。

## 十一、进阶拓展方向
基础部署完成后，还可以持续扩展小龙虾能力边界：
1. 自定义技能插件开发，对接自研业务接口；
2. 多智能体路由，创建多个独立Agent，分工处理不同类型任务；
3. 接入对象存储，挂载大容量知识库文件；
4. 配置HTTPS，加密IM回调通信；
5. 接入数据库，持久化存储任务记录。

## 结语
AI Agent的核心价值，在于打通IM交互、自动执行任务，而不是单纯对话。腾讯云Lighthouse的OpenClaw预制模板，把最繁琐的环境部署环节一键完成，大幅降低自建智能体门槛。

只需要简单配置大模型、IM通道、技能插件，就能快速搭建属于自己的小龙虾AI智能体。手机微信、企微随时下发自然语言指令，云端7×24小时自动完成资料检索、文档处理、定时推送等重复工作，把AI真正融入日常办公与学习流程。




https://bbs.xiaopeng.com/article/3967488
https://bbs.xiaopeng.com/article/3967487
https://bbs.xiaopeng.com/article/3967485
https://bbs.xiaopeng.com/article/3967483
https://bbs.xiaopeng.com/article/3967482
https://bbs.xiaopeng.com/article/3967474
https://bbs.xiaopeng.com/article/3967473
https://bbs.xiaopeng.com/article/3967472
https://bbs.xiaopeng.com/article/3967469
https://bbs.xiaopeng.com/article/3967467
https://bbs.xiaopeng.com/article/3967360
https://bbs.xiaopeng.com/article/3967355
https://bbs.xiaopeng.com/article/3967351
https://bbs.xiaopeng.com/article/3967350
https://bbs.xiaopeng.com/article/3967346
https://bbs.xiaopeng.com/article/3968930
https://bbs.xiaopeng.com/article/3968929
https://bbs.xiaopeng.com/article/3968928
https://bbs.xiaopeng.com/article/3968927
https://bbs.xiaopeng.com/article/3968926
https://bbs.xiaopeng.com/article/3968925
https://bbs.xiaopeng.com/article/3968924
https://bbs.xiaopeng.com/article/3968922
https://bbs.xiaopeng.com/article/3968921
https://bbs.xiaopeng.com/article/3968920
https://bbs.xiaopeng.com/article/3968919
https://bbs.xiaopeng.com/article/3968918
https://bbs.xiaopeng.com/article/3968917
https://bbs.xiaopeng.com/article/3968916
https://bbs.xiaopeng.com/article/3968915
https://bbs.xiaopeng.com/article/3968867
https://bbs.xiaopeng.com/article/3968866
https://bbs.xiaopeng.com/article/3968864
https://bbs.xiaopeng.com/article/3968862
https://bbs.xiaopeng.com/article/3968860
https://bbs.xiaopeng.com/article/3968857
https://pastebin.com/qqu7w8Qm
https://pastebin.com/6ZXZjyxz
https://pastebin.com/89mM2EBR
https://pastebin.com/4ZUFqtnJ
https://pastebin.com/ypgh5StP
https://pastebin.com/30Jd06mb
https://pastebin.com/D3pL8cMx
https://pastebin.com/rt3ckL6z
https://pastebin.com/91zjwbj8
https://pastebin.com/Dyzvf4Ge
https://pastebin.com/ggH4ut6P
https://pastebin.com/ZYrqFHMT
https://pastebin.com/zn5c66tm
https://pastebin.com/mvz6h05h
https://pastebin.com/H8mLMXwM
https://pastebin.com/ffbNmg7t
https://pastebin.com/gn8hLzaS
https://pastebin.com/TB4UpKWU
https://pastebin.com/NEGm6gAf
https://pastebin.com/81xG7x3R
https://bbs.xiaopeng.com/article/3970773
https://bbs.xiaopeng.com/article/3970772
https://bbs.xiaopeng.com/article/3970770
https://bbs.xiaopeng.com/article/3970769
https://bbs.xiaopeng.com/article/3970768
https://bbs.xiaopeng.com/article/3970767
https://bbs.xiaopeng.com/article/3970766
https://bbs.xiaopeng.com/article/3970764
https://bbs.xiaopeng.com/article/3970762
https://bbs.xiaopeng.com/article/3970760
https://bbs.xiaopeng.com/article/3970759
https://bbs.xiaopeng.com/article/3970758
https://bbs.xiaopeng.com/article/3970757
https://bbs.xiaopeng.com/article/3970756
https://bbs.xiaopeng.com/article/3970755
https://bbs.xiaopeng.com/article/3970754
https://bbs.xiaopeng.com/article/3970753
https://bbs.xiaopeng.com/article/3970751
https://bbs.xiaopeng.com/article/3970750
https://bbs.xiaopeng.com/article/3970748
https://bbs.xiaopeng.com/article/3970746
https://bbs.xiaopeng.com/article/3970744
https://bbs.xiaopeng.com/article/3970743
https://bbs.xiaopeng.com/article/3970741
https://bbs.xiaopeng.com/article/3970739
https://bbs.xiaopeng.com/article/3970737
https://pastebin.com/me9VcSMf
https://pastebin.com/xBnh3Z1t
https://pastebin.com/aYa2a5vh
https://pastebin.com/TyfJRyZ1
https://pastebin.com/3qKACUGJ
https://pastebin.com/xkT69fDa
https://pastebin.com/szuC4j88
https://pastebin.com/PS0HdEtL
https://pastebin.com/kvBUjqT0
https://pastebin.com/rhQqN7v4
https://pastebin.com/pSH7fUxf
https://pastebin.com/YNAvfMHE
https://pastebin.com/LgQFyqGr
https://pastebin.com/eHWVUe1m
https://pastebin.com/1LiCyBkX
https://pastebin.com/EJf8PmRL
https://pastebin.com/HC3nRXXG
https://pastebin.com/NDd1MK8s
https://pastebin.com/ivTndYuc
https://pastebin.com/t6brzYDu


https://ds.163.com/feed/6af5fe56ca8643644ad5727f/
https://ds.163.com/feed/6af5fea4290a623aac43618f/
https://ds.163.com/feed/6af5fecbca8643644ad572ec/
https://ds.163.com/feed/6af5fee87e55247b7d5e4fb1/
https://ds.163.com/feed/6af5ff09a9b3f049ca2be6f8/
https://ds.163.com/feed/6af5ff1d99f09836c92c9bbf/
https://ds.163.com/feed/6af5ff2d9331b1110e942976/
https://ds.163.com/feed/6af5ff3adc81547966f0add6/
https://ds.163.com/feed/6af5ff599331b1110e9429a9/
https://ds.163.com/feed/6af60aef18a5e875e217895a/
https://ds.163.com/feed/6af60b27d9851f70eca2b219/
https://ds.163.com/feed/6af60b1b1652f12cda982281/
https://ds.163.com/feed/6af60b651652f12cda9822ab/
https://ds.163.com/feed/6af60b7069a7294ae370872e/
https://ds.163.com/feed/6af60b7cd9851f70eca2b247/
https://ds.163.com/feed/6af60bb139537f331446ba4f/
https://ds.163.com/feed/6af60bc46cd8a224d6cc2f77/
https://ds.163.com/feed/6af60bd47e55247b7d5e5a95/
https://bbs.xiaopeng.com/article/3972832
https://bbs.xiaopeng.com/article/3972831
https://bbs.xiaopeng.com/article/3972830
https://bbs.xiaopeng.com/article/3972829
https://bbs.xiaopeng.com/article/3972827
https://bbs.xiaopeng.com/article/3972826
https://bbs.xiaopeng.com/article/3972825
https://bbs.xiaopeng.com/article/3972822
https://bbs.xiaopeng.com/article/3972821
https://bbs.xiaopeng.com/article/3972820
https://bbs.xiaopeng.com/article/3972819
https://bbs.xiaopeng.com/article/3972818
https://bbs.xiaopeng.com/article/3972817
https://bbs.xiaopeng.com/article/3972816
https://bbs.xiaopeng.com/article/3972815
https://bbs.xiaopeng.com/article/3972814
https://bbs.xiaopeng.com/article/3972812
https://bbs.xiaopeng.com/article/3972811
https://bbs.xiaopeng.com/article/3972810
https://bbs.xiaopeng.com/article/3972809
https://bbs.xiaopeng.com/article/3972808
https://bbs.xiaopeng.com/article/3972807
https://bbs.xiaopeng.com/article/3972806
https://bbs.xiaopeng.com/article/3972805
https://bbs.xiaopeng.com/article/3972804
https://bbs.xiaopeng.com/article/3972803
https://bbs.xiaopeng.com/article/3972802
https://bbs.xiaopeng.com/article/3972801
https://bbs.xiaopeng.com/article/3972800
https://bbs.xiaopeng.com/article/3972799
https://bbs.xiaopeng.com/article/3972795
https://bbs.xiaopeng.com/article/3972797
