 # Cordis 插件模型落地：构建可动态拼装、安全卸载的 AI 智能体运行时.md

## 前言
当 AI 智能体从单轮问答走向多工具协作、长任务执行、多场景部署，一个越来越突出的问题是：**智能体的能力组件能否像积木一样随时拼装，又能在不重启、不留残渣的情况下安全卸下？**

传统单体式 Agent 往往把模型调用、工具注册表、会话记忆、日志追踪、权限控制等逻辑写在一条主流程里。新增一个工具就要改动核心代码，停用一个组件要担心事件监听残留、定时器泄漏、数据库句柄未释放；线上热更时稍有不慎，整个智能体就会状态污染甚至崩溃。

Cordis 插件模型提供了另一条落地路径：以**微内核 + 可逆副作用 + 依赖拓扑**为骨架，把智能体的每一项能力拆成独立插件，运行时动态加载、拼装、停用与卸载。本文面向工程落地场景，梳理如何基于这套模型构建一个可动态组装、可安全回滚的 AI 智能体运行时，并拆解关键设计、代码骨架与实践红线。

## 一、为什么 Agent 运行时需要插件模型
在讨论 Cordis 落地之前，先明确智能体运行时与普通应用的差异，以及插件化要解决的具体工程问题。

### 1.1 Agent 运行时的典型组成
一个完整的智能体运行时通常包含：
- **模型适配器**：对接不同大模型（DeepSeek、OpenAI、本地 Ollama 等），封装对话补全、流式输出；
- **工具层**：联网搜索、代码沙箱、文件读写、数据库查询等可调用能力；
- **会话与记忆**：多轮上下文管理、历史轨迹存储、向量检索；
- **调度主循环**：接收用户输入 → 模型推理 → 工具调用 → 结果回填 → 继续推理；
- **横切能力**：日志、监控、权限校验、限流熔断、Web 管理界面。

这些组件在单体架构中互相引用，边界模糊；而在实际业务里，它们经常需要按场景增删、按版本升级、按故障隔离。

### 1.2 单体 Agent 的落地痛点
1. **耦合高、改动成本大**：换一个模型后端，可能要改动主循环、工具注册、记忆模块等多处代码；
2. **热插拔不可靠**：想在线停用一个出故障的工具插件，缺少统一的清理机制，容易残留监听和资源；
3. **故障扩散**：单个工具异常或内存泄漏，会拖垮整个智能体进程；
4. **场景复用困难**：不同业务线的 Agent 想复用记忆模块、沙箱模块，只能复制粘贴代码，难以独立演进；
5. **状态不可回溯**：组件启停过程缺乏统一生命周期事件，线上排查问题时难以还原组件加载顺序和残留状态。

Cordis 插件模型的落地目标，正是把这些痛点转化为可管理的工程约束：**每个能力组件独立成插件，内核统一负责调度、依赖解析与副作用回滚**。

## 二、落地架构：微内核 + 插件化能力层
### 2.1 整体分层
基于 Cordis 的智能体运行时可以分为三层：

```
┌─────────────────────────────────────────────┐
│  应用层：Profile 配置 / Bundle 组合包 / Patch 覆盖 │
├─────────────────────────────────────────────┤
│  插件层：模型插件、工具插件、记忆插件、沙箱插件、  │
│         日志插件、UI 插件等（可独立启停）          │
├─────────────────────────────────────────────┤
│  内核层：生命周期管理、依赖图、副作用容器、服务注入 │
└─────────────────────────────────────────────┘
```

- **内核层**：保持极薄，只提供插件注册、依赖解析、上下文隔离、销毁器调度等基础能力，不包含任何业务逻辑；
- **插件层**：所有具体能力以插件形式存在，对外暴露服务或注册事件、工具；
- **应用层**：通过配置文件声明本次运行要组装哪些插件、采用哪套组合方案，支持多层配置覆盖。

### 2.2 内核职责边界
落地时需要明确：内核不该做什么。以下能力都应交给插件实现：
- 具体的模型接入协议；
- 工具的定义与调用方式；
- 会话数据存储方式；
- 界面展示与交互。

