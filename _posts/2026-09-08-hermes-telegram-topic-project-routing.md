---
layout: post
title: "Hermes 实践笔记 01：让 Telegram Topic 自动回到正确的项目目录"
subtitle: "把 Topic、Session、工作目录和项目上下文拆开后，我做了一套路由方案"
date: 2026-09-08
author: BromdenX
header-img: img/post-bg-desk.jpg
catalog: true
tags:
  - AI-Agents
  - Personal-AI-OS
  - Technical-Notes
---

我把 Hermes Agent 常驻在一台 VPS 上，并通过 Telegram 使用它。随着讨论变多，我给英语学习、博客写作等长期主题分别建立了 Topic。这样聊天入口很清楚，但很快出现了一个问题：我会在 Topic 里用 `/new` 清理会话，新的 Session 却不应该忘记这个 Topic 对应哪个项目。

我想实现的效果很简单：

- Topic 是长期入口；
- Session 可以随时重建；
- 新 Session 仍能找到原来的项目目录；
- 没有绑定项目的聊天继续停留在中性的默认目录。

这篇文章记录我目前采用的轻量方案。它不是 Hermes Gateway 原生的 Topic 级 `cwd`，而是一层由路由表和 `AGENTS.md` 共同完成的逻辑绑定。

## 先把四个概念分开

一开始最容易混淆的是 Topic、Session、工作目录和 `AGENTS.md`。它们看起来都和“上下文”有关，实际职责不同。

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

- **Topic** 决定消息发到 Telegram 的哪个位置；
- **Session** 保存当前这段对话的历史，执行 `/new` 后可以换成新的 Session；
- **有效工作目录** 决定工具从哪里开始执行，以及 Hermes 从哪里发现项目上下文；
- **`AGENTS.md`** 保存项目级规则，它不是 Session，也不应该承担所有动态进度。

