# Agent Native UI 工程落地审计：Pen 与同花顺 Lumi 的结构、边界与实现方法

> 本文把 Pen/pen.dev 与同花顺远航版 Lumi 的实际实现拆成工程合同：产品对象是什么、点击如何变成语义事件、状态由谁拥有、智能体拿到什么上下文、能力如何发现、命令如何执行、配置放在哪里、怎样验收。
>
> 分析依据是本地 Pen 安装包和同花顺 Lumi 的静态源码/资源，以及之前对“分析智能炒股软件架构”对话中已经确认的运行时设计。Lumi 的宿主管道没有做真实端到端联调，所以文中把静态证据和已经确认的行为分开表述。

---

## 1. 先给结论：难点不是 UI，而是合同没有被显式建模

你觉得工程上“很吃力”，这个感觉是准确的。困难不是做一个聊天框，也不是把一个按钮接到一个模型，而是同时建立并保持下面十个合同：

| 合同 | 要回答的问题 | 没有它会出现什么 |
|---|---|---|
| 用户目标合同 | 用户要完成的是查看、定位、修改、比较还是持续监控？ | Agent 只能根据一句话猜，容易走错路 |
| 产品对象合同 | 被点击的东西究竟是节点、页面、证券、组件、数据集还是任务？ | UI 有对象，Agent 却没有稳定目标 |
| 交互事件合同 | 点击、切换、悬停、搜索、拖拽分别代表什么语义？ | 只能传坐标或模糊文本 |
| 运行时状态合同 | 当前页面、选中对象、权限、可用能力由谁提供？ | Agent 看到过期状态或自行脑补 |
| 能力合同 | 当前对象允许哪些动作，每个动作的参数和前置条件是什么？ | 接口能调用，但不知道是否适用于当前对象 |
| 执行合同 | 单步、批量、脚本、多步任务如何运行？ | 依赖顺序、页面切换、重试都会失控 |
| 配置合同 | 产品设置、模型设置、技能规则、权限和功能开关分别放哪里？ | 配置互相覆盖，调试无法判断生效来源 |
| 持久化合同 | 文档、会话、任务、快照、日志分别保存什么？ | 恢复时把历史记忆误当成当前事实 |
| 安全合同 | 哪些操作可读、可写、需确认、禁止执行？ | “工具成功”被误认为“业务成功” |
| 验收合同 | 怎样证明用户目标真的完成？ | 只能用“看起来可以”验收，无法稳定回归 |

Pen 和 Lumi 都已经解决了一部分问题，但解决的侧重点不同：

- **Pen 把设计文档对象做得很清楚**：节点有 ID、类型、层级、布局、组件引用和变量；Agent 可以读取节点树并执行语义化编辑。
- **Lumi 把业务宿主和运行时上下文做得更清楚**：当前证券、页面、焦点组件、最近交互、工具提示和权限可以进入 Agent 上下文；Agent 先发现工具和 Schema，再按业务路由执行。
- **两者都没有天然解决统一的用户目标、对象模型、任务状态、配置边界和验收协议**。这正是工程容易失控的地方。

可以把问题概括成一句话：

> 你们现在承担的不是“做 Agent UI”，而是在同时做一个产品模型、一个交互协议、一个宿主适配层、一个 Agent 编排器和一个可验证的业务执行系统。若这些边界不先写成合同，任何一个 UI 功能都会把多层逻辑搅在一起。

---

## 2. Pen 具体是怎样把“用户点了什么”交给 Agent 的

Pen 的产品位置是**设计文档编辑器**。它的第一目标不是执行业务流程，而是让用户编辑一个有层级结构的 .pen 文档。Agent 能理解用户操作，是因为 Pen 把画布上的点击转换成稳定节点身份，再允许 Agent 按需读取节点详情。

### 2.1 Pen 的对象模型

Pen 的文档不是一张截图，而是一个树状文档：

~~~text
Document
└── Frame / Group
    ├── Text
    ├── Rectangle / Ellipse / Path
    ├── Frame
    │   └── Text / Icon / Component instance
    └── Ref（可复用组件实例）
~~~

在 [pen-schema.md](<E:\UI-DESIGN\Pen\resources\app.asar.unpacked\out\skills\pen-dev\pen-schema.md:1>) 中，节点有 id、name、type、metadata、reusable、theme、enabled 等通用字段；Frame 还有布局、间距、内边距、对齐、填充、尺寸、裁剪和占位等属性。

Pen 的关键对象不是屏幕上的像素，而是：

~~~ts
type PenNode = {
  id: string;                    // 文档内稳定身份
  type: "frame" | "text" | "ref" | "path" | ...;
  name?: string;                 // 人和 Agent 共同使用的语义标签
  children?: string[];           // 文档层级
  reusable?: boolean;             // 是否是组件定义
  ref?: string;                  // 实例引用的组件 ID
  descendants?: Record<string, unknown>; // 实例级覆盖
  layout?: LayoutSpec;
  metadata?: Record<string, unknown>;
};
~~~

这给 Agent 一个重要前提：**先确定目标节点，再读取结构和属性**。它不需要把鼠标坐标当成业务接口。

### 2.2 Pen 的点击链路

实际链路可以还原为：

~~~text
用户点击画布
  ↓
sceneManager / selectionManager 命中画布节点
  ↓
selectionManager.selectedNodes 更新
  ↓
selectionChangeDebounced 事件
  ↓
IPC 通知 selection-changed
  payload = { nodes: [{ id, type, name }] }
  ↓
用户在 Agent 输入框中发送请求
  ↓
