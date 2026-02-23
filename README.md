# 共工（GongGong）
与你一起做事

他是一个专注于解决实际问题的本地化 AI 代理框架。系统借鉴了前沿的轻量级 Agent 架构，摒弃了臃肿的容器虚拟化，将大语言模型的推理能力与极轻量的“原生沙盒”相结合，提供从高层规划（HLP）到具体技能（Skill）执行的自动化闭环。

---

## 运行环境

* **系统环境**: Windows 10/11 (目前主要支持)，macOS (后续支持)。
* **通信总线**: 钉钉开放平台 API (采用 Stream 模式，主动拉取长连接，**无需内网穿透与公网 IP**)，预留其他 IM 扩展口。
* **执行环境**: 本机 Node.js 环境（零虚拟机开销）。
* **持久化存储**: SQLite (单文件本地落盘)。

## 详细技术栈选型

本系统采用极简、强类型的技术栈，以保障单机稳定运行并最小化资源占用：

* **核心运行时**: **Node.js v20+ & TypeScript 5.x**。
* **代理协同核心**: 借鉴Vercel AI SDK Core 的流式执行思想，构建纯函数驱动的 ReAct 循环。
* **通信网关**: **`dingtalk-stream-sdk-nodejs`**。钉钉官方长连接 SDK，剥离内网穿透带来的网络风险。
* **轻量级沙盒 (重点)**: 
    * **逻辑层**: **`isolated-vm`** (基于 V8 引擎的毫秒级隔离，用于执行不受信的清洗脚本)。
    * **系统层**: Node.js **`child_process`** 结合严格的 I/O 管控与超时熔断。
* **强类型约束**: **`zod`**。强制大模型按 JSON Schema 输出，提供函数调用防注入校验及幻觉自愈。
* **数据库与 ORM**: **`better-sqlite3`** (本地单线程极致读写) + **`drizzle-orm`** (无二进制依赖的轻量 TS ORM)。
* **任务调度**: **`node-schedule`** (纯内存定时器，配合 SQLite 实现状态恢复)。

## 技术栈

| 层 | 技术 |
|----|------|
| 框架 | Electron 40 |
| 前端 | React 18 + TypeScript |
| 构建 | Vite 5 |
| 样式 | Tailwind CSS 3 |
| 状态 | Redux Toolkit |
| AI 引擎 | Vercel AI SDK Core|
| 存储 | sql.js |
| Markdown | react-markdown + remark-gfm + rehype-katex |
| 图表 | Mermaid |
| 安全 | DOMPurify |
| IM | dingtalk-stream · @larksuiteoapi/node-sdk · grammY · discord.js |
---

## 核心模块实现机制

### 1. 轻量级沙盒与防爆机制 (Execution & Sandboxing)

为了防止 AI 执行的代码死循环或破坏宿主机，系统采用多重“轻隔离”策略，**启动速度在毫秒级，内存占用 < 50MB**：

* **纯函数执行沙盒 (`isolated-vm`)**:
    * 针对 JSON 清洗、文本提取等逻辑代码，注入到 V8 引擎独立的 Isolate 中执行。
    * **极限安全**: 沙盒内切断 `fs` (文件系统) 和 `net` (网络) 模块，内存强制封顶（如 64MB），彻底杜绝恶意代码越权。
* **原生脚本与子进程管控**:
    * 需要调用本地环境（如 Python 数据分析）时，通过 `child_process.spawn` 执行。
    * 强制绑定执行工作目录（`cwd: ./workspace`），并开启流式（Stream）监听。若执行时间超过 `timeout` 或内存异常飙升，主进程将无情下发 `SIGTERM` 强杀进程树。

### 2. 高层规划实现 (HLP - High Level Planning)

将模糊的自然语言需求转化为严谨的有向无环图（DAG）任务流。

* **防幻觉状态树**: 规划前，强制注入客观状态上下文（当前时间、工作区文件结构、DB Schema 等）。
* **ReAct 架构**:
    1. **Plan**: 拆解目标生成执行步骤。
    2. **Verify**: 基于前置条件检查依赖逻辑。
    3. **Execute**: 路由至具体 Skill 或轻量沙盒执行。
* **补偿自愈机制**: 若底层脚本报错，HLP 会提取 `stderr` 错误栈，自动生成修正代码并重试（严格限制重试次数阈值）。

### 3. 定时任务实现 (Cron Jobs)

赋予 AI 脱离即时对话的主动异步执行能力。

* **配置持久化**: 任务参数（CRON 表达式、HLP 模板、入参）落盘 SQLite。
* **主动触发流**: 任务到期生成无界面的虚拟会话（Headless Session），压入 HLP 并记录完整日志。
* **Misfire 恢复机制**: 系统断电重启后，调度器扫描数据库对比 `last_run_time`，对错过的关键任务执行补录或跳过策略。

### 4. 技能引擎 (Skills)

Agent 与外部世界交互的标准化触手。

* **强类型注册**: 所有 Skill 实现统一接口，入参由 `zod` 转换为 JSON Schema，支持未来向 MCP (Model Context Protocol) 协议平滑迁移。
* **核心内置 Skill**:
    * **WebSearch**: 内置 DOM 清洗器，剥离冗余 JS/CSS，仅提取高净值文本。
    * **DataAnalysis**: 转化自然语言为 Python/SQL 脚本下发执行。
* **人工审批流 (Human-in-the-loop)**: 调用高危 Skill（如发送邮件、覆盖文件）前自动挂起进程，向钉钉推送互动卡片，等待用户点击 `Approve` 后放行。

### 5. LLM 多源适配架构 (Provider Adapter)

避免供应商锁定，实现大模型灵活插拔，使用Vercel AI SDK Core。

* **统一 `ILLMProvider` 抽象**: 不硬编码特定 SDK。将 OpenAI、DeepSeek 或本地 Ollama 封装为标准适配器。
* **输入/输出归一化**: Adapter 层负责双向数据翻译，将大模型返回统一反序列化为标准的 `ToolCall` 或 `TextOutput`。
* **降本与高可用**: 支持基于任务复杂度的动态路由（轻量任务走本地，复杂代码生成走云端顶配），以及 API 限流时的静默降级。

### 6. 并发管控与多消息处理机制

针对长连接下高频、无序的外部输入，通过进程内管控确保本地资源稳定。

* **滑动窗口防抖 (Debounce)**: 设定 1500ms 窗口，合并同一会话的连续短消息，避免上下文碎片化。
* **Session 行级互斥锁**: 单会话严格串行。若前序任务执行中，新指令压入队列等待。遇“停止”类指令直接抢占锁并中止当前沙盒。
* **持久化死信队列 (DLQ)**: 关键指令入队前标记 `PENDING` 写入 SQLite。进程异常崩溃恢复后，率先拉取未消费消息重新执行。

---

## 使用方式

1. **服务启动**: 双击快捷方式或执行命令启动本地无界面的 Node.js 守护进程。
2. **日常交互**: 直接在绑定的钉钉群或私聊中艾特机器人下发自然语言指令，查收结果或进行卡片审批。
