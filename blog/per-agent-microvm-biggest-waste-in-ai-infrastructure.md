---
title: "Is the Per-Agent microVM the Biggest Waste in AI Infrastructure?"
layout: post
nav_order: 3
parent: Blog
description: "A per-agent microVM idles over 99% of the time waiting for the LLM (P95 1.06–1.18 s) while tool execution needs only ~2 ms. Sandswap time-shares a purpose-built tinyVM across agents at millisecond granularity, cutting sandbox cost 50–100×."
date: 2026-11-17
---

Even for the simplest thought, an LLM response has a P95 of over a second, while reading or writing a 1&nbsp;MB file takes about 2 milliseconds. Giving every AI agent a resident VM or microVM **wastes 99% of those resources**. Have agent sandbox technologies actually solved this pain point?

While the LLM thinks for a second, your microVM sits idle for close to a second; with no session active, it is 100% idle. **Tool execution is the only part that genuinely needs strong isolation**, and it lasts milliseconds. Stuffing an entire agent into a resident microVM and then making it "on-demand" with fast boot, snapshots, freeze, restore, hibernate, and so on is, in essence, putting lipstick on ten- and twenty-year-old abstractions (microVM, VM, container) and stitching patches onto them in the hope of serving a completely new kind of workload: the AI agent.

Below is the detailed analysis. I am also putting Sandswap and tinyVM, which I built from scratch, on the table. Challenges from any angle are welcome.

## Why run AI agents on server technology from ten or twenty years ago (VMs/containers)?

VMs were created more than twenty years ago to partition multi-core servers. Containers came about ten years ago to package applications across their lifecycle, and cloud-native application services grew on top of them.

Neither of these was designed for AI agents.

An agent autonomously generates untrusted code, waits several seconds for LLM inference, executes tools as the LLM instructs, sends the results back so the LLM can decide the next step, and repeats this loop over and over.

LLM responses, even without extended reasoning, measure P95 at 1.06 to 1.18 seconds in tests; reading or writing a 1&nbsp;MB file takes roughly 2 milliseconds. Put both the reasoning loop and tool execution inside a VM or container, and the core contradiction is **strict isolation versus extreme idleness**: your sandbox spins at 100% idle through the LLM's thinking loop, burning expensive compute.

```
one step of the agent loop, roughly to scale:

|------- LLM responds, P95 1.06-1.18 s -------|-- exec ~2 ms --|
               microVM idle                        microVM busy

2 ms of execution in a ~1.1 s step: the sandbox idles over 99% of the time
```

VMs, containers, cloud-native, service meshes... for many years this stack has faced long-running workloads, keeping resources busy most of the time by handling many concurrent business flows inside a single instance, and therefore provisioning capacity by expected concurrency.

That model carried business logic just fine for the past decade or so. But for a single-instance AI agent, the instance is idle when there is no session, and during an active session the resources are still idle while the LLM is inferring. Resources are only truly needed while the agent's main loop and tool execution run. How long is the main loop? **Milliseconds.** Tool execution? **Also milliseconds.** Keeping a VM instance (or a container inside a VM) assigned to a single agent at all times is simply too wasteful.

The AI agent is a new, IO-dominated workload, completely different from every workload that came before.

Yet every sandbox project today, facing this brand-new agent workload, responds the same way. Because agent tool execution runs unsafe, dynamically generated code and needs high-assurance isolation, they conclude the whole agent must go into a microVM, and then they patch the VM/container stack to shave the idle waste: one microVM or pod per agent (Firecracker, Kata, gVisor...), then pedal to the metal on microVM boot time, snapshot generation time, hibernate time, swap-out and restore time. Snapshot, suspend, pause, resume, restore, KSM, CoW... which of these is not a patch sewn onto ten or twenty years of old technology?

Making a VM or container "on-demand" through snapshots or fast boot is, in essence, lipstick on an old abstraction. It looks like idle time has been reclaimed through resource sharing, but do snapshot storage and restore, network bandwidth, and storage space not cost money? The extra cost simply pops out at the other end.

When a company like Meta configures a long-running VM for every user, the cost is far too high. Where is the sustainability in that?

## Extreme resource efficiency: the reasoning loop stays outside, strong isolation on demand

To compress the cost of serving a per-user AI agent to the limit, the stack must be purpose-fit to the agent's load characteristics.

An agent consists of two parts whose trust levels are exact opposites. That is why existing approaches cannot reach the efficiency limit.

The reasoning loop is the main-loop code that calls the model and decides what to do next. It is as trustworthy as any server-side code, and the logical isolation the internet has practiced on servers for a decade or two is security enough. Tool execution is the exact opposite: it is dynamically generated, humans neither need to nor can intervene on every execution (intervening every time would leave no intelligence), and **only this part needs VM-grade strong isolation**. Even then, it only needs it for the few milliseconds it runs.

Logical multi-tenancy is a game internet businesses have been running for decades. In every electronic transaction, e-commerce serves a hundred thousand or a million users on shared application servers, with users logically isolated from each other. Nobody opens a dedicated, strongly isolated VM per shopper; that is a model of extreme resource waste. Tens of thousands of agent reasoning loops are like tens of thousands of shoppers: logical isolation raises business density dramatically, and one server can serve **ten thousand or more AI agents**.

