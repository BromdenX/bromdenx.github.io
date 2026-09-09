---
layout: post
title: "Hermes 实践笔记 02：让 Hermes 为自己搭建 R2 加密备份"
subtitle: "人负责授权和验收，部署、校验与排错交给 Agent"
date: 2026-09-09
author: BromdenX
header-img: img/post-bg-desk.jpg
catalog: true
tags:
  - AI-Agents
  - Personal-AI-OS
  - Technical-Notes
---

我把 Hermes 长期运行在 VPS 上之后，很快遇到一个基础问题：**这台服务器如果明天损坏，我能不能把 Hermes 恢复回来？**

Hermes 里积累的不只是配置文件，还有会话、Memory、Skills、Cron、OAuth 凭据和其他运行状态。我决定把完整备份放到 Cloudflare R2，并在上传前使用 `rclone crypt` 做客户端加密。

最初整理这套方案时，我写下了很多 rclone、systemd 和 Shell 配置。后来发现方向偏了：既然 Hermes 已经运行在 VPS 上，它可以检查环境、编写脚本、配置任务并执行验证。使用者真正需要掌握的不是每一条服务器命令，而是：

1. 怎样把任务和安全边界讲清楚；
2. 哪些授权必须自己完成；
3. 怎样验收 Hermes 的交付确实可以恢复。

## 一、我和 Hermes 怎样分工

### 我负责账户、授权和决策

- 开通 Cloudflare R2；
- 创建独立的备份 Bucket；
- 创建只允许访问该 Bucket 的 API Token；
- 决定备份频率、保留时间和通知位置；
- 把恢复密钥保存到 VPS 之外；
- 审阅实施方案和验收证据。

### Hermes 负责 VPS 内部的实施

- 检查系统、Hermes 和 rclone 的真实状态；
- 设计备份、加密、上传、校验和清理链路；
- 安装自动化任务并限制资源占用；
- 处理失败、并发和错过后的补跑；
- 执行非破坏性的备份与下载验证；
- 输出恢复手册和真实结果。

这样做不是放弃控制，而是把控制点从“手工输入每条命令”换成“明确授权范围并验收结果”。

## 二、Cloudflare R2 需要我准备什么

### 1. 开通 R2，创建专用 Bucket

登录 Cloudflare Dashboard，进入：

**Storage & databases → R2 → Overview**

如果账户尚未开通 R2，先按页面完成订阅或结算设置，再创建一个只用于 Hermes 备份的私有 Bucket。

