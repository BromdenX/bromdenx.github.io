---
layout: post
title: "Hermes 实践笔记 01：从多层记忆到 Telegram Topic 项目路由"
subtitle: "Session 会结束，长期项目怎样继续保持上下文"
date: 2026-09-08
author: BromdenX
header-img: img/post-bg-desk.jpg
catalog: true
tags:
  - AI-Agents
  - Personal-AI-OS
  - Technical-Notes
---

我把 Hermes Agent 常驻在一台 VPS 上，并通过 Telegram 使用它。随着英语学习、博客写作等长期主题逐渐增多，我开始遇到一个很实际的问题：

> 如果我在 Telegram Topic 里执行 `/new`，新的 Session 怎样知道这里原来属于哪个项目？

一开始，我把它理解成一个简单的目录绑定问题。后来在 HermesCN 共学中深入讨论后，我发现它的基础其实是另一个问题：**Agent 的“记忆”不应该全部塞在一段无限增长的聊天记录里。**

Session、历史检索、长期 Memory、项目文件和 Skill 保存的是不同生命周期的信息。先把这些层次分开，Topic 到项目目录的路由才有清晰的设计依据。

这篇文章先整理我对 Hermes 多层记忆的理解，再记录如何用这套模型解决 Telegram Topic 的项目连续性问题。

## Hermes 的记忆不是一个文件

日常说“让 Agent 记住”，里面可能混着几种完全不同的需求：

- 在当前任务中记住刚才说过的话；
- 找回上周某次讨论的原始细节；
- 每次新会话都知道我的稳定偏好；
- 进入某个项目后恢复目标、规则和进度；
- 下次执行同类任务时复用已经验证过的步骤。

这些信息的适用范围、更新频率和失效条件都不同。如果全部写进全局 Memory，内容会迅速膨胀；如果全部依赖 Session，执行 `/new` 后又很难继续长期项目。

我现在把它们理解为三层原生记忆，加上一层项目治理。

## 第一层：当前 Session 与上下文压缩

当前 Session 是 Agent 的工作记忆。用户消息、Agent 回复和工具结果持续进入当前上下文，让多步骤任务保持连贯。

当对话越来越长时，Hermes 会对较早内容进行上下文压缩，在有限窗口中保留任务摘要和近期消息。压缩解决的是当前会话的容量问题，而不是把所有内容升级为长期事实。

这一层适合保存：

- 正在讨论但尚未确定的方案；
- 一次性参数和调试数据；
- 当前任务的中间结果；
- 很快会变化的临时状态。

它的特点是信息最丰富，但生命周期最短。执行 `/new` 后，新的 Session 不应该把这里的全部内容继续带上。

## 第二层：Session 历史与按需检索

Session 结束不等于历史记录消失。Hermes 会保存会话元数据和消息历史，并提供 `session_search` 按需搜索过去的对话。

这一层更像情景记忆：内容仍然存在，但不会在每次新会话中全部注入上下文。只有当当前任务需要回答“我们以前怎样决定的”或“上次做到哪里”时，再检索相关片段。

它适合：

- 找回某次讨论的原始理由；
- 核对历史决策；
- 从旧 Session 恢复遗漏细节；
- 避免把所有历史消息长期塞进提示词。

Session 历史是证据库，不等于当前项目状态。项目现在做到哪里，仍需要一个更短、更明确的权威入口。

## 第三层：全局长期记忆

Hermes 使用 `MEMORY.md` 和 `USER.md` 保存跨 Session 的稳定信息：

- `MEMORY.md`：环境事实、约定、工作流和长期经验；
- `USER.md`：用户偏好、沟通方式和稳定画像。

它们会在 Session 开始时以冻结快照的形式进入系统提示词。会话中对记忆文件的修改会立即持久化，但新的内容通常要到下一个 Session 才会成为启动上下文的一部分。

这一层适合真正跨项目、长期有效的信息，例如：

