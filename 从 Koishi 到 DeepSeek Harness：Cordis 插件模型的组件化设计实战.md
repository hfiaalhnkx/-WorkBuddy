# 从 Koishi 到 DeepSeek Harness：Cordis 插件模型的组件化设计实战.md
## 前言
Cordis 并不是凭空诞生的AI智能体框架，它最早是为 Koishi 聊天机器人框架而生的底层元框架。在 Koishi 数年大规模社区实践中，Cordis 验证了**可逆副作用、生命周期管控、依赖拓扑**这套插件体系；而后 Cordis 演进到 v4，被集成进 DeepSeek Harness，把原本服务于聊天机器人的组件化范式，平移到大模型Agent运行时，实现了从“聊天机器人插件生态”到“全组件化智能体底座”的跨越。

很多开发者会混淆三者关系：**Koishi 是 Cordis v3 的落地验证项目；DeepSeek Harness 基于 Cordis v4 重构，将“一切皆插件”推向极致**。在 Harness 中，模型适配器、工具调用、会话存储、沙箱、Agent主循环、WebUI 全部是 Cordis 插件，不再有固化的核心业务代码。本文沿着这条演进路线，拆解 Cordis 组件化设计思路，对比两代场景差异，并给出可落地的实战开发思路。

## 一、起源：Cordis 在 Koishi 中的实践沉淀
Koishi 是跨平台聊天机器人框架，早期生态就面临典型插件痛点：机器人同时接入多IM平台，搭载数千社区插件，随时启停、增删功能；一旦插件卸载不干净，事件监听残留、定时器泄漏、数据库句柄未关闭，机器人运行一段时间就内存暴涨、异常崩溃。

为了解决这个痛点，Cordis 被抽象独立出来，作为 Koishi 的底层微内核。
### Koishi 场景下 Cordis 的核心设计要点
1. **插件作为独立业务单元**
适配器、指令、定时任务、数据库、权限系统，全部拆分为独立插件。社区沉淀了超过4000个插件，不同开发者独立开发，互不侵入内核代码。
2. **可逆副作用初版实现**
所有通过上下文 `ctx` 注册的事件、指令、定时器，会自动收集销毁器。卸载插件时逆序回滚副作用，解决资源泄漏。开发者不需要手动维护清理清单，只使用框架内置API即可。
3. **依赖图驱动启停顺序**
插件可以声明依赖其他插件提供的服务。例如弹幕适配器依赖数据库插件，内核自动构建有向图，保证被依赖组件优先激活，后停用。
4. **配置驱动组装**
通过配置文件声明需要加载的插件，支持动态热插拔，不用重启机器人服务，在线开启/关闭功能插件。

> 局限：Koishi 场景的插件模型面向**消息事件驱动**，插件之间通信以事件为主；状态大多是短会话状态，缺少长任务、多轮推理、子Agent调度、工具沙箱这类大模型Agent所需要的能力。Cordis v3 完成了插件生命周期与资源清理的验证，但还没有针对智能体任务编排做扩展。

经过Koishi大规模生态验证，证明这套“微内核+可逆副作用+依赖注入”模型可以支撑海量第三方插件共存、动态启停。于是 Cordis 继续迭代到 v4，升级后的版本成为 DeepSeek Harness 的底层基石。

## 二、演进：从聊天机器人到Agent运行时，Cordis v4 的架构升级
DeepSeek Harness（简称 dsh）定位是Agent的运行底座，公式：`Agent = 大模型 + Harness`。大模型负责推理思考，Harness 负责会话管理、工具调度、任务循环、沙箱隔离、日志追踪。而整个Harness，完全构建在 Cordis v4 之上，源码内置 `@deepseek-ai/cordis` 包，**Harness 自身没有不可拆分的单体核心**。