Agent 调用 get_app_state
  ↓
得到当前文件、选中节点、顶层节点、可复用组件、浏览器状态
  ↓
Agent 调用 Get(nodeId) 读取精确子树
  ↓
Agent 选择 Insert / Update / Move / Replace / Delete
  ↓
宿主执行，文档变脏，保存或自动保存并返回执行结果
~~~

静态包中的关键行为包括：

- selectionChangeDebounced 对外通知的只是节点的 id、type、name。
- get_app_state 读取当前编辑器路径、选中节点、最多十个顶层节点、可复用组件、集成浏览器 URL 和选中元素路径。
- Agent 不会因为用户点了一个节点就自动收到整个节点树；它需要继续用 Get(path) 获取详细结构。
- [execute.md](<E:\UI-DESIGN\Pen\resources\app.asar.unpacked\out\skills\pen-dev\execute.md:1>) 把修改接口集中到 Insert、Copy、Update、Replace、Move、Delete、Generate、SetVariables 等操作。
- 操作目标是 ID 或实例路径，而不是节点对象本身。实例子节点使用 instanceId/childId 这种路径。
- Update 不能任意改变节点的 id、type、ref 或 children；结构替换要用 Replace。

因此，Pen 的“理解用户点了什么”本质是：

~~~ts
type SelectedNodeHint = {
  id: string;
  type: string;
  name?: string;
};
~~~

这只是**目标提示**，不是完整语义。Agent 还要通过 Get 补齐：

- 它属于哪个页面或 Frame；
- 是组件定义还是组件实例；
- 组件实例是否有 descendants 覆盖；
- 当前布局和父子关系；
- 当前变量主题和样式；
- 是否会影响其他实例；
- 是否处在可见区域、是否被裁剪。

这也是 Pen 设计的优点和局限：

- 优点：对象身份和编辑命令足够稳定，适合设计编辑。
- 局限：选中事件很薄，当前用户目标主要还在聊天提示里；Agent 需要自己把“这个按钮”“这个卡片”映射回节点名、父级和文档结构。

### 2.3 Pen 的状态归属

| 状态 | 真实拥有者 | Agent 如何读取 | 是否应该持久化 |
|---|---|---|---|
| 当前选中节点 | Renderer 的 selection manager | selection-changed、get_app_state | 通常不进入文档 |
| 节点树和样式 | .pen 文档或 Document Manager | Get、getDocumentContent | 是 |
| 当前编辑器文件 | DesktopResourceDevice / App Window | get_app_state | 最近文件可持久化 |
| Agent 会话 | ChatStore / SQLite worker | 会话与消息接口 | 是，有生命周期 |
| 当前 Agent 光标和工具状态 | Agent session / UI | agent-session-event | 通常只保存必要事件 |
| 浏览器拾取状态 | Browser integration | get_app_state | 只保留当前会话语义 |
| 模型和集成配置 | electron-store / AgentConfigManager | 配置接口 | 是 |
| 文档是否脏、是否保存 | DesktopResourceDevice / OpenDocument | 文件状态接口 | 由文档系统负责 |

Pen 的重要原则是：**文档状态不能由聊天历史代替，聊天历史也不能代替当前文档状态**。用户重新打开文件时，Agent 必须重新读取真实文档，而不是只相信上一次的上下文。

---

## 3. 同花顺 Lumi 具体是怎样把“用户点了什么”交给 Agent 的

Lumi 的产品位置是**业务客户端内的 Agent 操作层**。它面对的对象不是任意画布节点，而是证券、页面、组件、数据和可执行业务动作。因此它需要比 Pen 更强的运行时上下文和能力治理。

### 3.1 Lumi 的对象模型

Lumi 的对象至少分成五类：

~~~text
Market
├── currentMarket / tradingPhase
Security
├── code / name / market / securityType / tradingStatus
View
├── pageRoute
├── focusComponent
└── visibleComponents
Information
├── data source / time range / field completeness
Interaction
├── navigation / click / switch / search / hover / chart action
Capability
├── open / inspect / getComponentData / client operation
~~~

在 [client-ops.json](<D:\TongHuaShun\同花顺远航版\lumi\resources\agents\client-ops.json:1>) 和运行时上下文投影代码中，Agent 可得到的核心事实包括：

- 当前市场和交易阶段；
- 当前证券及其身份；
- 当前页面路由；
- 当前焦点组件和可见组件；
- 最近若干次用户交互；
- 当前可用的工具提示；
- 权限、脱敏后的上下文和有限的详细信息。

Agent Manager 会把多个 Context Projector 合成一个受限的 RuntimeSnapshot，包括：

~~~ts
type LumiRuntimeSnapshot = {
  schemaVersion: string;
  currentMarket?: {
    market?: string;
    tradingPhase?: string;
  };
  currentSecurity?: {
    code?: string;
    name?: string;
    market?: string;
    securityType?: string;
  };
  view?: {
    pageRoute?: string;
    focus?: {
      status?: string;
      componentId?: string;
      componentName?: string;
    };
    currentComponent?: unknown;
    visibleComponents?: unknown[];
  };
  interaction?: {
    navigationPath?: unknown[];
    recentActions?: unknown[];
    clickAction?: unknown;
    switchAction?: unknown;
    searchAction?: unknown;
    hoverAction?: unknown;
    chartAction?: unknown;
  };
  toolHints?: Array<{
    intent?: string;
    toolName?: string;
    advisory?: string;
  }>;
  permissions?: Record<string, unknown>;
  versions?: Record<string, string>;
};
~~~

