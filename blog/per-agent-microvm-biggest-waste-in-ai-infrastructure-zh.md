---
title: "为每个 AI Agent 配一台专属 microVM，是不是 AI 基础设施最大的浪费？"
layout: post
lang: zh-CN
nav_order: 4
parent: Blog
description: "为每个 AI Agent 配一台常驻 microVM，99% 的时间在等 LLM（P95 1.06–1.18 秒），而工具执行只需约 2 毫秒。Sandswap 以毫秒粒度在 Agent 间时分复用专用 tinyVM，沙箱成本降低 50–100 倍。"
date: 2026-11-17
---

即使最简单的思考，LLM 响应的 P95 都要 1 秒多，而读写 1MB 文件只要约 2 毫秒。为每个 AI Agent 配一台常驻 VM/microVM，**资源浪费 99%**。Agent 沙箱技术都解决这个痛点了吗？

LLM 思考 1 秒，你的 microVM 就空转近 1 秒；没有会话的时候，它还是 100% 闲置。**工具执行才是唯一真正需要强隔离的部分**，而且是毫秒级的。把整个 AI Agent 塞进一台常驻 microVM，再靠快速启动、快照、冻结、恢复、休眠等技术来实现"按需"——本质上是在给 10 年 20 年的旧技术抽象（microVM/虚拟机/容器）涂口红，缝缝补补想解决全新类型的 AI Agent 工作负载。

下面详细分析，也把我从头做的 Sandswap 和 tinyVM 拿出来，欢迎从各种角度来讨论。

## 为什么用十几二十年前的服务器技术（虚拟机/容器）跑 AI Agent？

虚拟机是 20 多年前为了切分多核服务器的；容器是 10 年前为了应用生命周期打包封装，然后是云原生应用服务。

这两样东西，没有一个是给 AI Agent 设计的。

AI Agent 会自主生成不可信代码，等 LLM 推理要等好几秒，然后根据 LLM 的指示执行工具，再送到 LLM 决定下一步，如此循环往复。

LLM 的响应，即使不带推理，测试显示 P95 也要 1.06 到 1.18 秒；而读写 1MB 文件，也就大约 2 毫秒。把推理循环和工具执行都塞进 VM/容器，核心矛盾就是：**严格隔离 vs 极致闲置**——你的沙箱在 LLM 思考循环期间 100% 空转，烧着昂贵的算力。

```
one step of the agent loop, roughly to scale:

|------- LLM responds, P95 1.06-1.18 s -------|-- exec ~2 ms --|
               microVM idle                        microVM busy

2 ms of execution in a ~1.1 s step: the sandbox idles over 99% of the time
```

虚拟机、容器、云原生、服务网格……长久以来，这套技术栈都是面向长驻负载、通过单实例多并发业务流处理来保障资源大部分时间忙碌，因此按照并发规模来预留资源上限。

这种业务逻辑以往十几年当然都跑得通，但是面向单实例的 AI Agent，没有会话的时候实例是空闲的，有活跃会话时，LLM 推理期间资源仍然是空闲的。实际上只有 AI Agent 主循环和工具执行的时候，才需要资源运行。主循环多久？**毫秒计**。工具执行多久？**也是毫秒计**。把一个虚拟机实例（或者跑在虚拟机里面的容器），无论何时都配置给单实例 AI Agent，资源实在太浪费了。

AI Agent 是典型的 IO 为主的新型工作负载，和以往任何一种工作负载都完全不同。

但是，现在所有的沙箱项目，针对 AI Agent 这种全新的工作负载，因为 AI Agent 的工具执行面临不安全的动态生成代码执行、需要高级别安全隔离，并没有考虑工作负载的特征需要全新的技术来应对，就把整个 AI Agent 都塞进 microVM，然后在虚拟机/容器的技术栈上缝缝补补来减少闲置浪费：每 Agent 一台 microVM/pod（Firecracker、Kata、gVisor……），然后使劲加速 microVM 的启动时间，加速快照生成时间，加速休眠时间，加速换出和恢复时间，snapshot/suspend/pause/resume/restore/ksm/cow……哪一项技术不是在十几二十年上缝缝补补过来的？