### 两大场景的核心差异
| 维度 | Koishi + Cordis v3 | DeepSeek Harness + Cordis v4 |
| ---- | ---- | ---- |
| 驱动模型 | IM消息事件驱动 | LLM推理+工具调用多轮任务驱动 |
| 插件范围 | 消息适配器、指令、数据库 | 模型适配器、工具、会话、沙箱、调度、UI、日志 |
| 状态特征 | 短会话、即时交互 | 长任务、可回溯、可fork、可恢复会话轨迹 |
| 副作用 | 事件、定时器、连接 | 外部文件读写、沙箱执行、子Agent派生、工具调用 |
| 组装能力 | 加载插件列表 | Bundle组合包 + Profile配置 + Patch多层配置覆盖 |

Cordis v4 在原有可逆副作用基础上，新增适配Agent场景的能力：
1. **时空可组合性（Spatiotemporal Composability）**
插件不仅可以静态组合，还能在任务运行的过程中动态替换、增删组件。例如在一段代码任务执行中途，替换模型适配器，或者临时加载代码沙箱插件，任务可以保持上下文继续运行。
2. **多层配置叠加机制**
引入 Profile、Bundle、Patch 三层组装模型：Bundle是一组配套插件打包；Profile定义整套Agent方案；Patch可以局部修改插件配置，多层配置逐级覆盖。开发者只需要修改配置，不需要改动源码，就能重组整套智能体。
3. **完整轨迹可观测**
所有插件产生的动作、模型输出、工具调用都会写入追加式会话日志，每一步操作都由对应插件产生，插件停用不会破坏历史轨迹，支持任务回溯、分支fork、重放调试。

## 三、Cordis组件化核心原理：服务注入、事件总线、可逆副作用
不管是Koishi还是DeepSeek Harness，Cordis组件化的三大支柱完全一致，这也是这套模型最核心的实战基础。

### 1. 服务注入：插件之间通过服务解耦
插件可以对外注册 `service`，其他插件在元信息里声明 `inject` 依赖列表，内核在激活阶段自动注入服务实例。
```typescript
// 服务提供者插件：llm-adapter
export const service = {
  async chat(messages) {
    // 调用大模型API
  }
}

// 使用者插件：tool-agent，声明依赖llm-adapter
export const inject = ['llm-adapter']
export function activate(ctx) {
  const llm = ctx['llm-adapter']
  ctx.on('task:run', async (task) => {
    const res = await llm.chat(task.messages)
  })
}
```
关键点：插件之间**不直接导入模块**，全部依赖内核上下文获取服务。当提供服务的插件deactivate，依赖它的插件会自动停用，保证运行时状态一致性。

### 2. 事件总线：插件通信的标准接口
Cordis 提供全局事件总线，插件通过 `ctx.on` / `ctx.emit` 发布订阅事件。
- Koishi：`message`、`command` 这类消息事件；
- DeepSeek Harness：`task:start`、`tool:call`、`session:append` 等智能体任务事件。

事件本身也属于可逆副作用，通过 `ctx.on` 注册的监听，插件卸载时自动解绑，不会残留事件句柄。

### 3. 可逆副作用：组件安全热插拔的底层保障
Cordis 的核心创新，就是所有通过ctx产生的副作用，都会自动生成销毁器，存入插件独立作用域。
- activate阶段：执行业务逻辑，收集销毁器；
- deactivate阶段：逆序执行销毁器，撤销所有副作用。

> 实战红线：**原生setInterval、addEventListener、直接全局变量修改，不属于Cordis托管副作用，卸载无法自动回滚，极易造成内存泄漏**。开发插件必须全部使用ctx封装的API。

## 四、DeepSeek Harness 实战：基于 Cordis 组装自定义Agent
下面以Harness的插件开发流程，演示Cordis组件化落地。

### 步骤1：定义插件元信息
每个插件导出元信息，声明名称、依赖、提供的服务。
```typescript
// my-search-plugin/index.ts
export const name = 'my-search-plugin'
export const inject = ['session', 'llm-adapter']
export function activate(ctx, config) {
  // 注册工具，作为可逆副作用
  ctx.tools.register({
    name: "web_search",
    description: "联网搜索",
    async invoke(query) {
      return await fetchSearchResult(query)
    }
  })
}
```
`ctx.tools.register` 是Harness封装的Cordis上下文API，注册工具操作会自动生成销毁器，插件停用的时候，该工具自动从Agent工具列表移除。