实际代码还会做几件关键的事：

- 对文本、组件、最近交互数量做预算限制；
- 对 authorization、cookie、credential、password、secret、token 等敏感字段脱敏；
- 过滤绝对本地路径；
- 先保证核心事实，再放可选事实；
- 让 Agent 得到当前页面事实，而不是无限增长的历史。

这说明 Lumi 不是把整个客户端 DOM 或屏幕截图丢给 Agent，而是由宿主主动挑选**业务上有意义的事实**。

### 3.2 Lumi 的动态能力发现

Lumi 的 Agent 不应该凭记忆猜一个远程工具名。它要求先动态发现：

~~~text
client_list / tool_search
  ↓
client_get_schema / 读取工具 Schema
  ↓
确认当前页面、组件、参数和权限
  ↓
client_call / tool_execute
  ↓
读取真实业务结果
  ↓
页面变化后重新 inspect
~~~

工具发现和执行规则记录在 [ops-reference.md](<D:\TongHuaShun\同花顺远航版\lumi\resources\built-in-skills\lumi-client-ops\references\ops-reference.md:1>)，路由规则记录在 [routing_rules.md](<D:\TongHuaShun\同花顺远航版\lumi\resources\built-in-skills\lumi-client-ops\references\routing_rules.md:1>)。

它把请求分成三层：

#### 第一层：导航问题

例如“条件选股在哪”“资金流向在哪”。

~~~text
feature_lookup
  → open
  → 告诉用户已打开的位置
~~~

这一层只负责定位功能，**不能假装已经拿到了业务数据**。

#### 第二层：宽泛查看问题

例如“看看贵州茅台的财务”。

~~~text
open
  → inspectPage
  → getComponentData
  → 按真实组件字段组织结果
~~~

这一层必须先进入页面，再读取页面和组件，不能因为知道一个后端接口就跳过宿主状态。

#### 第三层：精确字段或限定条件问题

例如“贵州茅台近三年 ROE”。

~~~text
识别证券、指标、时间范围
  → finance-data-router
  → 读取数据源和字段完整性
  → 返回来源、时间、缺失字段和结果
~~~

这与 Pen 的 Get 节点后执行 Update 不同：Lumi 的能力取决于**当前页面、当前证券、当前权限和真实运行时注册情况**。

### 3.3 Lumi 的执行边界

Lumi 的 Worker Prompt 和操作手册明确了这些约束：

- 禁止交易、委托、撤单、转账等高风险操作；
- 写操作串行执行；
- 页面变化后必须重新 inspect，不能继续使用旧的组件索引或 instanceId；
- client_op_batch 只用于参数和依赖全部明确的操作；
- 有依赖、循环或条件时使用受限脚本；
- 工具超时或结果未知时不能盲目重试写操作；
- 外层工具返回成功不等于业务目标完成；
- 结果需要明确 DONE、PARTIAL、BLOCKED 或 FAILED。

返回格式是一个很有价值的工程合同：

~~~text
STATUS: DONE | PARTIAL | BLOCKED | FAILED
RESULT: ...
FAILED_STEPS: ...
REMAINING: ...
NEXT: ...
~~~

这比只返回 { success: true } 强得多，因为它承认一个任务可能只完成了一部分，也可能停在权限、页面状态或数据缺失上。

### 3.4 Lumi 的验收方式

[test_cases.yaml](<D:\TongHuaShun\同花顺远航版\lumi\resources\built-in-skills\lumi-client-ops\tests\test_cases.yaml:1>) 中的验收重点不是“模型说得像不像”，而是检查边界行为：

- 导航问题是否只导航，不虚构数据；
- 缺少证券实体时是否先追问或阻塞；
- “活跃”“强势”等主观词是否追问定义；
- 比较请求是否保留比较对象和时间范围；
- 页面组件字段缺失时是否明确缺失；
- 没有适配器时是否不编造捷径；
- 交易相关请求是否拒绝；
- 任何安全失败是否导致整体失败。

它采用手工或 LLM-as-judge 方式，说明 Lumi 已经开始把**用户目标和禁止行为**纳入验收，而不只是测试工具是否能被调用。

---

## 4. Pen 与 Lumi 的本质差异：一个以文档图为中心，一个以运行时业务为中心

| 维度 | Pen | Lumi | 对统一系统的启发 |
|---|---|---|---|
| 产品位置 | 设计文档编辑器 | 业务客户端 Agent | 先定义宿主产品位置，再定义 Agent 权限 |
| 核心对象 | 节点、Frame、组件实例、变量 | 市场、证券、页面、组件、数据 | 需要统一对象引用，而不是统一所有对象字段 |
| 点击事件 | 选中节点 ID/type/name | 页面、证券、组件、最近操作 | 事件必须带语义和来源 |
| 当前事实 | 文档、选中节点、顶层节点、浏览器状态 | RuntimeSnapshot | 快照必须版本化、限量、脱敏 |
| 能力发现 | 固定的设计工具集 | 动态 client/tool discovery | 能力目录要支持静态和动态两种来源 |
| 读取细节 | Get(nodeId) | inspectPage、getComponentData | 所有修改前都需要 inspect |
| 写操作 | Insert/Update/Move/Replace/Delete | client_call、batch、脚本 | 写操作必须有前置条件和回读 |
| 组件语义 | ref、descendants、layout | componentId、业务字段、页面可见性 | UI 组件要带语义角色和可用动作 |
| 任务状态 | Agent session/tool events | DONE/PARTIAL/BLOCKED/FAILED | 引入统一 Task State Machine |
| 配置 | 文档变量、主题、Agent provider、集成 | 路由规则、技能、权限、工具注册 | 配置要分层并标明来源 |
| 主要风险 | 误改文档、实例覆盖、结构破坏 | 过期页面、错误数据、危险写操作 | 风险要按对象和动作分类 |
| 验收方式 | 结构校验、截图、编辑器状态 | 行为案例、业务结果、安全断言 | 视觉验收和业务验收要分开 |