内核只保留四件事：
1. 解析插件元信息与依赖关系，构建依赖图；
2. 按拓扑顺序激活、停用、卸载插件；
3. 收集每个插件产生的可逆副作用，并在停用阶段统一回滚；
4. 通过上下文向插件注入服务实例，隔离不同插件的作用域。

## 三、核心落地机制一：可逆副作用，实现安全卸载
安全卸载是插件模型最容易被忽视、却最影响线上稳定性的部分。Cordis 的核心做法是：**插件产生的每一项外部变更，都必须配套一个销毁器（disposer），由框架统一收集与调度**。

### 3.1 什么是可逆副作用
普通函数返回数据；而在 Cordis 插件上下文中，一个操作如果对外部状态产生了影响（注册事件、挂载工具、启动定时器、建立连接），它应当返回一个撤销函数。

```typescript
// 伪代码：通过上下文注册事件，自动获得销毁器
function registerMessageListener(ctx, callback) {
  const disposer = ctx.on('task:message', callback)
  // disposer 会被存入当前插件的副作用容器
  return disposer
}
```

插件激活阶段产生的销毁器，全部记录在该插件独立的作用域上下文里；停用阶段，框架按照**与注册相反的顺序**逐个执行销毁器：后创建的资源先释放，先创建的资源后释放，避免引用残留。

### 3.2 销毁器收集的典型副作用类型
在 Agent 运行时中，需要纳入托管的副作用包括：
- 事件总线订阅（监听模型回调、工具调用事件、会话更新事件）；
- 工具注册表项（向 Agent 工具列表注册可调用函数）；
- 定时任务与轮询器（心跳检查、会话过期清理、状态同步）；
- 外部连接（模型 API 长连接、数据库连接、消息队列订阅）；
- 上下文挂载的服务实例与中间件；
- UI 面板与可观测性埋点。

### 3.3 安全卸载的三个工程约束
1. **禁止脱离上下文创建资源**：直接使用原生 `setInterval`、`addEventListener`、全局变量修改，框架无法捕获，也就无法自动生成销毁器，是内存泄漏的主要来源。开发插件时必须使用上下文封装的 API。
2. **销毁器要具备异常保护**：单个销毁器执行报错时，框架应捕获异常、记录日志，并继续执行其余销毁器，尽可能完成全部回滚，避免清理流程中断导致部分资源残留。
3. **区分进程内状态与外部状态**：Cordis 可以回滚进程内的监听、连接、注册项；但插件写入数据库、文件、第三方系统的数据属于外部状态，框架不会自动删除，需要开发者在销毁器或业务补偿逻辑中手动处理。

### 3.4 卸载流程示例
以停用一个联网搜索工具插件为例，完整回滚顺序为：
1. 框架标记插件进入 deactivate 阶段；
2. 逆序执行销毁器：移除已注册的 `web_search` 工具、解绑搜索结果回调监听、停止定时刷新缓存的定时器、关闭 HTTP 连接；
3. 插件对外暴露的服务实例从上下文中注销；
4. 依赖该插件的其他插件先于它被停用；
5. 会话历史与日志轨迹保留，不因为插件卸载而丢失。

整个过程不需要重启 Agent 进程，其他插件继续运行。

## 四、核心落地机制二：依赖图与生命周期调度
插件之间存在依赖关系，例如工具插件需要调用模型服务插件，记忆插件依赖会话存储插件。落地时需要由内核构建有向依赖图，保证启停顺序正确。

### 4.1 生命周期四阶段
每个插件在运行时经历四个阶段：

| 阶段 | 动作 | 是否产生副作用 |
| ---- | ---- | ---- |
| install | 读取元信息、解析依赖、校验版本、加入依赖图 | 否，仅做元数据准备 |
| activate | 执行业务初始化逻辑，注册事件、工具、服务，收集销毁器 | 是 |
| deactivate | 逆序执行销毁器，回滚全部进程内副作用，保留代码与配置 | 是（回滚） |
| uninstall | 移除元数据与依赖引用，释放模块引用 | 否 |