Hermes 官方文档分别介绍了 [Telegram 接入](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/telegram/)、[Session 管理](https://hermes-agent.nousresearch.com/docs/user-guide/sessions/) 和 [Context Files](https://hermes-agent.nousresearch.com/docs/user-guide/features/context-files/)。对我帮助最大的一点，是意识到 Topic 和项目目录之间并不存在天然的一对一关系。这层关系需要自己明确建立。

## 为什么不直接修改全局工作目录

我的 Hermes 不只处理 Personal AI OS，也会处理临时问题、系统维护和其他独立项目。如果把 Gateway 的默认目录直接改成某个大型项目，所有新对话都可能加载不相关的规则。

所以我保留 `/root` 作为中性入口，只在 Topic 精确命中路由时进入对应项目。这样有两个好处：

1. 没有绑定的聊天不会被某个项目“污染”；
2. 每个长期项目可以保留自己的边界、规则和状态。

这里的“进入”是逻辑意义上的：Agent 后续读取项目规则，并把文件和终端操作锚定到该目录。它不是文件系统沙箱，也不会阻止 Agent 访问其他获得授权的路径。

## 用数字 ID 建立路由表

Topic 名称适合人阅读，但可以被修改。稳定的键应该是 Telegram 提供的 `chat_id` 和 `thread_id`。

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

公开文章里隐去了真实 ID。实际配置必须使用可信 Session 元数据中的原始值，不能根据 Topic 名称猜测，也不要把 Token 或其他凭据放进这张表。

`name` 只是方便维护人员阅读，真正用于匹配的是两级数字 ID。

## 让根级 AGENTS.md 负责分流

仅有 YAML 不够，还要告诉 Agent 如何使用它。我的 Gateway 默认从 `/root` 开始，因此根目录的 `AGENTS.md` 只承担路由职责，不放任何具体项目知识。

规则可以概括为：

```text
1. 读取当前 Telegram 消息的 chat_id 和 thread_id。
2. 在 topic-workspaces.yaml 中做精确匹配。
3. 命中后，把 workdir 当作本轮项目根目录。
4. 读取项目 AGENTS.md，并按其恢复顺序继续。
5. 没有命中时留在 /root，不根据 Topic 名称猜目录。
```

最重要的是第四步。路由只负责“找到项目”，项目内部如何恢复上下文，应该由项目自己决定。

## 每个项目只保留三份入口文件

我没有一开始就建立复杂的知识库，而是给每个长期项目准备三份文件：

```text
project/
├── AGENTS.md
├── PROJECT.md
└── HANDOFF.md
```

它们的分工是：

- `AGENTS.md`：恢复顺序、工作规则和安全边界；
- `PROJECT.md`：长期目标、范围和不经常变化的原则；
- `HANDOFF.md`：当前状态、正在做什么、下一步是什么。

这个拆分解决了一个实际问题。如果把所有信息都写进 `AGENTS.md`，它会越来越长，动态进度也会不断冲击稳定规则。反过来，如果只依赖 Session 历史，执行 `/new` 后又很难可靠恢复。

我的原则是：稳定信息放 `PROJECT.md`，最小动态状态放 `HANDOFF.md`，详细证据和产物留在项目文件中。`AGENTS.md` 只做入口和规则索引。

## `/new` 之后如何恢复

同一个 Topic 执行 `/new` 后，新的 Session 不继承上一段会话作为唯一依据。恢复过程重新从稳定入口开始：

```text
新消息
  → 读取可信 Topic 元数据
  → 查询路由表
  → 定位项目目录
  → 读取项目 AGENTS.md
  → 读取 PROJECT.md 和 HANDOFF.md
  → 从当前任务继续
```

这也是为什么我没有把 Topic 永久绑定到某一个 Session。Topic 是长期导航，Session 是可以替换的工作上下文，两者的生命周期不同。

## 我怎样验证它没有误路由

路由规则写完不代表它真的可靠。我至少检查了下面几种情况：

- 已登记 Topic 能解析到预期目录；
- Topic 内执行 `/new` 后仍能恢复项目；
- 另一个 Topic 不会加载当前项目；
- 错误的 `chat_id` 或 `thread_id` 不会模糊匹配；
- 未登记 Topic 保持在 `/root`；
- 项目入口文件真实存在，且没有编造项目进度。

其中最重要的测试不是 YAML 能否解析，而是用户侧的 `/new` 实测。因为真正要验证的是完整链路：Telegram 元数据、Session 重建、路由规则和项目上下文恢复能否一起工作。

## 另一个容易遇到的问题：Topic 自动改名

我的 Topic 名称是长期导航，例如 `English` 和 `MyBlog`。如果 Hermes 在新 Session 第一次问答后用自动标题覆盖 Topic 名称，导航会逐渐失去原来的语义。

Hermes 提供了对应配置：

```yaml
gateway:
  platforms:
    telegram:
      extra:
        disable_topic_auto_rename: true
```

开启后，Hermes 仍可以保存内部 Session 标题，但不会修改 Telegram Topic 名称。这个选项不会改变 `chat_id`、`thread_id` 或 `/new` 的行为。官方 [Telegram 文档](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/telegram/) 也给出了该配置。

修改 Gateway 配置后还要让运行中的进程重新加载，并做一次真实 Topic 测试。配置文件里显示为 `true`，不等于当前进程已经使用它。

## 这套方案的边界

这套设计目前适合我的使用规模，但需要明确几个边界：

- 它是 Agent 规则层的逻辑路由，不是 Gateway 原生强制 `cwd`；
- 它不是安全沙箱，不能替代系统权限控制；
- 数字 ID 必须来自可信元数据；
- 路由表只保存映射，不保存秘密；
- 项目状态需要主动维护，尤其是 `HANDOFF.md`；
- 修改已经加载过的上下文文件后，新 Session 是最可靠的重新加载边界。

如果以后 Topic 数量明显增加，我可能会把解析和校验做成专用工具，减少模型每次读取路由表的成本。但在只有少量长期项目时，YAML、`AGENTS.md` 和三份项目文件已经够用，也比较容易审计。

## 这次实践带来的认识

这次配置让我更清楚地看到，Agent 的长期连续性不等于保留一段无限增长的聊天记录。更可靠的做法，是把不同生命周期的信息拆开：

- Telegram Topic 提供稳定入口；
- Session 承载阶段性对话；
- 项目目录保存产物；
- Context Files 恢复规则和状态；
- 路由表把入口重新连接到项目。

执行 `/new` 并不可怕。只要恢复路径足够明确，Session 可以是临时的，项目仍然可以连续。

## 参考资料

- [Hermes Agent：Telegram](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/telegram/)
- [Hermes Agent：Sessions](https://hermes-agent.nousresearch.com/docs/user-guide/sessions/)
- [Hermes Agent：Context Files](https://hermes-agent.nousresearch.com/docs/user-guide/features/context-files/)