不要直接把 Pen 的节点 API 和 Lumi 的业务工具合成一个巨大工具。它们的共同点是“有语义对象和可验证动作”，差异点应该留在 Adapter 内。

---

## 5. 为什么工程会吃力：七个具体的“边界债务”

### 5.1 用户目标没有被单独建模

“用户点击了某个按钮”不等于“用户想执行这个按钮的动作”。

至少要分开：

~~~ts
type UserGoal =
  | { kind: "understand"; subject: ObjectRef; question: string }
  | { kind: "locate"; feature: string }
  | { kind: "inspect"; subject: ObjectRef; fields?: string[] }
  | { kind: "edit"; subject: ObjectRef; changes: ChangeSet }
  | { kind: "compare"; subjects: ObjectRef[]; dimensions: string[] }
  | { kind: "monitor"; subject: ObjectRef; condition: Condition };
~~~

如果用户只说“把这个做得更专业”，这不是一个可直接执行的命令，而是一个需要先确定范围、目标风格和验收条件的目标。Agent 应该把它转成澄清状态，而不是直接修改十几个节点。

### 5.2 “选中对象”与“可执行对象”混在一起

Pen 的选中节点是一种对象提示；Lumi 的焦点组件是另一种运行时事实。真正可执行的目标还需要：

~~~ts
type ObjectRef = {
  objectType: string;
  objectId: string;
  owner: "document" | "host" | "server";
  parentRef?: ObjectRef;
  version?: string;
  source: "selection" | "route" | "search" | "runtime";
};
~~~

没有 owner 和 version，系统不知道这个对象当前由谁拥有，也不知道 Agent 手里的引用是否过期。

### 5.3 UI 状态、宿主状态和 Agent 状态没有分层

常见错误是把 React 或 Vue 状态、IPC 消息、Agent prompt、数据库记录都叫“当前状态”。它们的真实性不同：

- UI 本地状态只说明组件当前如何显示；
- 宿主状态是实际文档、页面或客户端事实；
- RuntimeSnapshot 是宿主事实的有限投影；
- Agent Task State 是推理和执行进度；
- 数据库是历史记录，不一定是当前事实。

如果没有状态所有权表，任何一层都可能写另一层的字段，最后出现“界面显示成功，宿主没有变化”或“历史看起来正确，当前页面其实已经切换”的问题。

### 5.4 视觉组件没有语义能力

一个漂亮的 Button 只知道颜色、尺寸和 hover 状态，Agent 还需要知道：

~~~ts
type SemanticComponent = {
  componentId: string;
  objectRef: ObjectRef;
  role: "navigation" | "filter" | "data-view" | "editor" | "confirmation";
  state: Record<string, unknown>;
  actions: CapabilityRef[];
  dataRef?: DataRef;
  owner: "ui" | "host" | "server";
  version: string;
};
~~~

没有 role、actions、objectRef 和 version，这个按钮只是可点击的像素。

### 5.5 “万能 execute”与“碎片化业务工具”之间没有中间层

Pen 的 execute 很强，能执行 JS 和设计工具；Lumi 的工具更受治理，但需要动态发现。两种极端都有代价：

- 只有万能 execute：Agent 能做很多事，但权限、前置条件、结果语义和审计边界不清楚。
- 只有碎片化工具：安全清晰，但每个新页面都要注册工具、写 Schema、补路由和测试。

工程上需要增加一个中间层：**能力注册表 + 语义命令**。工具实现可以不同，但向 Agent 暴露统一描述：

~~~ts
type CapabilityDescriptor = {
  name: string;
  intent: string[];
  inputSchema: JsonSchema;
  targetTypes: string[];
  readWrite: "read" | "write";
  preconditions: Predicate[];
  sideEffects: string[];
  idempotency: "idempotent" | "keyed" | "unknown";
  confirmation: "none" | "explicit" | "forbidden";
  verify: VerificationSpec;
  source: "document" | "host" | "server" | "skill";
};
~~~

### 5.6 配置和代码承载没有边界

下面这些东西经常被放进同一个配置文件，导致系统无法判断谁负责：

| 内容 | 应该放在哪里 |
|---|---|
| 用户喜欢深色还是浅色 | 产品设置 |
| 选哪个模型和 API Provider | Agent 配置 |
| 交易功能是否禁止 | 安全策略 |
| “先 inspect 再写入” | 技能或操作策略 |
| 当前页面有哪些组件 | 宿主运行时注册 |
| 证券字段如何映射 | 数据路由配置 |
| 某按钮的颜色和布局 | UI/设计系统 |
| 某功能是否灰度开放 | Feature Flag |
| 这次任务执行到哪一步 | Task Store |
| 上次聊天说了什么 | Conversation Store |

配置不只是“把值挪到 JSON”。每一项都需要有 owner、scope、source、version 和生效时间。

### 5.7 验收仍然停留在自然语言

“智能体理解用户点击”是一个愿望，不是验收条件。可执行验收必须写成：

