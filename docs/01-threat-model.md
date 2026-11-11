# 01 · Threat Model

[English](#1-assets) · [中文摘要](#中文摘要)

> This document is the contract between the project and its users. Everything the gate claims later must be traceable to a row in this table.

---

## 1. Assets

| ID | Asset | Why it matters | Loss impact |
|---|---|---|---|
| A1 | The archive data at rest (photos, tax records, source code, family documents, research notebooks) | Cannot be regenerated; often the only copy | Irreversible |
| A2 | The integrity of that data (silent corruption) | A wrong record is worse than a missing one | Severe, delayed |
| A3 | The audit record of who wrote what, when | Needed to know whether a compromise happened | Severe |
| A4 | The gate's own provisioning secrets (scope policy, per-card keys, byte budgets) | Compromise converts the gate into a switch | Total |
| A5 | The availability of the host machine | The gate must not brick the workflow | Medium |

Explicitly **not** an asset here: the confidentiality of data while the host is reading it. Read paths are open by design (see §7).

---

## 2. Adversaries

| ID | Adversary | Capability assumed | Motivation |
|---|---|---|---|
| **AD1** | Autonomous software agent / malware with host root | Full read/write to RAM, disk, devices; can reflash anything the host can reach; can call any syscall; can render convincing UI; can wait | Mass encryption (ransom), exfiltration, sabotage |
| **AD2** | Supply-chain attacker inside a benign-looking application | Code execution inside the user's session, no kernel privileges at first | Same, plus persistence |
| **AD3** | Remote attacker via network/service | Whatever a compromised service grants; may reach AD1 over time | Ransom, espionage |
| **AD4** | Careless or tired human, buggy script | Legitimate authority, wrong target | Destructive accident |
| **AD5** | Physical thief (machine powered off) | Can remove drives, boot from USB, read raw NAND | Data theft |
| **AD6** | Coercing human ("open it or else") | Physical access to the operator | Forced disclosure |
| **AD7** | Malicious insider at the gate (someone who can reflash gate firmware) | Full control of the gate | Anything |

**Not modelled:** nation-state hardware interdiction at the fab, TEMPEST-grade side-channel collection, or an adversary who can rewrite the gate's silicon. These are listed as residual risk, not silently assumed away.

---

## 3. Trust boundaries

```
  ┌─────────────────────────── untrusted ───────────────────────────┐
  │  Application → Kernel → pwgd → USB/UART link                    │
  └────────────────────────────────┬────────────────────────────────┘
                                   │  one-way request frames only
  ┌────────────────────────────────▼────────────────────────────────┐
  │  trusted-for-integrity: gate MCU firmware + switch + log store  │
  │  (trust earned by open design, reproducible build, review)      │
  └────────────────────────────────┬────────────────────────────────┘
                                   │  GPIO / electrical
  ┌────────────────────────────────▼────────────────────────────────┐
  │  trusted-for-confidentiality: the human hand and the NFC card   │
  └─────────────────────────────────────────────────────────────────┘
```

Three claims the project makes about this picture, and the assumption each rests on:

1. **The host cannot authorise.** Assumption: the request link is physically one-way, or at minimum the gate's authorisation state lives in registers the host bus cannot address.
2. **The gate cannot be silently rewritten during a session.** Assumption: firmware update requires the gate to be disarmed by a physical action; boot is from write-protected storage.
3. **The human sees the truth before consenting.** Assumption: the gate displays its own proposal, not a host-rendered one. If the display is driven by host data, the entire scheme collapses — this is the single most fragile point in the design.

---

## 4. Attack tree against the gate

Root goal: **write to A1 without a physical confirmation by the legitimate operator.**

```
WRITE WITHOUT CONSENT
├── 1. Forge a grant
│   ├── 1.1 Inject a valid session frame (needs nonce + MAC key)
│   │        → mitigated: MAC key lives in MCU OTP; frames are one-shot
│   ├── 1.2 Replay a previously approved frame
│   │        → mitigated: monotonic counter in MCU, nonce window, TTL
│   └── 1.3 Overrun the byte budget after grant
│            → mitigated: bridge drops out-of-scope block commands; budget counted in gate
├── 2. Assert the switch without a grant
│   ├── 2.1 Write a host-visible register that controls the switch
│   │        → mitigated: switch is driven by MCU GPIO only; no host-addressable path
│   ├── 2.2 Glitch the MCU into asserting the pin
│   │        → partially mitigated: brown-out detector, pin state re-asserted every cycle; open research
│   └── 2.3 Power-cycle the gate until it comes up in an "open" state
│            → mitigated: default state on reset is open-circuit; needs HW verification
├── 3. Make the operator approve something else
│   ├── 3.1 Render a fake proposal on the host screen
│   │        → mitigated: gate has its own display; host UI is irrelevant
│   ├── 3.2 Swap the proposal after display, before press (TOCTOU)
│   │        → mitigated: the displayed hash is bound into the grant; gate re-checks at close
│   └── 3.3 Approve a small benign write, then substitute the payload
│            → mitigated: scope + byte budget + post-write hash verification
├── 4. Suppress the audit
│   ├── 4.1 Delete host-side logs
│   │        → accepted loss; host logs are advisory only
│   └── 4.2 Corrupt the gate log
│            → mitigated: hash chain + append-only store + optional signed anchor export
└── 5. Bypass by removing the gate
    ├── 5.1 Pull the drive and attach it directly
    │        → mitigated: self-encrypting drive (SED) or encrypted volume whose key lives in the gate
    └── 5.2 Reflash the gate
             → mitigated: signed firmware, physical disarm required, tamper switch wipes keys
```

Row 5.1 is the one that decides whether this project is real. **If the data on the drive is readable when the drive is removed, the gate protects nothing against a determined attacker with ten minutes and a screwdriver.** Encryption key custody inside the gate is therefore not optional — it is the actual root of the scheme. This is why [docs/02](02-architecture.md) treats key custodian mode as the primary configuration and "clear-text drive + gate" as a demo configuration only.

---

## 5. Residual risk register

| # | Risk | Likelihood | Impact | Accepted? | Note |
|---|---|---|---|---|---|
| R1 | Approved-window abuse (write garbage inside the budget) | High | Medium | Yes | Bounded by TTL + budget + verification |
| R2 | Operator coerced | Medium | High | Partially | Duress code → decoy volume |
| R3 | Gate firmware vulnerability (memory corruption in frame parser) | Medium | High | No — must be fixed | Fuzzing is a required gate in CI |
| R4 | Fault injection / voltage glitch on MCU | Low | High | Research | Brown-out detect; no formal countermeasure in v0.1 |
| R5 | Supply-chain compromise of MCU or NFC reader silicon | Low | Total | Accepted for M0/M1 | Mitigated only by open design and multi-vendor BOM |
| R6 | Side-channel leakage of keys from the gate | Low | High | Research | Out of scope v0.1, published as open problem |
| R7 | User disables the gate because it is annoying | **High** | High | Design concern | Bypass must cost the user something visible; see §6 |
| R8 | The operator approves without reading | High | Medium | Design concern | Forced interaction: button hold + card tap + audible read-back |

R7 and R8 are the two risks that actually kill products like this in the field. They are design problems, not cryptographic ones, and they belong in the threat model rather than in a footnote.

---

## 6. Usability constraints treated as security requirements

1. **A bypass must leave a mark.** A physical jumper that disables the gate is acceptable *only* if it (a) requires opening the case, and (b) is recorded in the gate log as a tamper event on next boot.
2. **Approval must take between 3 and 10 seconds.** Shorter trains the operator to mash the button. Longer trains them to route around the gate.
3. **The gate must never be the reason a routine task fails.** If a legitimate write is rejected more than once per week, the design has failed regardless of its security properties.
4. **Recovery must exist without weakening the model.** Lost NFC card ⇒ a recovery procedure that requires physical disassembly plus an offline recovery key. Never a "reset by email".

---

## 7. Out of scope

| Area | Status |
|---|---|
| Read confidentiality on a live host | Not protected. Assume host compromise ⇒ data readable. Encryption at rest is a separate control. |
| Data in use (screen, clipboard, RAM) | Not protected |
| Network exfiltration of already-read data | Not protected — the gate does not sit on the network path. A companion project could. |
| Backup | Explicitly not provided. |
| Multi-user / enterprise key management | Deferred; single-operator provisioning in v0.1 |

---

## 中文摘要

**资产**：A1 归档数据本体、A2 其完整性、A3 审计记录、A4 闸门自身的配给密钥、A5 主机可用性。明确**不保护**"读"。

**对手**：AD1 拥有主机 root 的自主软件智能体（最主要对手）、AD2 供应链投毒、AD3 远程攻击者、AD4 疲惫的人或出 bug 的脚本、AD5 关机状态的物理窃贼、AD6 胁迫者、AD7 能重刷闸门固件的内鬼。不建模国家级硬件植入与 TEMPEST 级侧信道。

**三个信任边界假设**：①请求链路物理单向，或授权状态存放在主机总线寻址不到的地方；②会话期间固件无法被静默改写，启动介质写保护；③**人看到的是闸门自己显示的提案，而不是主机渲染的界面**——这一条是整个方案最脆弱的地方，一旦被攻破，全部设计失效。

**攻击树要点**：伪造授权（nonce+MAC、单调计数器）、直接拉高开关（GPIO 独占、复位默认断开、欠压检测）、骗人批准（自有显示屏、显示哈希与授权绑定、范围+额度+写后校验）、抹掉审计（哈希链+只增存储+签名锚点）、**拆掉闸门直接挂盘（必须靠密钥托管在闸门内来解决，这是整个方案真正的根）**。

**残余风险**中最要紧的不是密码学问题，而是 R7"用户嫌麻烦就绕过去"和 R8"操作者不看就按"。所以把可用性写成安全需求：绕过必须留痕、批准耗时须在 3–10 秒、合法写入每周被拒超过一次即视为设计失败、恢复流程绝不能靠"发邮件重置"。

**明确不在范围**：主机上的读机密性、使用中数据、网络外传、备份、多用户密钥管理。