现有的沙箱技术，想让 VM 或容器通过快照或快速启动变得"按需"，本质上只是给旧抽象涂口红。好像空闲的时间被资源复用起来了，但是快照存储/恢复、网络带宽和存储空间的消耗不是费用？额外的成本只不过在另外一端又冒出来了。

像 Meta 这样，给每个用户配置长期运行的 VM，成本实在太高了，可持续性在哪里？

## 极致的资源效率：推理循环外置，按需强隔离

要把为单用户服务的 AI Agent 成本压缩到极致，就必须针对 AI Agent 的负载特征提供极端适配的技术栈。

AI Agent 由两部分组成，信任等级完全相反。这是现有做法为什么不够高效极致的原因。

推理循环，就是调模型、决定下一步做什么的那段主循环代码，它和任何服务端代码一样可信，互联网十几年的服务器端逻辑隔离，其安全性就足够。而工具执行则完全相反：动态生成，人类无须也无法每次执行都介入（每次都介入就没有智能了），**只有这一块才需要虚拟机级别的强隔离**，而且只在运行的那几毫秒需要。

逻辑多租的玩法，互联网业务已经跑了几十年。每一次电子业务交易，电商用共享的应用服务器服务十万上百万用户，用户之间逻辑隔离。没人给每个购物者开一台专用强隔离 VM，那是极端资源浪费的模型。数万个 Agent 推理循环和数万个购物者一样，都可以通过逻辑隔离大幅度提高业务密度，单服务器服务**1 万甚至更多的 AI Agent**。

Anthropic 在 Managed Agents 里写的是同一个拆法，大脑和手分离。

大脑部分的资源效率可以这样提升，那工具执行呢？如何做到只在毫秒级需要强隔离的时候才进行隔离？这要求**资源的毫秒级极致周转效率**，现有的 VM/microVM/容器技术架构和堆栈，完全无法做到。

因此，针对 AI Agent 毫秒级按需执行、不浪费额外资源的诉求，在 AI Agent 的负载特征以及极致成本效率的背景下，诞生了全新的技术栈 Sandswap，并定义了一种全新的 VM：tinyVM。

## Sandswap：以毫秒计的 tinyVM 强隔离，成本降 50-100 倍

Sandswap 把 AI Agent 的强隔离工具执行视为一等公民，完全为其打造专用的 tinyVM，在 tinyVM 上构建全新技术栈，实现毫秒级别的隔离技术。