Anthropic describes the same split in Managed Agents: separating the brain from the hands.

That is how the brain's resource efficiency scales up. But what about the hands? How do you isolate only during the milliseconds when strong isolation is actually required? That demands **millisecond-grade, extreme turnover of resources**, which the existing VM/microVM/container architectures and stacks are completely unable to deliver.

So, to answer the demand for millisecond on-demand execution with no wasted resources, against the backdrop of the agent workload and extreme cost efficiency, a brand-new stack was born: Sandswap, which also defines a brand-new kind of VM, the tinyVM.

## Sandswap: tinyVM strong isolation metered in milliseconds, cost down 50–100×

Sandswap treats the strongly isolated tool execution of AI agents as a first-class citizen: a purpose-built tinyVM, with an entirely new stack constructed on top of it, delivering isolation at millisecond granularity.

A compute-and-disk decoupled tinyVM architecture built from scratch for AI agents ([github.com/ChaoyiHuang/sandswap](https://github.com/ChaoyiHuang/sandswap)):

- ⚡ Pure-compute tinyVM: only a custom-hardened guest kernel plus a PID 1 warden. Zero built-in rootfs.
- ⚡ Dynamic SandFS swap: the agent's rootfs directory (SandFS) mounts on demand via virtio-fs allow-dirs (about 2.3 ms cold, about 0.96 ms on the warm path); files and pip/apt installs persist.
- ⚡ Zero legacy baggage: no boot, no image pull, no snapshot/suspend/resume/restore.
- ⚡ Millisecond reclamation: once idle exceeds a configurable threshold (as low as 1 ms), SandFS swaps out asynchronously without blocking the response path (about 3.1 ms), cleaning state for the next agent.
- ⚡ Four layers of strong isolation: a custom-hardened guest kernel, the PID 1 warden, the dedicated sandvmm, and host-side defenses together form four isolation layers, matching or exceeding a traditional VM.

In bare-metal KVM benchmarks, a single tinyVM can be reused up to 183 times per second in the ideal case. Agent reasoning takes seconds while command execution takes milliseconds, so one tinyVM rotates through 50–100 agents per second: **sandbox cost drops by 50–100×**.

Agents execute commands through two interfaces with identical semantics, HTTPS and MCP (zero code changes). Every path is designed for ultra-low latency, and isolation security is never traded for speed. The same architecture extends to serverless scenarios.

See the appendix for the load benchmark report, the benchmark report, and reproduction instructions.

## Does Sandswap survive the hard questions?

### When a tinyVM is reused across AI agents, is the isolation guaranteed?

Sandswap addresses secure isolation from three angles.

Zero residue: **nothing of agent A ever survives onto agent B**. Swap-out is more than an unmount. The warden kills the agent's entire process tree, unmounts every mount, drops caches, and sweeps kernel objects and dead sockets; the host then empties the virtio-fs allowlist. If any step cannot prove cleanliness, that tinyVM is destroyed on the spot and a new one takes its place; it is never reused with doubt lingering. A tinyVM may serve another SandFS only once the previous agent's SandFS is proven to be completely gone.

Sandswap also builds four layers of strong isolation into the tinyVM:

| Layer | Where | Component | What it enforces |
| ----- | ----- | --------- | ---------------- |
| 1 | Guest | Hardened custom guest kernel + PID 1 warden | Every syscall the agent can reach is allowlisted and least-privileged |
| 2 | Hypervisor | sandvmm | sandvmm holds no capabilities; a compromised sandvmm sees only the current SandFS, never another agent's |
| 3 | Tenant binding | Sandlot | One tinyVM holds exactly one SandFS at a time; the SandFS volname is randomized and directory names are invisible inside the guest |
| 4 | Host | sandvmm, sandlot, every process and resource | Role separation with minimal privilege per compromise, so a breach cannot spread; bpf programs secure the transport paths |

Details in the four-layer isolation section: [Sandswap: ultra efficient sandboxes for AI agents, §11](https://chaoyihuang.github.io/posts/blog/sandswap-ultra-high-efficient-sandbox-for-ai-agents.html#11). In practice, under overlay SandFS the base layer is enforced read-only at the host level: the agent can only write to `workspace/` and `/home/agent`, and cannot tamper with anything in the base layer.

And one higher degree of isolation still: the single-use tinyVM. A tinyVM contains only the kernel and the PID 1 warden, so a fresh one can be booted into the pool extremely fast. A tinyVM can then serve just a single SandFS: as soon as it is swapped out, the tinyVM is destroyed. Dropping the recycle policy's reuse count to one agent buys very high-assurance isolation security. The author leaves this capability to vendors as a differentiator.

The isolation Sandswap delivers is no weaker than what current VMs provide; if anything, it holds itself to a stricter bar.

Attempts at cross-agent isolation attacks from any angle, and vulnerability reports, are welcome.

### Switching tinyVMs on every command: isn't that slow, especially for Python, TypeScript, and other scripting languages?

First, correct the premise. Consecutive commands under the same agent SandFS **reuse the still-attached tinyVM within a sticky window** (100 ms by default, configurable down to 1 ms), and that warm path costs about 0.96 ms.

Sandswap also runs longer tasks in background mode: background execution returns an exec_id immediately, the tinyVM is held for exactly as long as the task runs, and a command can request how long it needs the tinyVM. Usage on demand.

Now dig one level deeper: why use interpreted languages that are slow to start and slow to execute in the first place? In the past, software engineers wrote the code, so interpreted languages, which are programming for humans with a gentle learning curve, were popular. Now the code inside the sandbox is written by the AI agent. Interpreted or compiled-and-fast: to an AI agent they are the same, so the performance-versus-language tradeoff is something the agent is fully smart enough to weigh for itself.

How quickly each command's process comes up depends on the command's own startup speed. Server-side applications serving many concurrent business flows tend to initialize large amounts of memory; a MySQL serving a million users wants a large memory cache, so initialization and startup are slow. But when an agent executes a task, it faces an individual job. Porting the old server-side application model over may itself be nothing more than a wrong habit of traditional thinking.

Start from the nature of the work: what genuinely needs to run long can run in the background, and short tasks execute as synchronous commands. Trust that the agent has enough intelligence. Sandswap also bakes timing metrics into every command execution response, so the agent can optimize its own behavior.

### With heavy file IO, doesn't all the pressure move onto the storage system?

In a multi-node distributed system this really comes down to storage capability. Sandswap itself does not restrict which storage system the backend uses; there could even be storage systems purpose-optimized for systems like Sandswap. A SandFS corresponds to a directory. How that directory is constructed, whether on a distributed filesystem or on a filesystem over block devices, is open as long as the integration requirements are met.

Every filesystem on every virtual machine in the public cloud today is in fact built on top of distributed storage. Today's internet businesses overwhelmingly run on public-cloud VMs, and in practice nobody challenges that storage-system IO is a problem. On many public clouds storage pressure routinely sits within 10%. Storage systems have far more headroom than we imagine, and it is safe to trust that storage providers have the wisdom and the capability to meet latency and bandwidth demands.

### Can Sandswap support large-scale data analysis, computer use, and browser use?

There is a common illusion that because Sandswap works through command execution, it can only run short-lived things such as shell commands. In reality, Sandswap provides background mode precisely to meet long-running tasks.

The model's own context window is finite, and whatever flows back to the model is constrained by that window. So no matter how large a data-analysis task is, whether traditional rule-programmed software does more of the work, or the AI agent does, or the agent keeps splitting it into subtasks through just-in-time programming, each load-and-analyze pass over very large data comes down to the programming model: how to balance persistent storage against in-memory data, which loses everything on restart but performs well.

Sandswap doing this kind of work is not in question. Sandswap's first design principle is that everything an AI agent does always gets persistency: no matter how a failure happens, it can always recover or roll back and move forward from the breakpoint.

Take browser use. If the agent wants to navigate forward and backward the traditional way, with the history living only in the browser's memory, the browser runs in the background and the agent interacts with the running browser through a synchronous command.

### Does Sandswap support traditional server-type applications?

Sandswap provides egress network traffic only and does not support ingress. This follows from the AI agent's perspective, and from security considerations.

Opening a server port on a tinyVM to accept external access is not Sandswap's business model. Sandswap does not position itself as a replacement for the existing VM/container world of server and microservice applications.

Sandswap is an ultra efficient sandbox built for AI agents, focused on extreme resource efficiency and a 50–100× reduction in AI agent execution cost.

### Inside Sandswap, who manages credentials and egress policy?

Sandswap hands egress policy control to third parties, but on the host side, outside the tinyVM, it provides a tamper-proof identity for traffic, with two ways to mark which SandFS a flow belongs to. One embeds the SandFS identifier directly into the source IPv6 address, with the Agent ID and SandFS ID packed into the IPv6 address itself, so egress policy control can be done in a way fully decoupled from Sandswap. The other places the Agent ID and SandFS ID in shared memory or a bpf map, reachable directly through the ifindex, so the identity of a flow can be fetched in nanoseconds to microseconds, which suits identifying traffic and enforcing network policy in kernel or user space on the same machine.

<section class="appendix">
<h2>Appendix</h2>
<ul class="refs">
  <li>Code and docs: <a href="https://github.com/ChaoyiHuang/sandswap">github.com/ChaoyiHuang/sandswap</a></li>
  <li>Bare-metal benchmark: <a href="https://chaoyihuang.github.io/sandswap/benchmark/baremetal.html">chaoyihuang.github.io/sandswap/benchmark/baremetal.html</a></li>
  <li>Bare-metal load benchmark: <a href="https://chaoyihuang.github.io/sandswap/benchmark/baremetal-load.html">chaoyihuang.github.io/sandswap/benchmark/baremetal-load.html</a></li>
  <li>Claude Managed Agents: get to production 10x faster: <a href="https://claude.com/resources/articles/claude-managed-agents">claude.com/resources/articles/claude-managed-agents</a></li>
</ul>
<p class="disclaimer">(All views in this post are the author's own.)</p>
</section>