> deactivate 与 uninstall 的区别：停用后插件代码仍在，可再次激活；卸载则会清理模块引用，再次使用需要重新安装。线上热更新通常走 deactivate → 替换代码 → activate，而不是直接 uninstall。

### 4.2 拓扑启停顺序
假设插件 B 依赖插件 A：
- **激活顺序**：先激活 A，再激活 B；
- **停用顺序**：先停用 B，再停用 A。

内核在依赖图中检测循环依赖时应直接抛出异常并拒绝加载，避免运行时出现双向引用死锁。跨插件通信优先使用事件总线，而不是双向服务注入。

### 4.3 服务注入解耦
插件可以对外注册服务，其他插件在元信息中声明依赖列表，内核在激活阶段自动把服务实例注入上下文。

```typescript
// 模型适配器插件：对外暴露 llm 服务
export const name = 'llm-adapter'
export const service = {
  async chat(messages) {
    // 调用模型 API
  }
}

// 工具插件：声明依赖 llm-adapter
export const name = 'search-tool'
export const inject = ['llm-adapter']

export function activate(ctx, config) {
  const llm = ctx['llm-adapter']

  ctx.tools.register({
    name: 'web_search',
    description: '联网检索资料',
    async invoke(query) {
      const result = await fetchSearchResult(query)
      return llm.summarize(result)
    }
  })
}
```

这种注入方式带来两个落地价值：
- 插件之间不直接 import 模块，通过上下文获取服务，替换实现时互不影响；
- 提供服务的插件停用后，依赖它的插件会被框架先停用，保证不会出现“调用了已失效服务”的运行时错误。

## 五、核心落地机制三：动态拼装与组合配置
可动态拼装意味着：同一套内核，通过不同插件组合和配置，就能产出面向不同场景的 Agent。落地时通常引入 Bundle、Profile、Patch 三层配置模型。

### 5.1 三层配置模型
- **Bundle（组合包）**：把一组配套插件打包，例如“基础对话包”“代码任务包”“办公文档包”；
- **Profile（方案）**：基于 Bundle 定义一整套 Agent 运行方案，选择模型、工具、记忆策略；
- **Patch（局部覆盖）**：在 Profile 基础上修改个别插件配置，例如更换 API Key、调整工具超时时间、禁用某个插件。

配置逐级叠加：基础 Bundle → Profile 默认值 → Patch 局部覆盖，最终生成本次运行的插件清单。

### 5.2 典型拼装场景
1. **代码分析 Agent**：基础会话插件 + 代码沙箱插件 + 代码解释工具 + 模型适配器；
2. **文档问答 Agent**：基础会话插件 + PDF 解析插件 + 向量检索插件 + 记忆插件；
3. **闲聊陪伴 Agent**：基础对话插件 + 轻量记忆插件，不加载重工具与沙箱，降低资源占用。

不同场景之间复用同一批插件，只改变组合方式与配置，避免代码复制。

### 5.3 运行时动态增删
在运行过程中，Agent 可以根据任务阶段动态拼装插件：
- 收到代码任务时，临时激活沙箱插件与代码工具；
- 任务结束后停用沙箱插件，释放文件系统与执行环境资源；
- 切换模型后端时，只替换模型适配器插件，主循环与工具插件保持不变。

这种动态拼装依赖前述的可逆副作用与依赖调度：组件增删过程状态可控、回滚完整。

## 六、工程落地：一个最小 Agent 运行时的插件结构
下面用目录结构和代码骨架，描述落地时的典型组织方式。

### 6.1 目录结构
```
agent-runtime/
├── kernel/                # Cordis 微内核封装（不写业务）
├── plugins/
│   ├── llm-deepseek/      # 模型适配器插件
│   ├── tool-web-search/   # 联网搜索工具插件
│   ├── memory-session/    # 会话记忆插件
│   ├── sandbox-code/      # 代码沙箱插件
│   └── logger-trace/      # 日志与轨迹插件
├── bundles/
│   ├── base/              # 基础组合包
│   └── code-agent/        # 代码任务组合包
├── profiles/
│   └── default.yml        # 默认 Agent 方案
└── src/
    └── main.ts            # 启动入口：读取配置、激活插件、启动主循环
```