- 用户长期偏好；
- 多个项目都会使用的环境事实；
- 反复验证过的通用约束；
- 能避免未来重复调查的经验。

它不适合保存某个项目的全部细节。全局记忆会进入未来大量会话，写入越多，带来的上下文成本和错误影响范围也越大。

## 项目层：用 Context Files 管理长期项目

项目级记忆不是一个单独的 Hermes 全局 Memory 文件，而是我建立的一套上下文治理方式。每个长期项目使用独立目录，并保持三份入口文件：

```text
project/
├── AGENTS.md
├── PROJECT.md
└── HANDOFF.md
```

它们的职责分别是：

- `AGENTS.md`：Agent 的恢复顺序、行为规则和安全边界；
- `PROJECT.md`：项目为什么存在、长期目标、范围和成功标准；
- `HANDOFF.md`：当前进度、最近验证、阻塞和下一步。

详细证据、日志和正式产物继续放在项目目录中，而不是全部塞进入口文件。

这层设计补上了 Session 和全局 Memory 之间的空档：信息需要跨该项目的多个 Session 保留，但没有必要影响其他项目。

需要说明的是，**这是一套建立在 Hermes Context Files、工作目录和文件工具之上的治理模型，不是 Hermes 内置的一台“统一记忆生命周期状态机”。** Hermes 提供底层能力，哪些内容进入哪一层、何时更新和是否需要审批，仍然需要自己定义规则。

## Skill 是另一种“记忆”

如果把“记忆”理解得更广，Skill 也可以看作程序性记忆：它保存的不是“发生过什么”，而是“这类任务应该怎样做”。

例如，一套 Blog 发布 Skill 可以规定：

```text
整理素材
→ 校验文章
→ 生成 SHA-256
→ 用户确认
→ 提交发布
→ 验证线上页面
```

这类可复用流程不适合写进 `MEMORY.md`，也不应该依赖某次 Session 的聊天历史。官方架构里 Skill 是独立系统；把它称为程序性记忆，是为了帮助理解，而不是说 Hermes 官方把所有这些机制定义成同一个多层记忆模型。

## 记忆为什么要“晋升”和“降级”

在 HermesCN 共学讨论中，我最关心的是：这些层之间的规则能否自动处理？

我后来形成的判断是：**是否进入更长期的层，不只取决于“重要不重要”，而要同时看作用域、稳定程度、复用频率、敏感性和失效条件。**

可以把信息流简化为：

```text
当前 Session
  ↓ 反复出现、已经验证
项目日志或 HANDOFF
  ↓ 长期稳定、影响项目目标或规则
PROJECT.md / AGENTS.md
  ↓ 跨项目稳定复用
USER.md / MEMORY.md

重复出现且可以程序化的方法
  → Skill 候选
```

“晋升”是把已经验证、复用价值更高的信息移动到更长期的层；“降级”则是把过时、局部或不再常用的内容移出启动上下文，保留在日志、历史 Session 或归档中。

例如：

- 一次性的端口号留在当前 Session；
- 本轮已经验证的项目阻塞写入 `HANDOFF.md`；
- 项目的稳定目标进入 `PROJECT.md`；
- 跨项目都适用的长期偏好进入 `USER.md`；
- 重复验证过的发布流程整理成 Skill；
- 已完成的动态状态从 `HANDOFF.md` 移入日志或 Git 历史。

## 哪些可以自动，哪些需要人工确认

理论上，Agent 可以自动收集信息、判断候选作用域，甚至生成修改建议。但“可以自动生成”不等于“应该自动生效”。

我更认可下面这条流水线：

```text
自动采集
→ 生成候选
→ 判断作用域与置信度
→ 检查敏感性和冲突
→ 低风险写入，或对高影响变更请求确认
→ 写后验证
→ 保留审计和回滚依据
```

适合自动处理的内容包括：