截至本文核对时，R2 Standard storage 的月度免费层包括 10 GB-month 存储、100 万次 Class A 操作、1000 万次 Class B 操作，以及免费的直接网络流出。Infrequent Access 不适用同一套免费层，还可能产生读取费用。价格会变化，应以 [R2 Pricing](https://developers.cloudflare.com/r2/pricing/) 为准。

### 2. 创建最小权限凭据

进入：

**R2 Overview → Manage R2 API tokens → Create API token**

选择：

- **Object Read & Write**；
- **Apply to specific buckets only**；
- 只授权刚刚创建的备份 Bucket。

创建后保存 Access Key ID、Secret Access Key 和 S3 API endpoint。普通 endpoint 的形式是：

```text
https://<ACCOUNT_ID>.r2.cloudflarestorage.com
```

region 使用 `auto`。如果选择特定 jurisdiction，则使用对应的 jurisdiction endpoint。

Secret 创建后不能再次查看。我不会把它粘贴到聊天、文章、Git 或普通日志里，而是要求 Hermes 提供本机交互式录入方式，通过可信终端输入且不回显秘密。

## 三、我怎样把任务交给 Hermes

下面是我实际会使用的任务说明。时间、保留天数和通知位置可以自行修改。

```text
请为这台运行 Hermes Agent 的 VPS 建立 Cloudflare R2 加密备份和灾难恢复方案。

目标：
- 每天创建一次 Hermes 完整备份；
- 上传前加密内容、文件名和目录名；
- 本地保留 3 天，R2 保留 30 天；
- 错过执行时间后能够补跑，并避免并发执行；
- 失败时不得删除已有可用备份，并报告失败阶段。

安全边界：
- 不要要求我在聊天中粘贴 Access Key、Secret、Account ID 或 crypt 密码；
- 提供本机安全录入凭据的方法，不把秘密写入历史或日志；
- Token 只能访问指定 Bucket；
- 配置和恢复材料采用最小文件权限；
- 不把凭据、恢复配置或原始备份提交到 Git；
- 提醒我把恢复材料保存到 VPS 之外。

实施要求：
- 先检查系统、Hermes、rclone、时区、磁盘和已有任务，不要猜测；
- 使用 Hermes 当前版本的官方备份能力；
- 按“生成 → 本地校验 → 加密上传 → 远端校验 → 清理”执行；
- 根据 VPS 资源限制并发和内存；
- 先完成一次实际测试，再启用长期调度。

验收要求：
- 本地备份结构完整；
- crypt remote 能看到解密后的文件名；
- 远端对象与本地备份一致；
- 从 R2 实际下载一份到临时目录，再校验结构和哈希；
- 展示定时器状态、下一次运行时间和最近一次结果；
- 提供新 VPS 的恢复步骤，但不要在当前生产实例做破坏性导入；
- 明确哪些步骤已验证，哪些仍需隔离环境演练。

完成后给我一份简洁报告：
- 建立了什么；
- 凭据和恢复材料保存在哪里（只显示路径，不显示内容）；
- 备份频率与保留策略；
- 每项验收的真实结果；
- 仍然存在的风险；
- 我必须离线保存什么。
```

这里最重要的不是指定某条命令，而是把执行顺序、安全边界和验收证据写进任务本身。

## 四、我怎样验收

### 1. 备份范围经过核实

Hermes 应读取当前版本的 `hermes backup --help` 或官方文档，说明哪些 Hermes 状态会进入完整备份，哪些外部项目、系统脚本和服务需要单独保护。

不能只凭经验复制某个目录，然后把它称为完整备份。

### 2. R2 上保存的是客户端密文

需要区分两层能力：

- R2 API 凭据决定能否访问 Bucket；
- crypt 密码决定能否解密备份。

正常备份只通过 crypt remote 上传。R2 控制台不应直接看到 ZIP 明文和原始文件名。但对象数量、时间和大致体积仍可能被观察到，这不是匿名系统。

### 3. 验证内容，而不只是上传状态

最低证据包括：

- 本地 ZIP 完整性检查通过；
- 远端对象存在；
- `rclone cryptcheck` 无差异；
- 从 R2 实际下载成功；
- 下载后的 ZIP 能正常打开；
- 下载前后的 SHA-256 一致。

只看到“上传成功”或远端出现文件名，不足以证明备份可用。

### 4. 定时任务真的在运行

我会要求 Hermes 展示：

- 定时器已经启用并处于活动状态；
- 下一次运行时间符合预期时区；
- 最近一次任务退出码为 0；
- 错过时间后可以补跑；
- 并发启动不会产生两个备份任务；
- 失败日志能指出具体阶段。

### 5. 新备份验证后才清理旧副本

正确顺序应当是：

```text
生成新备份
→ 本地校验
→ 加密上传
→ 远端校验
→ 成功后清理过期副本
```

如果上传或校验失败，旧备份必须保留。

R2 的 Object Lifecycle Rules 可以作为存储侧第二道清理机制，但时间不应短于脚本保留期。Cloudflare 说明，过期对象通常在到期后的 24 小时内删除，因此它不是精确调度器。

### 6. 恢复密钥不只存在于原 VPS

R2 Token 可以撤销重建；crypt 的 `password` 和 `password2` 如果丢失，又没有其他副本，远端密文就无法恢复。

恢复材料至少应进入两个独立位置，例如密码管理器安全附件和离线加密介质。不能只保存在当前 VPS、Git 仓库、聊天记录或未加密网盘里。

### 7. 分级验证恢复能力

我把恢复验证分为三层：

1. **对象级**：能够列出、解密和下载；
2. **归档级**：下载文件结构完整、哈希一致；
3. **系统级**：在全新 VPS 或隔离环境中导入，验证 Memory、Sessions、Skills、Cron 和消息渠道。

前两层可以自动验证。第三层不应在当前生产 Hermes 上直接执行，而应单独安排恢复演练。

## 五、这次实际验收到了什么

完成部署后，我要求 Hermes 重新读取系统状态并实际验证。结果包括：

- 每日任务已启用并处于活动状态；
- 下一次运行时间符合预定时区；
- 最近一次任务执行成功，退出码为 0；
- crypt remote 能列出连续生成的备份；
- 最新本地 ZIP 结构检查通过；
- 本地备份与 R2 crypt remote 校验无差异；
- Hermes 从 R2 实际下载了最新备份；
- 下载后的 ZIP 再次通过检查；
- 下载文件与本地文件的大小和 SHA-256 完全一致；
- 临时验证文件已经删除。

因此，对象级和归档级恢复已经验证。系统级恢复仍需在隔离环境中演练，我没有让 Hermes 在当前生产 VPS 上执行破坏性导入。

## 六、我的判断

把任务交给 Agent，不等于把责任也交给 Agent。

对于运行在服务器上的 Hermes，我不需要先学会每个 systemd 字段和 rclone 参数，再逐条替它操作。更有效的方法是：

- 我定义目标、权限和不可触碰的边界；
- Hermes 根据真实环境实施和排错；
- 我用实际下载、哈希和系统状态验收；
- 破坏性步骤保留人工决策；
- 恢复密钥放到 Agent 所在机器之外。

这才是我希望的 Agent 实践：不是让它再写一份操作教程，而是让它完成真实工作，同时留下足够的证据，让人能够判断这项工作是否可靠。

## 参考资料

- [Cloudflare R2：Get started](https://developers.cloudflare.com/r2/get-started/)
- [Cloudflare R2：S3 API 与凭据配置](https://developers.cloudflare.com/r2/get-started/s3/)
- [Cloudflare R2：API Tokens](https://developers.cloudflare.com/r2/api/tokens/)
- [Cloudflare R2：Pricing](https://developers.cloudflare.com/r2/pricing/)
- [Cloudflare R2：Data location](https://developers.cloudflare.com/r2/reference/data-location/)
- [Cloudflare R2：Object lifecycle rules](https://developers.cloudflare.com/r2/buckets/object-lifecycles/)
- [rclone crypt](https://rclone.org/crypt/)
- [rclone cryptcheck](https://rclone.org/commands/rclone_cryptcheck/)
- [Hermes Agent 文档](https://hermes-agent.nousresearch.com/docs/)