### 6.2 插件标准结构
每个插件目录内通常包含：元信息声明、激活逻辑、服务导出、配置校验。

```typescript
// plugins/tool-web-search/index.ts
import { definePlugin } from 'cordis'

export default definePlugin({
  name: 'tool-web-search',
  inject: ['llm-adapter', 'session-store'],
  schema: {
    apiKey: { type: 'string', required: true },
    timeout: { type: 'number', default: 5000 }
  },

  service: {
    async search(query: string) {
      // 调用搜索 API
    }
  },

  activate(ctx, config) {
    // 注册工具：该操作由上下文托管，自动生成销毁器
    ctx.tools.register({
      name: 'web_search',
      description: '根据关键词检索公开网页资料',
      async invoke(query: string) {
        return ctx['tool-web-search'].search(query)
      }
    })

    // 注册事件监听：同样纳入副作用容器
    ctx.on('task:before-run', () => {
      ctx.logger.info('搜索插件已就绪')
    })
  }
})
```

### 6.3 启动与组装流程
入口代码的职责很薄：读取 Profile 配置 → 加载对应 Bundle 与 Patch → 交给内核解析依赖图 → 按拓扑顺序激活插件 → 启动 Agent 主循环。

```typescript
// src/main.ts
import { Runtime } from './kernel'
import profile from '../profiles/default.yml'

async function main() {
  const runtime = new Runtime()

  // 依据配置安装并激活插件
  await runtime.bootstrap(profile)

  // 启动 Agent 主循环（主循环本身也是一个内核插件）
  runtime.startLoop()
}

main()
```

### 6.4 配置文件示例
```yaml
profile:
  bundle:
    - base
    - code-agent
patch:
  llm-deepseek:
    model: "deepseek-chat"
    timeout: 10000
  tool-web-search:
    enabled: true
```

## 七、可观测性与故障隔离
插件化运行时要在线上稳定运行，必须配套可观测性与故障隔离机制，这些本身也以插件形式接入内核。

### 7.1 生命周期事件埋点
内核在 install、activate、deactivate、uninstall 各阶段发出事件，日志插件订阅后记录：
- 插件名称与版本；
- 激活/停用时间戳；
- 依赖关系；
- 销毁器执行结果与异常信息。

线上排查时，可以通过轨迹还原：某个时间点加载了哪些插件、停用顺序如何、是否存在清理失败的资源。

### 7.2 故障隔离策略
- 插件激活阶段抛出异常：内核回滚该插件已产生的副作用，阻断加载，并通知上层；
- 运行时插件内部异常：由对应插件的错误边界捕获，记录日志后可单独停用该插件，不扩散到主循环；
- 依赖服务失效：内核自动停用依赖它的下游插件，避免持续调用失效实例。

### 7.3 多次启停测试
落地实践中，单元测试应覆盖插件**反复 activate / deactivate 循环**：多次加载卸载后检查事件监听器数量、定时器数量、连接数是否回到初始水平，确认没有资源残留。这是插件质量验收的关键项。

## 八、落地红线与常见误区
### 8.1 常见误区
1. **把业务逻辑塞进内核**：为了省事把模型调用、工具注册写进内核代码，长期下来内核重新变厚，插件化名存实亡；
2. **插件职责过重**：一个插件同时实现工具调用、记忆存储和界面渲染，导致依赖关系混乱、无法独立复用；
3. **绕过上下文创建资源**：在插件代码里直接操作浏览器原生 API、全局定时器，框架无法托管，卸载后泄漏；
4. **误以为外部状态可自动回滚**：插件在数据库写入了任务记录，停用插件后数据仍然存在，需要业务层设计补偿或归档；
5. **忽略循环依赖检测**：插件之间互相注入服务，加载时形成环，运行时容易出现初始化顺序问题。