- 有真实执行证据的日志；
- `HANDOFF.md` 中的最近验证和明确阻塞；
- 原始材料索引；
- 已经确认过的决策状态同步；
- 陈旧项和冲突项的检查报告。

更适合先生成候选或 diff 的内容包括：

- 新的长期偏好；
- 跨项目全局记忆；
- `PROJECT.md` 的目标和范围变化；
- `AGENTS.md` 的行为、安全与权限规则；
- 新 Skill 或现有 Skill 的关键流程变化。

需要人工确认的重点不是文件名，而是影响范围：越靠近全局、权限、安全和不可逆外部操作，写入越应该保守。

## 没有显式指定作用域时怎样判断

用户不会每次都说明“这句话只对当前 Session 有效”或“请写入项目 Memory”。因此 Agent 仍然要主动判断，只是不能盲目持久化。

我目前采用的信号优先级是：

```text
1. 用户当前明确指定的范围
2. 可信入口元数据，例如 chat_id + thread_id
3. Topic 到项目目录的精确路由
4. 当前工作目录及项目 AGENTS.md
5. 明确的任务对象、文件路径或仓库
6. 当前 Session 的连续上下文
7. 文本语义推断
```

前面的信号可以覆盖后面的信号。高置信度、低影响的信息可以按判断结果直接使用；置信度不足，或者会改变长期目标、安全规则和外部权限时，应该先询问。

正是这套作用域判断，把多层记忆问题带回到了 Telegram Topic。

## Topic、Session 和项目目录不是一回事

在 Telegram 里，我为长期主题分别建立了 Topic。Topic 适合作为稳定入口，但它不是 Hermes Session，也不天然对应一个项目工作目录。

```text
Telegram Topic（长期聊天入口）
        ↓  chat_id + thread_id
Topic 路由表
        ↓
项目目录（长期项目边界）
        ↓
AGENTS.md（恢复规则）
        ↓
PROJECT.md（稳定目标）+ HANDOFF.md（当前状态）
```

我的理解是：

- **Topic** 决定消息出现在 Telegram 的哪个位置；
- **Session** 承载当前阶段的对话，执行 `/new` 后可以重建；
- **工作目录** 决定项目文件和工具操作从哪里开始；
- **项目 Context Files** 负责跨 Session 恢复规则、目标和状态；
- **全局 Memory** 只保存跨项目仍然成立的稳定信息。

所以我没有把 Topic 永久绑定到某一个 Session，而是把它绑定到项目目录。Session 可以更新，项目归属保持稳定。

## 用数字 ID 建立轻量路由

Topic 名称可以被修改，真正稳定的键应该来自 Telegram 的 `chat_id` 和 `thread_id`。

我在 Hermes 状态目录中增加了一份路由表：

```yaml
version: 1
platforms:
  telegram:
    chats:
      "<telegram-chat-id>":
        topics:
          "<english-topic-thread-id>":
            name: English
            workdir: /root/english-growth
          "<blog-topic-thread-id>":
            name: MyBlog
            workdir: /root/myblog
```

公开文章里隐去了真实 ID。实际匹配只使用可信消息元数据中的原始值，不能根据 Topic 名称猜测，也不能把 Token 或其他凭据放进路由表。

Gateway 默认从 `/root` 开始，根目录的 `AGENTS.md` 只承担分流职责：

```text
1. 读取可信元数据中的 chat_id 和 thread_id。
2. 在路由表中做精确匹配。
3. 命中后，将 workdir 视为本轮项目根目录。
4. 读取项目 AGENTS.md，再按恢复顺序读取 PROJECT.md 和 HANDOFF.md。
5. 没有命中时留在 /root，不根据 Topic 名称猜项目。
```

这里的“进入项目”是 Agent 规则层的逻辑路由，不是 Gateway 原生强制 `cwd`，也不是文件系统安全沙箱。

## `/new` 后怎样恢复项目记忆