~~~text
Given 当前文件打开，用户选中 id=card-12
When 用户说“把这个卡片的标题改成销售额”
Then Agent 必须：
  1. 读取 card-12 的实际类型和子树；
  2. 确认存在标题文本子节点；
  3. 只修改标题文本；
  4. 回读 card-12；
  5. 报告修改后的文本和文档 dirty 状态；
  6. 如果标题不存在，返回 BLOCKED，不得创建同名节点冒充完成。
~~~

没有这种验收，系统很容易在演示里“看起来懂了”，在边界条件下却无法解释。

---

## 6. 建议采用的统一模型：Object、View、Task 三层

不要试图用一个巨大上下文承载全部信息。建议把 Agent Native UI 拆为三个正交层。

### 6.1 Object 层：系统正在处理什么

Object 层只描述身份、类型、归属和版本：

~~~ts
type ObjectRef = {
  objectType: string;
  objectId: string;
  owner: "document" | "host" | "server";
  parentId?: string;
  version?: string;
  source: "selection" | "route" | "search" | "runtime";
};
~~~

Pen 例子：

~~~json
{
  "objectType": "pen.node",
  "objectId": "card-12",
  "owner": "document",
  "parentId": "dashboard-frame",
  "version": "doc:42",
  "source": "selection"
}
~~~

Lumi 例子：

~~~json
{
  "objectType": "stock.security",
  "objectId": "sh.600519",
  "owner": "host",
  "version": "security:20261026T1015",
  "source": "runtime"
}
~~~

### 6.2 View 层：用户当前看到什么

View 层描述路由、焦点、可见对象和当前交互：

~~~ts
type ViewState = {
  route: string;
  focus?: ObjectRef;
  visibleObjects: ObjectRef[];
  selection: ObjectRef[];
  viewport?: {
    x: number;
    y: number;
    width: number;
    height: number;
    zoom?: number;
  };
  recentInteractions: InteractionEvent[];
};
~~~

它不替代产品数据，也不替代文档本身。它只回答“用户当前处于什么视图”。

### 6.3 Task 层：用户希望达成什么结果

Task 层描述目标、步骤、状态、证据和下一步：

~~~ts
type TaskState =
  | "draft"
  | "clarifying"
  | "planned"
  | "running"
  | "succeeded"
  | "partial"
  | "blocked"
  | "failed"
  | "unknown";

type AgentTask = {
  taskId: string;
  goal: UserGoal;
  status: TaskState;
  steps: TaskStep[];
  currentStep?: string;
  evidence: Evidence[];
  createdAt: string;
  updatedAt: string;
};
~~~

### 6.4 统一运行时上下文

把这三层组合成一个有版本的快照：

~~~ts
type RuntimeContextEnvelope = {
  schemaVersion: "1.0";
  source: "pen" | "lumi" | "browser" | "native";
  sessionId: string;
  capturedAt: string;

  view: ViewState;
  selection: ObjectRef[];
  subject?: ObjectRef;
  userGoal?: UserGoal;

  capabilities: CapabilityDescriptor[];
  permissions: PermissionSnapshot;

  versions: {
    document?: string;
    host?: string;
    runtime?: string;
    capabilityCatalog?: string;
  };

  limits: {
    maxObjects: number;
    maxInteractions: number;
    redactions: string[];
  };
};
~~~

快照要带 capturedAt 和版本；Agent 不能把它当永久事实。页面切换、文档变化、证券切换或权限变化后，都要刷新。

---

## 7. 语义组件接口：组件必须同时服务用户和 Agent

推荐把 UI 组件拆成三部分：

~~~text
Visual Component
  ├── visual props：颜色、尺寸、布局、图标
  ├── semantic props：objectRef、role、state、actions
  └── interaction adapter：把 click/switch/search 转成语义事件
~~~

一个可被 Agent 理解的组件接口可以长这样：

~~~ts
type AgentAwareComponentProps = {
  componentId: string;
  objectRef?: ObjectRef;
  role: "navigation" | "data-view" | "filter" | "editor" | "confirmation";
  state: Record<string, unknown>;
  actions: CapabilityRef[];
  dataRef?: DataRef;
  disabledReason?: string;
  onSemanticEvent?: (event: SemanticInteractionEvent) => void;
};

type SemanticInteractionEvent = {
  eventId: string;
  kind:
    | "click"
    | "select"
    | "switch"
    | "search"
    | "hover"
    | "drag"
    | "submit"
    | "cancel";
  componentId: string;
  objectRef?: ObjectRef;
  payload?: Record<string, unknown>;
  beforeVersion?: string;
  afterVersion?: string;
  occurredAt: string;
  source: "user" | "agent" | "system";
};
~~~

这里有三个关键点：

1. 事件携带的是 componentId 和 objectRef，不是屏幕坐标。
2. 事件有 beforeVersion 和 afterVersion，能判断是否发生了并发或过期。
3. source 区分用户、Agent 和系统，便于回放、撤销和审计。

### 7.1 点击事件应该如何进入 Agent

推荐只在两处把上下文交给 Agent：

- 用户主动发送消息时，注入当前快照；
- 用户明确要求“解释刚才的操作”时，注入最近语义事件。

不要每一次点击都自动触发 Agent 推理。否则上下文噪声过大，模型会把普通浏览误认为任务，成本和响应时间不可控，用户也无法区分“我在浏览”和“我要求你执行”。

### 7.2 UI 中应该显示哪些 Agent 状态

建议在 UI 上提供可见但不打扰的状态投影：