### 8.2 落地检查清单
- [ ] 所有外部副作用均通过上下文 API 创建，并自动纳入销毁器；
- [ ] 插件之间单向依赖，无循环注入；
- [ ] 外部数据写入有对应的业务清理或补偿方案；
- [ ] 生命周期事件接入日志与监控；
- [ ] 插件通过多次启停泄漏测试；
- [ ] 内核不包含具体业务实现，能力全部来自插件；
- [ ] 配置与代码分离，场景组合通过 Profile / Bundle 完成。

## 九、落地价值总结
基于 Cordis 插件模型构建 AI 智能体运行时，获得的工程价值主要体现在四个方面：

1. **可动态拼装**：按场景选择模型、工具、记忆、沙箱等插件，组合出不同 Agent，能力复用而不复制代码；
2. **可安全卸载**：可逆副作用机制统一管理资源回滚，插件在线停用不留监听、定时器、连接等残渣；
3. **可故障隔离**：单个插件异常可单独停用与回滚，不影响整个智能体运行；
4. **可独立演进**：第三方或业务团队可以独立开发、升级插件，通过依赖注入接入运行时，不需要改动内核与主流程。

## 结语
Cordis 插件模型落地到 AI 智能体运行时，本质上是把“软件组件的生命周期”变成一等工程约束：内核只负责调度与回滚，插件负责业务能力，配置负责场景拼装。它让 Agent 不再是一个难以拆分的单体程序，而是一组可以动态组合、安全卸载、独立迭代的能力单元。

在大模型应用日益复杂、工具与场景持续膨胀的背景下，这种微内核 + 可逆副作用的插件架构，为构建稳定、可扩展、可运维的智能体底座提供了一条清晰的落地路径。真正落地时，关键不在于“把代码拆成插件文件”，而在于**把每一项副作用纳入框架管理，把依赖边界收敛到内核调度之内**——做到这两点，动态拼装与安全卸载才不是口号，而是可验证、可运维的工程能力。