一套从零开始、为 AI Agent 打造的计算与磁盘解耦的 tinyVM 架构（[github.com/ChaoyiHuang/sandswap](https://github.com/ChaoyiHuang/sandswap)）：

- ⚡ 纯计算 tinyVM：只有定制加固的 guest kernel + PID 1 warden。零内置 rootfs。
- ⚡ 动态 SandFS swap：Agent 的 rootfs 目录（SandFS）通过 virtio-fs allow-dirs 按需挂载（冷路径约 2.3ms，热路径约 0.96ms），文件和 pip/apt 安装持久保留。
- ⚡ 零历史包袱：没有 boot，没有 image pull，没有 snapshot/suspend/resume/restore。
- ⚡ 毫秒级回收：空闲超过可配置阈值（最低 1ms）后，SandFS 异步换出、不阻塞响应路径（约 3.1ms），为下一个 Agent 清理状态。
- ⚡ 四层强隔离：定制加固的 guest kernel、PID 1 warden、专用 sandvmm、主机侧防御，共同实现四层隔离——匹配甚至超过传统 VM。

在裸金属 KVM 基准测试中，单台 tinyVM 理想情况下每秒可复用高达 183 次。Agent 推理要几秒，而命令执行只要毫秒，所以一台 tinyVM 每秒轮转 50-100 个 Agent——**沙箱成本降低 50-100 倍**。

Agent 通过两个接口执行命令，语义完全一致——HTTPS 和 MCP（零代码改动）。每一条路径都为超低延迟设计，隔离安全从不为了速度妥协。同一架构也可延伸到 serverless 场景。

参考附录的负载基准报告、基准报告和复现说明。

## Sandswap 经得起灵魂拷问吗

### tinyVM 跨 AI Agent 复用，安全隔离有保障吗？

Sandswap 从三个方面解决安全隔离。

零残留：**A 的东西绝不会活到 B 身上**。换出不只是卸载：warden 杀掉 Agent 的整个进程树，卸掉所有挂载，清缓存，扫掉内核对象和死掉的 socket；宿主机再清空 virtio-fs 白名单。任何一步证明不了干净，这台 tinyVM 直接销毁、换一台新的，绝不带着疑问复用。一台 tinyVM 可以被另一个 SandFS 使用，前提是上一个 Agent 的 SandFS 被证明已经彻底消失。

Sandswap 给 tinyVM 做了四层强隔离：

| 层 | 位置 | 组件 | 保障什么 |
| - | ---- | ---- | -------- |
| 1 | Guest | 定制加固的 guest kernel + PID 1 warden | Agent 能用到的 syscall 全部白名单控制、最小权限化 |
| 2 | Hypervisor | sandvmm | sandvmm 无任何 capability；被攻破也只能看到当前 SandFS，看不到其他 AI Agent 的 |
| 3 | 租户绑定 | Sandlot | 一台 tinyVM 同一时刻只挂一个 SandFS；SandFS volname 随机化，目录名在 guest 内不可见 |
| 4 | 主机侧 | sandvmm、sandlot 及所有进程与资源 | 角色分离，单点攻破只能获得最小权限、风险不扩散；bpf 程序保障传输路径安全 |

详细可参考四层隔离一节：[Sandswap：面向 AI Agent 的超高效沙箱，第 11 页](https://chaoyihuang.github.io/posts/blog/sandswap-ultra-high-efficient-sandbox-for-ai-agents.html#11)。实际上在 overlay SandFS 下，base layer 在宿主机层面确保只读，Agent 只能写 `workspace/` 和 `/home/agent`，篡改不了 base layer 的任何东西。

还有一个更高意义上的安全隔离：只用一次的 tinyVM。tinyVM 只包含 kernel 和 PID 1 warden，因此可以急速启动一台新的 tinyVM 放进 pool。这样 tinyVM 甚至可以只服务单个 SandFS：一旦被 swap-out，就销毁这台 tinyVM。把 recycle policy 的复用次数降到 1 个 AI Agent，就确保了极高强度的隔离安全。作者把这个功能留给厂商去做差异化竞争力。

Sandswap 的安全隔离程度不低于当前的 VM，甚至要求更严。

欢迎从各种角度尝试跨 Agent 安全隔离攻击并上报漏洞。

### 每条命令都切换 tinyVM，是不是会很慢，特别是 python、typescript 等脚本语言？

先纠正前提：同一个 Agent SandFS 的连续命令，**在粘性窗口内复用还挂着的 tinyVM**（默认 100 毫秒，最低可以 1 毫秒），这一 warm 路径约 0.96 毫秒。

Sandswap 支持较长任务用 background 方式运行：后台执行立刻返回 exec_id，任务跑多久 tinyVM 就占多久，命令可以请求占用 tinyVM 多长时间，按需使用。

再深挖一层：为什么要用启动和执行都慢的解释性语言？过去，代码是软件工程师写的，解释性语言是面向人的编程，学习曲线平缓的解释性语言很受欢迎；现在沙箱里的代码是 AI Agent 写的，解释性语言还是执行速度高的编译型语言，对 AI Agent 来说都一样，性能和语言的取舍，完全可以由 AI Agent 自己智能权衡。

每次执行命令拉起一个进程的快慢，取决于命令本身的启动速度。过去为了服务端应用承载多业务并发流，往往会初始化大量内存，比如 MySQL 服务 100 万用户，内存 cache 就希望大一些，初始化启动就慢。但 AI Agent 执行任务时，面对的是个体作业，把原来服务器端应用的模式移植过来，本身是否就是一种错误的传统习惯思维？

从业务本质出发：确实需要长时间运行的，用 background 方式跑；较短的任务，用同步命令执行。要相信 AI Agent 有足够的智能，而且 Sandswap 每个命令执行的响应内置了时间延迟指标，AI Agent 可以自己优化自己的行为。

### 大量文件 IO，是否全部转移到存储系统的压力了？

在多节点分布式系统下，这其实涉及存储系统的能力。Sandswap 本身并不限制后端用什么存储系统，甚至可以出现专为 Sandswap 这样的系统优化的存储系统。SandFS 实际上对应着一个目录，至于这个目录如何构建——是分布式文件系统，还是块设备上面的文件系统——只要满足集成要求，都可以。

当前公有云上所有虚拟机的文件系统，实际上都构建在分布式存储系统之上。当今互联网业务基本都构建在公有云虚拟机上，实际上也没有谁挑战说存储系统的 IO 有问题。很多公有云的存储压力经常只在 10% 以内，存储系统比我们想象的有更大的能力冗余，要相信存储供应者有足够的智慧和能力解决时延和带宽的诉求。

### Sandswap 能支持大规模数据分析、computer use 和 browser use 吗？

常常会有一个错觉，认为 Sandswap 通过命令执行的方式，好像只能完成短暂命令比如 shell 命令的运行。实际上 Sandswap 提供 background 模式，正是为了满足长任务执行的需要。

大模型本身的上下文窗口是有限的，回流到大模型的内容受到上下文窗口的约束。所以，不管多大的数据分析任务，无论是传统规则编程软件做得多一点，还是 AI Agent 做得多一点，或者 AI Agent 通过即时编程不断拆分子任务，每次超大规模数据的加载与分析，最后都落在编程模式上：如何在持久化存储和内存数据之间权衡——内存快，但重启即丢。

Sandswap 完成这样的工作肯定没有问题。Sandswap 的第一条设计原则就是 AI Agent 的一切工作总是做 persistency：不管如何失败，总能恢复或者回退，在断点上继续前行。

比如说 browser use，AI Agent 如果想以传统的方式进行页面的前进后退，且前进后退只在 browser 的内存里，则 browser 可以在 background 运行，通过一个同步命令和正在运行的 browser 交互即可。

### Sandswap 支持原来的 Server 类型应用吗？

Sandswap 只提供 egress 出网络流量，不支持 ingress 入网络流量，这主要是从 AI Agent 出发，也是从安全角度考虑。

在 tinyVM 上开服务器端口、接受外部访问，不是 Sandswap 的业务模式。Sandswap 并不定位在替换所有已存在的虚拟机/容器的服务器/微服务应用模式。

Sandswap 是专为 AI Agent 打造的 ultra efficient sandbox，专注于极致的资源效率和 50~100 倍的 AI Agent 执行成本降低。

### 在 Sandswap 里，凭据和出网策略谁管？

Sandswap 把 egress 的 policy 控制交由第三方完成，但 Sandswap 在 tinyVM 外的主机侧提供不可篡改的流量 identity，用两种方式标识流量属于哪一个 SandFS。一种是把 SandFS 的标识直接嵌入源 IPv6 地址，在 IPv6 地址中直接包含 Agent ID 和 SandFS ID，这样可以用和 Sandswap 完全解耦的方式做 egress 流量的策略控制；另一种是把 Agent ID 和 SandFS ID 放在共享内存或 bpf 的 MAP 里，直接通过 ifindex，以纳秒微秒计获得流量的标识信息，适合在同一台机器的内核态或用户态做流量识别和网络策略控制。

<section class="appendix">
<h2>附录</h2>
<ul class="refs">
  <li>代码和文档：<a href="https://github.com/ChaoyiHuang/sandswap">github.com/ChaoyiHuang/sandswap</a></li>
  <li>裸金属 benchmark：<a href="https://chaoyihuang.github.io/sandswap/benchmark/baremetal.html">chaoyihuang.github.io/sandswap/benchmark/baremetal.html</a></li>
  <li>裸金属下负载 benchmark：<a href="https://chaoyihuang.github.io/sandswap/benchmark/baremetal-load.html">chaoyihuang.github.io/sandswap/benchmark/baremetal-load.html</a></li>
  <li>Claude Managed Agents: get to production 10x faster：<a href="https://claude.com/resources/articles/claude-managed-agents">claude.com/resources/articles/claude-managed-agents</a></li>
</ul>
<p class="disclaimer">（文中观点均为作者个人观点）</p>
</section>