同一个 Topic 执行 `/new` 后，恢复路径重新从稳定入口开始：

```text
新消息
  → 读取可信 Topic 元数据
  → 查询路由表
  → 定位项目目录
  → 读取项目 AGENTS.md
  → 读取 PROJECT.md 和 HANDOFF.md
  → 必要时检索历史 Session 或详细日志
  → 从当前任务继续
```

这条路径把不同记忆层组合起来：

- 路由表回答“这是哪个项目”；
- `AGENTS.md` 回答“应该怎样工作”；
- `PROJECT.md` 回答“长期要做什么”；
- `HANDOFF.md` 回答“现在做到哪里”；
- Session 搜索和日志回答“过去具体发生了什么”；
- 全局 Memory 提供跨项目通用的稳定背景。

## 我怎样验证它没有误路由

配置文件能解析不代表完整链路可靠。我实际检查了下面几类情况：

- 已登记 Topic 能解析到预期目录；
- Topic 内执行 `/new` 后仍能恢复项目；
- 另一个 Topic 不会加载当前项目；
- 错误的 `chat_id` 或 `thread_id` 不会模糊匹配；
- 未登记 Topic 保持在中性目录；
- 项目入口文件真实存在；
- 恢复结果不把计划写成已经完成。

其中最关键的不是 YAML 检查，而是用户侧的 `/new` 实测。它验证的是 Telegram 元数据、Session 重建、路由规则和项目 Context Files 能否一起工作。

## Topic 自动改名的问题

长期 Topic 还是导航入口。如果 Hermes 在新 Session 第一次问答后用自动标题覆盖 Topic 名称，导航语义会逐渐丢失。

Hermes 提供了对应配置：

```yaml
gateway:
  platforms:
    telegram:
      extra:
        disable_topic_auto_rename: true
```

开启后，Hermes 仍可保存内部 Session 标题，但不再修改 Telegram Topic 名称。配置修改后还需要让运行中的 Gateway 重新加载，并做一次真实 Topic 测试。

## 这套模型的边界

这套设计适合我目前的使用规模，但它不是一套已经全自动完成的记忆系统：

- 多层记忆是对 Hermes 原生能力与个人项目治理的组合解释；
- 项目路由是 Agent 规则层的逻辑映射，不是安全沙箱；
- 项目文件需要维护，尤其是动态的 `HANDOFF.md`；
- 全局 Memory 容量有限，应该持续精炼而不是无限追加；
- Session 历史可以找回证据，但不能替代当前状态文件；
- 自动化适合生成候选和处理低风险事实，高影响规则仍需审批；
- 修改已加载的 Context Files 或全局 Memory 后，新 Session 是更可靠的重新加载边界。

如果以后项目明显增多，我会考虑把路由解析、候选记忆、过期检查和冲突检测做成专用工具。但在少量长期项目阶段，YAML、`AGENTS.md`、三份项目入口文件和 Hermes 原生 Session/Memory 已经能组成一套可审计的基础系统。

## 这次实践带来的认识

我现在不再把 Agent 的连续性理解成“保留一段永远不结束的聊天”。更可靠的做法，是让信息停留在与它生命周期相匹配的位置：

```text
短期推理留在 Session
历史细节按需检索
项目规则和状态放进项目目录
跨项目稳定事实进入全局 Memory
重复验证的方法沉淀为 Skill
```

Topic 路由只是这套模型的一个入口层实践。它解决的不是“让新 Session 记住所有旧聊天”，而是让新 Session 能先回到正确的项目，再按需恢复正确的那部分信息。

## 参考资料

- [Hermes Agent：Memory](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory/)
- [Hermes Agent：Sessions](https://hermes-agent.nousresearch.com/docs/user-guide/sessions/)
- [Hermes Agent：Context Files](https://hermes-agent.nousresearch.com/docs/user-guide/features/context-files/)
- [Hermes Agent：Telegram](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/telegram/)
