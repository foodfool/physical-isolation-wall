# 04 · Software

[English](#1-principles) · [中文摘要](#中文摘要)

> The software's job is to make the gate pleasant enough that people keep it plugged in. Its security job is only to *propose* — never to decide.

---

## 1. Principles

1. **Nothing on the host is trusted.** `pwgd` is written assuming the process itself may be compromised and that an attacker can read its memory, patch it, and lie to it.
2. **No secrets beyond the request MAC.** If `pwgd`'s keys leak, the attacker gains the ability to *ask*, not to *approve*.
3. **Fail loudly to the human, quietly to the log.** The operator gets an actionable message; the audit log gets a structured record.
4. **Zero configuration by default.** A gate that requires editing sysctl values will be removed from the machine within a week.

---

## 2. Components

```
pwgd/            daemon: IPC, state cache, device I/O, policy hints
pwguard/         CLI and TUI: request, status, log, provision
pwlib/           protocol framing, MAC, state machine mirror
pw-fuse/         optional read-only FUSE view for file-level scopes (experimental)
pw-snapshot/     device-mapper hook that snapshots before an approved write
firmware/        no_std Rust (RP2040) or C (STM32) — see docs/03
```

Language: **Rust**, first-party crates only where possible; `no_std` for firmware so the same frame codec is compiled and unit-tested once and used on both sides. This is deliberate: a protocol implemented twice is a protocol with two bugs.

---

## 3. Daemon behaviour

### 3.1 API surface

| Surface | Purpose | Auth |
|---|---|---|
| Unix socket `/run/pwgd/control.sock` | local applications request writes | peer credentials + per-uid policy |
| D-Bus (`org.piw.Gate1`) | desktop integration, tray notifications | session bus, optional |
| Windows named pipe | parity with the Unix socket | SID checks |
| Read-only `/run/pwgd/status.json` | dashboards, monitoring | world-readable by design |

Requests carry the fields from [docs/02 §2.2](02-architecture.md#22-request-frame-host--gate-64-bytes-fixed). `pwgd` fills in `purpose_code` from a documented enum; a request without a purpose is rejected.

### 3.2 What pwgd does when the gate says no

| Condition | Host-side response |
|---|---|
| `NO_PRESENCE` | Block until the operator reacts, with a notification that says *which device display to look at*, plus the proposal hash |
| `OUT_OF_SCOPE` | Fail the operation and explain the policy that refused it, including the exact LBA range |
| `BUDGET_EXHAUSTED` | Fail with the remaining daily budget, and the date it resets |
| `LOCKED` after a grant | Assume the session was revoked; do not retry silently |

Rule: **never retry a rejected write more than once.** A daemon that loops on refusal trains the operator to approve reflexively, which destroys the human channel.

### 3.3 Interaction budget

Approving must cost the operator between 3 and 10 seconds (see [threat model §6](01-threat-model.md#6-usability-constraints-treated-as-security-requirements)). Implementation consequences:

- Proposals are **coalesced**: if ten writes land within two seconds, the operator sees one proposal covering all ten, with the union of their scopes.
- A single approved session may serve a burst (package install, `git clone`) up to the byte budget, then must be re-approved.
- The gate's display shows: bytes requested, byte budget, the *plain-language* purpose, the expiry countdown, and a short hash. No filenames — the host could put a misleading filename on the screen, and the gate must not depend on host text.

---

## 4. Operating-system side (Linux)

| Layer | Implementation | Honest label |
|---|---|---|
| Mount | Archive mounted `ro,nosuid,nodev`; remounted `rw` only for the session | Tier C |
| Block | `blockdev --setro` on the underlying device when idle | Tier C |
| Device mapper | `pw-snapshot` creates a thin snapshot before the first write of each session | Tier C (but immensely useful for ransomware recovery) |
| Integrity | Optionally `dm-verity` for read paths, `dm-integrity` for write detection | Tier C |
| Auditing | eBPF tracepoint on `block_rq_issue` to attribute writes to a process | Tier C (best-effort attribution; a rootkit can lie) |
| Device authorisation | USBGuard rules that deny unknown storage devices | Tier C |
| Boot | Measured boot where available; report, do not advertise | Tier C |

**None of these are the boundary.** They exist so that the *host* half of the system behaves sanely and so that attribution is possible after the fact.

### 4.1 Snapshot-on-write

The single most useful tier-C feature: before the first write of an approved session, take a thin-provisioned snapshot. If the session goes wrong, the operator can roll back without touching the gate. This converts "approved window abuse" from *unrecoverable* to *annoying* — which materially changes the risk calculus in [R1](01-threat-model.md#5-residual-risk-register).

Caveat: snapshots consume space and are only as good as the volume manager above them. Document the space budget.

### 4.2 Windows

Parity is harder and the project should say so:

- No device-mapper equivalent; snapshots require VSS, which a compromised host can subvert.
- Kernel filtering needs a signed driver, which means a Microsoft-controlled trust chain — a *different* root of trust, and one that is not obviously compatible with this project's threat model.
- v0.1 ships a user-mode implementation only (it can enforce "ask the gate before opening a file for write" at the API level, which is enough for office workflows and useless against kernel-level malware). This limitation goes in the README of the Windows package, in the first sentence.

---

## 5. Testing

| Kind | What it proves | Gate in CI |
|---|---|---|
| Unit tests on the frame codec, shared by host and firmware | That both sides agree on bytes | required |
| Property tests on the state machine (no transition into `WRITING` without `ARMED`) | That the model itself is sound | required |
| Fuzzing on `parse_frame()` (cargo-fuzz / AFL) | That a malformed frame cannot close the switch | required before any tagged release |
| Hardware-in-the-loop: replay, forged MAC, out-of-scope LBA, budget overrun, power cut mid-session | That firmware behaves under attack | nightly, on real hardware |
| Glitch test (crowbar on VDD, short pulses) | That a fault does not arm the switch | best-effort, documented |
| Power-loss matrix: cut at every state | That the resting state is always safe | required |

The HIL rig is the project's most valuable artefact after the protocol. It should be published as a test jig design so others can reproduce the claims.

---

## 6. Packaging

| Target | Artefact |
|---|---|
| Debian/Ubuntu | `.deb` + systemd unit, `udev` rules, AppArmor profile |
| Arch | `PKGBUILD` |
| Fedora | `.rpm` with SELinux policy notes |
| Windows | MSI, user-mode only initially |
| Firmware | UF2 (RP2040) and a signed binary for STM32; reproducible build recipe |

Reproducible firmware builds are a requirement, not a nice-to-have: the entire tier-B claim rests on the user being able to verify that the binary in their device matches the source in the repository.

---

## 7. Code-level rules

1. `unsafe` in host crates: requires a comment explaining the invariant being upheld.
2. Firmware: `#![no_std]`, no allocation in the frame parser, fixed buffers only.
3. Every protocol constant lives in one `protocol.rs`, included by both host and firmware via `include!`, so drift is impossible.
4. Error paths in firmware must not `panic!` — they must open the switch.
5. Any new `msg_type` requires an RFC, a test vector, and a firmware-version gate.

---

## 中文摘要

**原则**：主机侧的一切都不被信任，`pwgd` 是按"自己已被攻陷、攻击者能读它的内存、能改它的代码、能对它撒谎"来写的。它唯一的密钥是请求 MAC——泄了也只换来"能喊话"，换不来"能批准"。默认零配置，一个要改 sysctl 才能用的闸门，一周内就会被拆掉。

**组件**：`pwgd`（守护进程）、`pwguard`（CLI/TUI）、`pwlib`（协议与 MAC）、`pw-fuse`（可选的只读 FUSE 视图，实验性）、`pw-snapshot`（写前快照钩子）、`firmware`。全部用 Rust；固件用 `no_std`，**让同一份帧编解码代码在主机和固件上各编译一次、共用一份单元测试**——协议实现两遍，就等于协议有两个 bug。

**交互预算（重要）**：批准必须花 3–10 秒。所以 ①两秒内的十次写入要**合并成一个提案**（取范围并集）；②一次批准可以覆盖一次突发（装包、`git clone`），直到额度用尽再重新批准；③闸门屏幕上只显示字节数、额度、白话用途、倒计时和短哈希——**不显示文件名**，因为主机能往屏幕上塞误导性文件名，闸门不能依赖主机的文本。

**被拒之后**：`NO_PRESENCE` 就等着，并告诉操作者去看**设备上那块屏**；`OUT_OF_SCOPE` 要说清是哪条策略、哪个 LBA 区间拒绝了它；**任何被拒的写入最多重试一次**——一个对着拒绝循环重试的守护进程，会把操作者训练成条件反射地按确认键，那等于毁掉整个人工通道。

**Linux 侧**：只读挂载、`blockdev --setro`、device-mapper 写前快照、可选的 dm-verity/dm-integrity、eBPF 归因、USBGuard、度量启动——**全部标注为 C 级**。它们存在只是为了让主机侧行为正常、让事后能归因，不是安全边界。

**写前快照是 C 级里最值钱的一条**：第一次批准写入前打一个 thin snapshot，把"批准窗口内的滥用"从**不可逆**变成**很烦**，这直接改变了 R1 的风险算计。代价是占空间，而且只能好到卷管理器那么好。

**Windows**：诚实说明更难——没有 device-mapper、VSS 会被攻陷的主机绕过、内核过滤要签名驱动（等于把信任根换成微软控制的链条，与本项目威胁模型并不天然兼容）。v0.1 只出用户态实现，这句限制要写在 Windows 包 README 的第一句。

**测试**：帧编解码单测、状态机属性测试（"没有 ARMED 就进不了 WRITING"）、`parse_frame()` 模糊测试（**任何带标签的发布之前必须跑过**）、真机 HIL（重放、伪造 MAC、越界 LBA、额度超支、会话中途断电）、故障注入（crowbar 打 VDD）、**逐状态断电矩阵**（证明静止态永远安全）。HIL 台架是这个项目仅次于协议的第二件资产，应该连治具设计一起公开，让别人能复现你的声明。

**打包**：deb/rpm/PKG/Arch、Windows MSI（初期仅用户态）、固件 UF2 与签名二进制。**固件可复现构建是硬要求**——整个 B 级声明都建立在"用户能验证设备里的二进制确实对应仓库里的源码"之上。

**代码规则**：`unsafe` 必须注释不变量；固件 `no_std`、解析器零分配定长缓冲；协议常量只存在于一个 `protocol.rs`，主机与固件用 `include!` 共用；固件错误路径禁止 `panic!`，必须断开关；新增 `msg_type` 需要 RFC + 测试向量 + 固件版本门。
