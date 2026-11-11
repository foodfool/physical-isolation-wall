# 05 · Roadmap

[English](#milestones) · [中文摘要](#中文摘要)

> Dates are deliberately absent. A security project that ships to a deadline ships a lie. What matters is the **acceptance criteria** — what must be true before the next stage is allowed to exist.

---

## Milestones

### P0 — Argument (now)

**Goal:** the design survives contact with hostile reviewers.

| Deliverable | Acceptance |
|---|---|
| Threat model | Every protection claim mapped to tier A or B; every tier-C element labelled as such |
| Protocol spec | Frame layouts fixed; state machine exhaustive over its inputs; no transition into `WRITING` without `ARMED` |
| This repository | Bilingual README; docs 01–05 complete |
| External review | At least 3 people who are not the author have read the threat model and left objections |

Exit criterion: an issue tracker containing real objections, not compliments.

---

### P1 — One working gate (M0)

**Goal:** a physical thing on a desk that refuses a write, and can prove it refused.

| Deliverable | Acceptance |
|---|---|
| M0 board (breadboard or carrier) | Default state on power-up is disconnected — *measured*, not assumed |
| Firmware v0.1 | Fixed buffering, no `panic!` on error paths, watchdog owned by MCU |
| Presence | Button + NFC challenge-response; UID cloning does not pass |
| Log | Hash-chained, append-only, survives 50 power cuts mid-record without losing the last good record |
| Demo | A script that tries to write 10 times; nine are refused; the tenth, after physical approval, succeeds and is logged |
| HIL log | Published as `docs/test-results/p1-*.md`, including the failures |

Exit criterion: someone who is not the author can build it from the BOM and reproduce the demo.

---

### P2 — Read-open, write-gated (M1)

**Goal:** a usable archive drive that stays readable while writes require a hand.

| Deliverable | Acceptance |
|---|---|
| SBC bridge | Host sees a mass-storage device; archive readable continuously without any presence event |
| Direction cut | Write data physically cannot reach the drive when the MCU holds the enable line low — verified by pulling the enable line and attempting a raw write |
| K1 key custody | Archive encrypted with a gate-held KEK; a stolen powered-off machine yields no readable data (demonstrated, not asserted) |
| Snapshot hook | Write session restores cleanly from snapshot after a deliberate corruption test |
| Rate policy | Daily budget, sessions-per-day, TTL cap all enforced in firmware counters, not in `pwgd` |

Exit criterion: an outside reviewer can attempt the [attack tree](../docs/01-threat-model.md#4-attack-tree-against-the-gate) and fails at every branch that the threat model claims is closed.

---

### P3 — Something other people can build (M2)

**Goal:** a PCB and a process, so the design stops depending on one person's bench.

| Deliverable | Acceptance |
|---|---|
| 4-layer reference board | Open EDA files, fabrication outputs, BOM with two sources for each critical part |
| Reproducible firmware build | A third party builds the same binary bit-for-bit from tagged source |
| Test jig | Published design; HIL suite runs unattended |
| Independent audit | At least one external party reviews the protocol and firmware, findings public |

---

## RFC index

| RFC | Title | Status |
|---|---|---|
| 0001 | Physical write gate: problem statement and non-goals | **draft — this repository** |
| 0002 | One-way request protocol and the no-return-channel problem | open for adoption |
| 0003 | K1 key custody: secure element selection and zeroisation policy | open for adoption |
| 0004 | Direction-selective switching for SATA at 6 Gbps | open for adoption |
| 0005 | Duress mode and plausible deniability | open, contentious |
| 0006 | Log anchoring without write-once media | open |
| 0007 | Windows parity and the signed-driver trust problem | open |

Process: an RFC is a file in `rfcs/`, merged when it has one approving reviewer who is not the author. Objections are recorded, not deleted.

---

## Open research questions

These are the parts where the answer might be "this does not work". Naming them is the honest move.

1. **Fault injection on commodity MCUs.** A €1 microcontroller can be glitched. Do brown-out detection plus redundant pin assertion actually help, or is the phrase "physical" overclaiming once an attacker has a probe and a pulse generator? An honest answer may be: *the gate resists software, not hardware.*
2. **Read-open, write-gated is genuinely better?** Or does leaving reads open mean the interesting attacks (exfiltration) never needed the write path at all? If so, the project should say that its main value is **integrity**, and stop implying confidentiality.
3. **The human channel degrades.** Every study of confirmation fatigue says the same thing: people approve faster over time. Does a 3–10 s cost survive month 6? Needs a longitudinal study, not an opinion.
4. **Does the audit log matter if nobody reads it?** Log design assumes an operator who reviews anchors. Most people will not. Maybe the correct output is a monthly signed postcard, not a log file.
5. **Bridge architecture vs. pass-through.** If the bridge is a general-purpose Linux SoC, its attack surface is a Linux SoC. Is tier-B really acceptable as the main path, or does the project have to hold out for M2?

---

## How to claim work

1. Open an issue with the prefix `RFC:` or `TASK:` describing what you will do and what "done" means.
2. If it changes the protocol, write the RFC first. A patch without an RFC will be asked for one.
3. Hardware claims require measurements in the PR description: oscilloscope captures, eye diagrams, or a stated uncertainty.
4. Security claims require a failing test. A fix without a test that fails before it is not a fix.

---

## Definition of done for a security claim

This repository uses one rule everywhere:

> **A protection claim is allowed only when (a) it is implemented, (b) it is tested by an attempt to defeat it, and (c) the test is published.**

Anything else is a design intention and must be written in the future tense.

---

## 中文摘要

**没有时间表，只有验收标准**——一个按截止日期交付的安全项目，交付的是一句谎话。

- **P0 论证阶段（现在）**：威胁模型里每条防护声明都要能映射到 A 级或 B 级，每个 C 级要素都必须标明；协议帧格式固定、状态机对输入穷举；**至少 3 个非作者的人读过威胁模型并留下反对意见**才算完成。退出标准是：issue 区里装的是真反对，不是恭维。
- **P1 第一台能工作的闸门（M0）**：上电默认断开要**实测**而不是假设；固件定长缓冲、错误路径不 panic、看门狗归 MCU；按键 + NFC 挑战应答（克隆 UID 过不了）；哈希链日志在 50 次记录中途断电后仍不丢失最后一条好记录；演示脚本连写 10 次、9 次被拒、第 10 次物理批准后成功并留痕；**测试日志公开，包括失败**。退出标准：非作者能照 BOM 复现。
- **P2 读常开、写受控（M1）**：主机看到的是大容量存储设备，归档盘无需任何在场事件即可持续读取；把使能线拉低后直接裸写**物理上到不了盘**（要实测验证）；K1 密钥托管下，偷走一台关机的机器拿不到任何可读数据（要演示，不是声称）；故意破坏后能从快照干净恢复；额度、每日会话数、TTL 上限由**固件计数器**而非 `pwgd` 执行。退出标准：外部评审者照攻击树逐个试，威胁模型声称关闭的每一条分支都必须攻不进去。
- **P3 别人也能造（M2）**：4 层参考板（开放 EDA 文件、制造输出、关键器件双供应商 BOM）；可复现固件构建（第三方从标签源码构建出逐位相同的二进制）；公开的测试治具与无人值守 HIL；至少一家外部机构评审协议与固件且结论公开。

**RFC 清单** 0001 本项目 → 0002 单向协议与"没有回程"的难题 → 0003 K1 密钥托管与零化策略 → 0004 6 Gbps SATA 的方向切换 → 0005 胁迫模式（有争议）→ 0006 无一次性介质的日志锚定 → 0007 Windows 与签名驱动的信任问题。RFC 需要一位非作者的赞成评审才合并，反对意见记录不删除。

**五个可能得出"这行不通"的研究问题**：①一元级 MCU 的抗故障注入到底有没有用，也许诚实的结论是"它防软件，不防硬件"；②"读常开"到底是不是好事——如果真正危险的攻击（外传）根本不需要写通路，那本项目的主要价值其实是**完整性**，就不该再暗示机密性；③**人工通道会退化**——所有关于确认疲劳的研究都指向同一件事，3–10 秒的成本能不能撑到第 6 个月？这需要纵向研究而不是观点；④如果没人看日志，日志还有什么用？也许正确的产物是一张每月寄出的签名明信片，而不是一个日志文件；⑤桥接架构 vs 直通——如果桥接是一台通用 Linux SoC，它的攻击面就是一台 Linux SoC，B 级真的能当主路径吗？

**认领规则**：issue 用 `RFC:` 或 `TASK:` 前缀，说清做什么、什么算完成；改协议先写 RFC；硬件声明必须附示波器截图、眼图或明确的不确定度；安全声明必须附一个"修之前会失败"的测试。

**安全声明的完成定义**：①已实现 ②被一次针对它的击败尝试测试过 ③测试公开。三者缺一，只能写成将来时。