~~~text
当前对象：卡片「销售概览」
当前页面：Dashboard / Overview
可用动作：查看结构、改标题、移动位置
任务状态：等待确认 / 执行中 / 已完成 / 被阻塞
证据：已回读节点 card-12，标题为「销售额」
~~~

这比只显示一个旋转中的“思考中”更有用。用户可以判断 Agent 当前使用的对象、正在执行的动作以及完成依据。

---

## 8. 命令、结果和状态机：不要只返回 success

统一命令结构：

~~~ts
type SemanticCommand = {
  commandId: string;
  capability: string;
  target: ObjectRef;
  input: Record<string, unknown>;
  expectedVersion?: string;
  preconditions: Predicate[];
  idempotencyKey?: string;
  requestedBy: "user" | "agent";
  confirmation?: {
    required: boolean;
    confirmedAt?: string;
  };
};
~~~

统一结果结构：

~~~ts
type SemanticCommandResult = {
  commandId: string;
  status:
    | "accepted"
    | "running"
    | "succeeded"
    | "partial"
    | "blocked"
    | "failed"
    | "unknown";
  target: ObjectRef;
  changedRefs?: ObjectRef[];
  finalState?: Record<string, unknown>;
  evidence?: Evidence[];
  failedSteps?: string[];
  remaining?: string[];
  next?: string[];
  error?: {
    code: string;
    message: string;
    retryable: boolean;
  };
};
~~~

### 8.1 为什么要有 unknown

写操作超时的时候，系统通常不知道：

- 宿主没有执行；
- 宿主执行了但没有返回；
- 宿主执行成功但页面刷新中；
- 宿主已经改变状态但回执丢失。

此时返回 failed 会诱导 Agent 重试，可能造成重复写入；返回 succeeded 又会误导用户。应返回 unknown，然后执行查询或让用户确认实际状态。

### 8.2 页面切换后的强制刷新

任何会改变路由、证券、焦点组件或文档结构的命令都应该带：

~~~ts
type RefreshPolicy = {
  invalidates: ("route" | "selection" | "component-tree" | "security" | "data")[];
  refreshBeforeNextStep: boolean;
};
~~~

Lumi 的“页面变化后重新 inspect”是这一原则的业务版本；Pen 的 Get 和结构校验是这一原则的文档版本。

---

## 9. 代码结构：把宿主差异收进 Adapter，把 Agent 协议固定下来

建议的目录边界如下：

~~~text
src/
├── domain/
│   ├── object-ref.ts
│   ├── user-goal.ts
│   ├── semantic-component.ts
│   ├── capability.ts
│   ├── command.ts
│   └── task.ts
│
├── runtime/
│   ├── context-envelope.ts
│   ├── context-store.ts
│   ├── context-normalizer.ts
│   ├── versioning.ts
│   └── redaction.ts
│
├── host-adapter/
│   ├── host-adapter.ts
│   ├── pen/
│   │   ├── pen-document-adapter.ts
│   │   ├── pen-selection-adapter.ts
│   │   └── pen-capability-provider.ts
│   ├── lumi/
│   │   ├── lumi-runtime-adapter.ts
│   │   ├── lumi-tool-discovery.ts
│   │   └── lumi-page-inspector.ts
│   └── browser/
│
├── capability-registry/
│   ├── registry.ts
│   ├── schema-validator.ts
│   ├── permission-policy.ts
│   └── capability-cache.ts
│
├── intent/
│   ├── intent-classifier.ts
│   ├── route-policy.ts
│   ├── clarification.ts
│   └── goal-normalizer.ts
│
├── task/
│   ├── orchestrator.ts
│   ├── planner.ts
│   ├── executor.ts
│   ├── retry-policy.ts
│   ├── task-store.ts
│   └── task-events.ts
│
├── verification/
│   ├── verifier.ts
│   ├── final-state-check.ts
│   ├── evidence-builder.ts
│   └── acceptance-cases/
│
├── ui/
│   ├── semantic-event-bus.ts
│   ├── selection-store.ts
│   ├── runtime-panel/
│   ├── task-timeline/
│   └── component-annotations/
│
└── persistence/
    ├── conversation-store.ts
    ├── document-store.ts
    ├── runtime-log.ts
    └── config-store.ts
~~~

### 9.1 各层的责任

| 层 | 负责 | 不应该负责 |
|---|---|---|
| domain | 稳定的数据合同 | 具体 Electron、IPC、React 细节 |
| runtime | 快照、版本、脱敏、上下文预算 | 业务动作执行 |
| host-adapter | 把宿主事实转换为统一对象和能力 | Agent 推理和用户目标判断 |
| capability-registry | 能力描述、Schema、权限和来源 | 直接决定用户想做什么 |
| intent | 目标分类、澄清和路由 | 直接修改文档或页面 |
| task | 计划、顺序、暂停、恢复、重试 | 自己猜宿主内部状态 |
| verification | 回读、断言、证据和最终状态 | 只看外层 HTTP success |
| ui | 展示和产生语义事件 | 伪造宿主真实状态 |
| persistence | 历史、文档、任务和配置存储 | 把历史记录当当前事实 |

### 9.2 一个具体的跨宿主接口

~~~ts
interface HostAdapter {
  getRuntimeContext(): Promise<RuntimeContextEnvelope>;
  inspect(target: ObjectRef, options?: InspectOptions): Promise<InspectResult>;
  listCapabilities(query: CapabilityQuery): Promise<CapabilityDescriptor[]>;
  getCapabilitySchema(name: string): Promise<JsonSchema>;
  execute(command: SemanticCommand): Promise<SemanticCommandResult>;
  refresh(invalidation: Invalidation[]): Promise<RuntimeContextEnvelope>;
  subscribe(listener: (event: HostEvent) => void): Unsubscribe;
}
~~~