### 步骤2：通过 Profile 组装整套Agent
在 `cordis.yml` 中，声明加载哪些插件、配置Bundle包：
```yaml
profile:
  bundle:
    - dsh-base # Harness基础组合包，内置会话、日志、任务循环插件
    - my-search-plugin
patch:
  my-search-plugin:
    api_key: "xxx"
```
启动命令：`npx @deepseek-ai/dsh web`，内核自动解析依赖拓扑，按顺序激活全部插件。

### 步骤3：运行时热插拔
在WebUI插件面板，可以直接停用 `my-search-plugin`：
1. 进入deactivate阶段，执行销毁器；
2. web_search工具从工具注册表移除；
3. 事件监听、内部定时器全部撤销；
4. 会话历史日志保留，不会丢失任务轨迹。

整个过程**无需重启Harness**，其他插件（模型、会话、任务循环）持续正常运行，实现组件的在线替换。

### 典型实战场景
1. **按需加载工具**：代码任务加载沙箱插件；文档任务加载PDF解析插件；闲置时停用释放资源。
2. **替换模型后端**：不改动Agent任务循环插件，只替换llm-adapter插件，切换OpenAI、DeepSeek、本地Ollama模型。
3. **故障隔离**：某个工具插件抛出异常，单独停用该插件，其余Agent组件不受影响。

## 五、工程踩坑与最佳实践（来自Koishi与Harness社区经验）
1. **区分内部状态和外部状态**
Cordis可以回滚进程内副作用；但插件写入外部数据库、文件、第三方API的数据，属于外部状态，销毁器只能断开连接，**不能自动删除外部数据**。这类业务需要开发者在销毁器中自行处理数据清理或回滚逻辑。

2. **避免循环依赖**
Cordis内核会检测循环依赖并抛出异常。设计插件边界时，服务单向依赖，A依赖B，B不能再依赖A；跨组件通信优先用事件总线而不是双向注入。

3. **插件职责单一**
遵循组件化原则：一个插件只解决一类能力。不要在同一个插件里同时实现工具调用、会话存储、UI渲染。Koishi生态数千插件能稳定运行，正是依靠这种单一职责规范。

4. **测试插件独立启停**
单元测试要覆盖activate / deactivate多次循环，反复加载卸载插件，验证是否存在内存泄漏、残留定时器、事件堆积。这是Cordis插件最关键的测试项。

## 六、架构思考：Cordis组件化范式给Agent开发带来的启示
传统Agent项目，往往是单体架构：模型调用、工具解析、会话管理全部写在一套代码里。新增能力要修改主流程，替换底层模块需要重构，一旦组件故障整个Agent崩溃。

Cordis从Koishi的聊天机器人场景，演进到DeepSeek Harness的智能体底座，证明了一件事：**Agent可以被拆成一组相互独立、可单独启停、可替换的组件**。内核不承载业务逻辑，只负责生命周期、依赖解析、副作用管理。

这种架构带来几个长期价值：
- **生态可扩展**：第三方开发者独立开发插件，不需要修改主仓库源码；
- **能力按需组合**：根据任务场景，像搭乐高一样挑选插件组装Agent；
- **可运维、可热更**：线上环境动态替换、升级单个组件，不用整体重启；
- **可调试可观测**：每个动作归属对应插件，方便定位故障组件。

## 结语
从Koishi到DeepSeek Harness，Cordis插件模型完成了一次跨场景的范式迁移。Koishi验证了“可逆副作用+依赖拓扑”在事件驱动系统的可用性；DeepSeek Harness将这套组件化思想升级，应用在大模型Agent运行时，实现模型、工具、调度、UI全部插件化。

Cordis的核心不是“写插件”，而是**用生命周期约束所有组件的副作用，让软件系统在运行过程中可以安全重组**。随着Agent系统复杂度越来越高，这种可组合、可回滚、支持热插拔的组件化设计，会成为下一代智能体运行时的重要参考范式。

---



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