https://yuanzhuo.bnu.edu.cn/user/5c623e472e39f348655368ed6c6df501cb194579/about
https://yuanzhuo.bnu.edu.cn/user/7f4a0532d28294382f469e9e5d1f8104c76d799d/learn
https://yuanzhuo.bnu.edu.cn/user/20bd5e9b46f0d7a8f6b4753da8a4f70d59ac6a73/favorited
https://yuanzhuo.bnu.edu.cn/user/fece4199b016fb75d8e91b8ef1974f36dd54face/group
https://yuanzhuo.bnu.edu.cn/user/f7773eeed1f8d3425e7b5ed700a1bb6dcde73c36/following
https://yuanzhuo.bnu.edu.cn/user/6fb4b548a8c5a157b65acecfd279d19b9b5d7121/about
https://yuanzhuo.bnu.edu.cn/user/aa7e7a417f972a433169760b1a9e444f07081400/learn
https://yuanzhuo.bnu.edu.cn/user/958798d85500a2ae24cd2203b1ab18c96d4e7fd5/favorited
https://yuanzhuo.bnu.edu.cn/user/9e2660d21271ac0bd0ab141e330f0258b8efea7f/group
https://yuanzhuo.bnu.edu.cn/user/ff0a231c28cfa5b689645b8a76cabcbb1002b026/following
https://yuanzhuo.bnu.edu.cn/user/8fcb65bac7673ea51f15774231cfedaf68aa7bea/about
https://yuanzhuo.bnu.edu.cn/user/8d6e16b2520f82ad9adb53c3f68d15151d360b89/learn
https://yuanzhuo.bnu.edu.cn/user/92aeac6b47c55e3003dfb088f8fce4c3481be931/favorited
https://yuanzhuo.bnu.edu.cn/user/597f62f6c26e5d52055030624da88bf9a4f90eaa/group
https://yuanzhuo.bnu.edu.cn/user/eb719e5e16299407170171d596090a7a0f5326fa/following
https://yuanzhuo.bnu.edu.cn/user/be5b7f5c8965d8a44945ef774f81a81c4c7edc57/about
https://yuanzhuo.bnu.edu.cn/user/ddfbdaf6e77588301332e5c008d66b0cf1458aef/learn
https://yuanzhuo.bnu.edu.cn/user/8018ab5d2208bf7961bc5c924643f28ff1cbb789/favorited
https://yuanzhuo.bnu.edu.cn/user/5c623e472e39f348655368ed6c6df501cb194579/group
https://yuanzhuo.bnu.edu.cn/user/7f4a0532d28294382f469e9e5d1f8104c76d799d/following
https://yuanzhuo.bnu.edu.cn/user/20bd5e9b46f0d7a8f6b4753da8a4f70d59ac6a73/about
https://yuanzhuo.bnu.edu.cn/user/fece4199b016fb75d8e91b8ef1974f36dd54face/learn
https://yuanzhuo.bnu.edu.cn/user/f7773eeed1f8d3425e7b5ed700a1bb6dcde73c36/favorited
https://yuanzhuo.bnu.edu.cn/user/6fb4b548a8c5a157b65acecfd279d19b9b5d7121/group
https://yuanzhuo.bnu.edu.cn/user/aa7e7a417f972a433169760b1a9e444f07081400/following
https://yuanzhuo.bnu.edu.cn/user/958798d85500a2ae24cd2203b1ab18c96d4e7fd5/about
https://yuanzhuo.bnu.edu.cn/user/9e2660d21271ac0bd0ab141e330f0258b8efea7f/learn
https://yuanzhuo.bnu.edu.cn/user/ff0a231c28cfa5b689645b8a76cabcbb1002b026/favorited
https://yuanzhuo.bnu.edu.cn/user/8fcb65bac7673ea51f15774231cfedaf68aa7bea/group
https://yuanzhuo.bnu.edu.cn/user/8d6e16b2520f82ad9adb53c3f68d15151d360b89/following
https://yuanzhuo.bnu.edu.cn/user/92aeac6b47c55e3003dfb088f8fce4c3481be931/about
https://yuanzhuo.bnu.edu.cn/user/597f62f6c26e5d52055030624da88bf9a4f90eaa/learn
https://yuanzhuo.bnu.edu.cn/user/eb719e5e16299407170171d596090a7a0f5326fa/favorited
https://yuanzhuo.bnu.edu.cn/user/be5b7f5c8965d8a44945ef774f81a81c4c7edc57/group
https://yuanzhuo.bnu.edu.cn/user/ddfbdaf6e77588301332e5c008d66b0cf1458aef/following
https://yuanzhuo.bnu.edu.cn/user/8018ab5d2208bf7961bc5c924643f28ff1cbb789/about
https://yuanzhuo.bnu.edu.cn/user/5c623e472e39f348655368ed6c6df501cb194579/learn
https://yuanzhuo.bnu.edu.cn/user/7f4a0532d28294382f469e9e5d1f8104c76d799d/favorited
https://yuanzhuo.bnu.edu.cn/user/20bd5e9b46f0d7a8f6b4753da8a4f70d59ac6a73/group
https://yuanzhuo.bnu.edu.cn/user/fece4199b016fb75d8e91b8ef1974f36dd54face/following
https://yuanzhuo.bnu.edu.cn/user/f7773eeed1f8d3425e7b5ed700a1bb6dcde73c36/about
https://yuanzhuo.bnu.edu.cn/user/6fb4b548a8c5a157b65acecfd279d19b9b5d7121/learn
https://yuanzhuo.bnu.edu.cn/user/aa7e7a417f972a433169760b1a9e444f07081400/favorited
https://yuanzhuo.bnu.edu.cn/user/958798d85500a2ae24cd2203b1ab18c96d4e7fd5/group
https://yuanzhuo.bnu.edu.cn/user/9e2660d21271ac0bd0ab141e330f0258b8efea7f/following
https://yuanzhuo.bnu.edu.cn/user/ff0a231c28cfa5b689645b8a76cabcbb1002b026/about
https://yuanzhuo.bnu.edu.cn/user/8fcb65bac7673ea51f15774231cfedaf68aa7bea/learn
https://yuanzhuo.bnu.edu.cn/user/8d6e16b2520f82ad9adb53c3f68d15151d360b89/favorited
https://yuanzhuo.bnu.edu.cn/user/92aeac6b47c55e3003dfb088f8fce4c3481be931/group
https://yuanzhuo.bnu.edu.cn/user/597f62f6c26e5d52055030624da88bf9a4f90eaa/following
https://yuanzhuo.bnu.edu.cn/user/eb719e5e16299407170171d596090a7a0f5326fa/about
https://yuanzhuo.bnu.edu.cn/user/be5b7f5c8965d8a44945ef774f81a81c4c7edc57/learn
https://yuanzhuo.bnu.edu.cn/user/ddfbdaf6e77588301332e5c008d66b0cf1458aef/favorited
https://yuanzhuo.bnu.edu.cn/user/8018ab5d2208bf7961bc5c924643f28ff1cbb789/group
https://yuanzhuo.bnu.edu.cn/user/5c623e472e39f348655368ed6c6df501cb194579/following
https://yuanzhuo.bnu.edu.cn/user/7f4a0532d28294382f469e9e5d1f8104c76d799d/about
https://yuanzhuo.bnu.edu.cn/user/20bd5e9b46f0d7a8f6b4753da8a4f70d59ac6a73/learn
https://yuanzhuo.bnu.edu.cn/user/fece4199b016fb75d8e91b8ef1974f36dd54face/favorited
https://yuanzhuo.bnu.edu.cn/user/f7773eeed1f8d3425e7b5ed700a1bb6dcde73c36/group
https://yuanzhuo.bnu.edu.cn/user/6fb4b548a8c5a157b65acecfd279d19b9b5d7121/following
https://yuanzhuo.bnu.edu.cn/user/aa7e7a417f972a433169760b1a9e444f07081400/about
https://yuanzhuo.bnu.edu.cn/user/958798d85500a2ae24cd2203b1ab18c96d4e7fd5/learn
https://yuanzhuo.bnu.edu.cn/user/9e2660d21271ac0bd0ab141e330f0258b8efea7f/favorited
https://yuanzhuo.bnu.edu.cn/user/ff0a231c28cfa5b689645b8a76cabcbb1002b026/group
https://yuanzhuo.bnu.edu.cn/user/8fcb65bac7673ea51f15774231cfedaf68aa7bea/following
https://yuanzhuo.bnu.edu.cn/user/8d6e16b2520f82ad9adb53c3f68d15151d360b89/about
https://yuanzhuo.bnu.edu.cn/user/92aeac6b47c55e3003dfb088f8fce4c3481be931/learn
https://yuanzhuo.bnu.edu.cn/user/597f62f6c26e5d52055030624da88bf9a4f90eaa/favorited
https://yuanzhuo.bnu.edu.cn/user/eb719e5e16299407170171d596090a7a0f5326fa/group
https://yuanzhuo.bnu.edu.cn/user/be5b7f5c8965d8a44945ef774f81a81c4c7edc57/following
https://yuanzhuo.bnu.edu.cn/user/ddfbdaf6e77588301332e5c008d66b0cf1458aef/about
https://yuanzhuo.bnu.edu.cn/user/8018ab5d2208bf7961bc5c924643f28ff1cbb789/learn
https://yuanzhuo.bnu.edu.cn/user/5c623e472e39f348655368ed6c6df501cb194579/favorited
https://yuanzhuo.bnu.edu.cn/user/7f4a0532d28294382f469e9e5d1f8104c76d799d/group
https://yuanzhuo.bnu.edu.cn/user/20bd5e9b46f0d7a8f6b4753da8a4f70d59ac6a73/following
https://yuanzhuo.bnu.edu.cn/user/fece4199b016fb75d8e91b8ef1974f36dd54face/about
https://yuanzhuo.bnu.edu.cn/user/f7773eeed1f8d3425e7b5ed700a1bb6dcde73c36/learn
https://yuanzhuo.bnu.edu.cn/user/6fb4b548a8c5a157b65acecfd279d19b9b5d7121/favorited
https://yuanzhuo.bnu.edu.cn/user/aa7e7a417f972a433169760b1a9e444f07081400/group
https://yuanzhuo.bnu.edu.cn/user/958798d85500a2ae24cd2203b1ab18c96d4e7fd5/following
https://yuanzhuo.bnu.edu.cn/user/9e2660d21271ac0bd0ab141e330f0258b8efea7f/about
https://yuanzhuo.bnu.edu.cn/user/ff0a231c28cfa5b689645b8a76cabcbb1002b026/learn
https://yuanzhuo.bnu.edu.cn/user/8fcb65bac7673ea51f15774231cfedaf68aa7bea/favorited
https://yuanzhuo.bnu.edu.cn/user/8d6e16b2520f82ad9adb53c3f68d15151d360b89/group
https://yuanzhuo.bnu.edu.cn/user/92aeac6b47c55e3003dfb088f8fce4c3481be931/following
https://yuanzhuo.bnu.edu.cn/user/597f62f6c26e5d52055030624da88bf9a4f90eaa/about
https://yuanzhuo.bnu.edu.cn/user/eb719e5e16299407170171d596090a7a0f5326fa/learn
https://yuanzhuo.bnu.edu.cn/user/be5b7f5c8965d8a44945ef774f81a81c4c7edc57/favorited
https://yuanzhuo.bnu.edu.cn/user/ddfbdaf6e77588301332e5c008d66b0cf1458aef/group
https://yuanzhuo.bnu.edu.cn/user/8018ab5d2208bf7961bc5c924643f28ff1cbb789/following
https://yuanzhuo.bnu.edu.cn/user/5c623e472e39f348655368ed6c6df501cb194579
https://yuanzhuo.bnu.edu.cn/user/7f4a0532d28294382f469e9e5d1f8104c76d799d
https://yuanzhuo.bnu.edu.cn/user/20bd5e9b46f0d7a8f6b4753da8a4f70d59ac6a73
https://yuanzhuo.bnu.edu.cn/user/fece4199b016fb75d8e91b8ef1974f36dd54face
https://yuanzhuo.bnu.edu.cn/user/f7773eeed1f8d3425e7b5ed700a1bb6dcde73c36
https://yuanzhuo.bnu.edu.cn/user/6fb4b548a8c5a157b65acecfd279d19b9b5d7121
https://yuanzhuo.bnu.edu.cn/user/aa7e7a417f972a433169760b1a9e444f07081400
https://yuanzhuo.bnu.edu.cn/user/958798d85500a2ae24cd2203b1ab18c96d4e7fd5
https://yuanzhuo.bnu.edu.cn/user/9e2660d21271ac0bd0ab141e330f0258b8efea7f
https://yuanzhuo.bnu.edu.cn/user/ff0a231c28cfa5b689645b8a76cabcbb1002b026
https://yuanzhuo.bnu.edu.cn/user/8fcb65bac7673ea51f15774231cfedaf68aa7bea
https://yuanzhuo.bnu.edu.cn/user/8d6e16b2520f82ad9adb53c3f68d15151d360b89
https://yuanzhuo.bnu.edu.cn/user/92aeac6b47c55e3003dfb088f8fce4c3481be931
https://yuanzhuo.bnu.edu.cn/user/597f62f6c26e5d52055030624da88bf9a4f90eaa
https://yuanzhuo.bnu.edu.cn/user/eb719e5e16299407170171d596090a7a0f5326fa
https://yuanzhuo.bnu.edu.cn/user/be5b7f5c8965d8a44945ef774f81a81c4c7edc57
https://yuanzhuo.bnu.edu.cn/user/ddfbdaf6e77588301332e5c008d66b0cf1458aef
https://yuanzhuo.bnu.edu.cn/user/8018ab5d2208bf7961bc5c924643f28ff1cbb789