Pen Adapter 的实现可以把：

- inspect 映射到 Get(nodeId)；
- execute 映射到 Insert/Update/Move/Replace/Delete；
- refresh 映射到重新读取文档和选中状态。

Lumi Adapter 的实现可以把：

- listCapabilities 映射到 client_list/tool_search；
- getCapabilitySchema 映射到 client_get_schema；
- inspect 映射到 inspectPage/getComponentData；
- execute 映射到 client_call/tool_execute；
- refresh 映射到新的 RuntimeSnapshot。

Agent 编排器只依赖 HostAdapter，不直接引用 Electron、Pen 节点 API 或 Lumi 的远程工具名。

---

## 10. 配置边界：哪些写代码，哪些写配置

建议把配置分为六层，并且每层都带 scope 和 source：

~~~text
1. Product Config
   产品名称、默认入口、导航、设计系统、默认主题

2. Agent Config
   模型、Provider、上下文预算、超时、会话策略

3. Capability Config
   能力名称、Schema、目标类型、读写属性、权限、验证器

4. Host Runtime Config
   页面、组件、工具注册、当前状态、连接信息

5. Policy Config
   禁止动作、确认规则、脱敏规则、重试规则、数据保留期

6. User / Workspace Config
   用户偏好、最近文件、个人工作区、允许的集成
~~~

配置不要通过“后写覆盖前写”隐式生效，建议显式计算：

~~~ts
type ConfigValue<T> = {
  value: T;
  source: "default" | "product" | "workspace" | "user" | "runtime";
  scope: string;
  version: string;
  effectiveAt: string;
};
~~~

例如“交易禁止”不能因为用户配置了 allowTrade: true 就被覆盖。安全策略应该拥有更高优先级，并且禁止用普通产品配置解除。

### 10.1 技能文件与代码的边界

以 Lumi 为例：

- routing_rules.md 适合描述路由规则、澄清边界和回答方式；
- ops-reference.md 适合描述操作纪律、工具发现和安全规则；
- client-ops.json 适合描述 Agent 角色、可用工具和返回格式；
- 宿主 Adapter 代码负责真实状态、工具注册和执行；
- test_cases.yaml 适合验收案例，但不应代替真正的运行时权限检查。

技能文件可以约束 Agent，但不能成为唯一安全边界。禁止交易、权限校验和参数校验必须由代码和宿主共同保证。

---

## 11. “用户点一下，然后 Agent 理解”应该如何分阶段实现

不要一上来实现任意页面、任意动作。先做一个完整纵向切片。

### 阶段 A：选中对象可解释

目标：用户选中一个对象，Agent 能准确说出它是谁。

实现：

1. 每个对象有稳定 ObjectRef；
2. SelectionStore 记录当前 selection；
3. 事件总线产生 selection-changed；
4. get-runtime-context 返回页面、对象、版本；
5. Agent 只做 inspect 和解释，不写入；
6. UI 显示“当前对象”和“来源”。

验收：

~~~text
用户选中一个对象后询问“我选中了什么”
必须返回对象类型、名称、父级和当前页面。
不能只返回“一个按钮”这种视觉猜测。
~~~

### 阶段 B：单步读取

目标：用户说“看看这个对象的结构或数据”，Agent 能读取真实内容。

实现：

1. Capability Registry 暴露 inspect；
2. Agent 先读取 Schema；
3. Host Adapter 执行 inspect；
4. 结果带 capturedAt、版本和缺失字段；
5. UI 以结果卡或结构树展示。

验收：

~~~text
对象不存在或已过期时返回 BLOCKED 或 FAILED；
不能根据对象名称编造不存在的字段。
~~~

### 阶段 C：单步写入和回读

目标：只修改一个明确目标，并证明修改成功。

实现：

1. 命令带 expectedVersion；
2. 权限策略先检查；
3. Host 执行一条写命令；
4. 立即回读目标；
5. Verifier 检查最终值；
6. Task 记录 evidence 和 changedRefs。

验收：

~~~text
用户说“把当前卡片标题改成销售额”
只有找到真实标题子节点并回读成功，才返回 SUCCEEDED；
找不到标题时返回 BLOCKED，不得创建一个同名文本冒充完成。
~~~

### 阶段 D：页面变化和多步任务

目标：支持“打开页面—定位组件—读取—修改—验证”。

实现：

1. 每一步产生新 RuntimeSnapshot；
2. 路由变化使旧引用失效；
3. 下一步前强制 inspect；
4. 需要依赖的步骤串行执行；
5. 任务时间线显示每一步状态；
6. 未知结果进入查询恢复，不自动重复写入。

### 阶段 E：多宿主适配

只有前面四阶段稳定后，再接 Pen、Lumi、Browser 等宿主。统一的是 ObjectRef、CapabilityDescriptor、SemanticCommand 和 TaskState，差异点应该留在宿主 Adapter 内。

---

## 12. 具体验收案例：用行为证明系统可落地

建议至少建立以下案例，不要只测模型最终回答。

### Case 1：点击理解

~~~text
Given 当前页面存在组件 card-12，用户点击它
When 用户说“我刚才点的是什么”
Then：
  - selection 中有 card-12；
  - Agent 能通过 inspect 得到真实类型和名称；
  - 回答包含页面、父级和对象版本；
  - 不得把坐标或截图区域当作唯一证据。
~~~

### Case 2：选中对象读取

~~~text
Given 用户选中 Pen 节点 card-12
When 用户说“看一下它的结构”
Then：
  - 调用 Get(card-12)；
  - 返回 children、layout、ref/descendants（如存在）；
  - 不修改文档；
  - 文档 dirty 状态不变。
~~~

### Case 3：当前证券数据读取

~~~text
Given Lumi 当前证券为 sh.600519，当前页面为财务页
When 用户说“看看近三年 ROE”
Then：
  - 识别证券和时间范围；
  - 选择数据路由；
  - 报告来源、年份、字段完整性；
  - 页面和数据不一致时说明差异。
~~~

### Case 4：页面变化导致旧引用失效

~~~text
Given Agent 已经读取 page-A 的组件索引
When 用户或 Agent 切换到 page-B
Then：
  - 旧索引不能直接写入；
  - 必须重新 inspect page-B；
  - 如果目标不存在，返回 BLOCKED。
~~~

### Case 5：未知写入结果

~~~text
Given 写命令已经发出，但宿主超时
When Agent 收到 timeout
Then：
  - 结果为 UNKNOWN；
  - 不得盲目重复写入；
  - 应先查询实际状态或让用户确认。
~~~

### Case 6：不允许的高风险动作

~~~text
Given 用户请求交易、委托或撤单
When Agent 识别到高风险能力
Then：
  - 不调用写工具；
  - 返回明确拒绝原因；
  - 不提供绕过限制的脚本。
~~~

### Case 7：目标含糊

~~~text
Given 页面中有多个“销售额”卡片
When 用户说“把销售额调大”
Then：
  - Agent 列出候选对象；
  - 询问具体目标或提供可选项；
  - 不随机选择一个对象执行。
~~~

### Case 8：部分完成

~~~text
Given 任务需要修改三个组件，其中一个组件不存在
When 前两个修改已验证成功
Then：
  - 状态为 PARTIAL；
  - 列出 changedRefs 和失败对象；
  - 给出剩余动作；
  - 不返回整体 DONE。
~~~

---

## 13. 应该怎样判断当前项目缺什么

你可以用下面这张表对照现有工程。如果一项回答不清楚，它就是当前“吃力”的来源之一：

| 问题 | 必须能指出的代码或配置 |
|---|---|
| 点击对象的稳定 ID 从哪里来 | selection store / host adapter |
| 谁拥有对象的真实状态 | document manager / host runtime / server |
| Agent 拿到的当前快照是什么 | context envelope 和 schema version |
| Agent 怎样知道当前能做什么 | capability registry / tool discovery |
| 一个动作的输入和输出是什么 | command/result schema |
| 页面变化后谁使旧引用失效 | versioning / invalidation policy |
| 写入成功怎样证明 | verifier / final-state check |
| 配置来自哪里 | config source、scope、优先级 |
| 历史会话和当前事实如何区分 | persistence contract |
| 失败时用户看见什么 | task timeline、status、evidence |
| 高风险动作由谁拦截 | policy layer + host enforcement |
| 能否回归验证 | acceptance cases 和自动或人工执行方式 |

如果现在主要是“组件代码里调用一个模型，然后模型自己决定下一步”，那问题不在模型能力，而在于上面的合同没有被拆出来。

---

## 14. 最实际的工程路线

我建议不要继续先堆 UI，也不要先做一个大而全 Agent。按下面顺序把工程压平：

1. **先写四个稳定合同**：ObjectRef、RuntimeContextEnvelope、CapabilityDescriptor、SemanticCommandResult。
2. **选一个宿主和一个对象类型**做完整纵向切片。例如 Pen 的“选中一个 Frame，读取结构，改一个标题，回读验证”。
3. **把 Selection、Runtime、Task 三个 Store 分开**，禁止 UI 直接修改宿主事实。
4. **实现 HostAdapter**，让 Agent 编排器不依赖 Pen 或 Lumi 的内部 API。
5. **实现 Capability Registry**，把动作的 Schema、权限、前置条件、幂等性和验证器一起注册。
6. **实现 unknown、partial、blocked 三种非成功状态**，让失败变成结构化结果。
7. **用行为案例验收**，每个案例同时检查 Agent 输出、宿主最终状态和禁止行为。
8. **最后再做视觉优化、自动建议、复杂多步任务和多宿主扩展**。

这样做的原因是：一旦一个纵向切片拥有对象身份、状态所有权、能力 Schema、执行结果和验收证据，后续扩展才是在复制边界，而不是继续复制不确定性。

---

## 15. 最终判断

Pen 给出了“设计对象可被结构化读取和修改”的答案；Lumi 给出了“业务宿主可以持续提供运行时事实，并通过动态能力和路由让 Agent 执行”的答案。

它们共同指向一个更具体的 Agent Native UI 定义：

> 用户通过 UI 操作真实对象；系统把操作转成带身份、版本和来源的语义事件；宿主发布当前运行时快照和能力 Schema；Agent 先确认目标和前置条件，再执行语义命令；宿主回读最终状态；验证器根据证据决定任务是完成、部分完成、阻塞、失败还是结果未知；UI 把对象、动作、进度和证据展示给用户。

你现在觉得“工程很难落地”，主要不是实现能力不足，而是项目同时跨越了产品建模、交互建模、宿主适配、Agent 编排和验收治理五个系统，却还没有把它们拆成独立合同。先把“对象是谁、状态归谁、能做什么、怎么证明完成”固定下来，UI、模型和工具才会从互相拉扯变成可以逐层实现的工程。

