# Contributing

[中文要点](#中文要点)

This project has one unusual rule that overrides politeness: **disagreement is a contribution, and unsupported claims are a bug.**

---

## Before you write code

1. Open an issue. Prefix it `RFC:` if it changes the protocol, the threat model, or a security claim; `TASK:` if it is implementation work.
2. Say what "done" means. A task without an acceptance criterion will be asked for one.
3. If the change affects a protection claim, state which tier (A / B / C) it lands in. See [README §3.2](../README.md#32-what-physical-is-doing-in-that-word).

---

## Rules for claims

| Claim type | Required evidence |
|---|---|
| "This blocks writes" | An attempt to write that fails, with the method described |
| "This is physically disconnected" | A measurement: continuity, scope capture, or current draw |
| "This is encrypted" | The cipher, the mode, the key custody model, and what happens on tamper |
| "This is more secure" | A threat from `docs/01` that is now mitigated, plus what it cost |

Prohibited phrasing, enforced in review:

- "Military-grade", "unhackable", "100% secure", "AI-proof".
- Calling a software-only control "physical".
- Describing a K0 (clear-text drive) configuration as secure.

---

## Code standards

- Rust for host software; `no_std` Rust or C for firmware (justify the choice in the PR).
- Protocol constants live only in `protocol.rs`, shared by `include!` between host and firmware. Never retype a constant.
- Firmware error paths must open the switch. `panic!` in firmware is a defect.
- New `msg_type` requires: RFC, test vector, firmware version gate.
- Every security fix requires a test that fails without the fix.

---

## Hardware contributions

- Design files in KiCad (or a documented alternative), with fabrication outputs included.
- BOM with **two sources** for every critical part, and a note on what happens if one is unavailable.
- Any signal-integrity claim needs a capture: eye diagram, insertion loss, or at minimum a scope shot with the timebase stated.
- Mechanical files in STEP, printable parts in STL with print settings.

---

## Documentation and translation

- Chinese and English are maintained in parallel. If you change one README, change the other or open an issue saying you could not.
- Diagrams: Mermaid preferred (renders on GitHub, diffable in review). No binary-only images for anything that contains text.
- Explain the *why* before the *how*. The audience includes people who will never build this.

---

## Review

- One approving reviewer who is not the author merges an RFC.
- Objections are recorded in the RFC, not deleted after the argument is settled.
- Anyone may request a security review hold on any PR; a hold is cleared only publicly.

---

## 中文要点

- **这个项目有一条压过礼貌的规矩：反对意见就是贡献，没有证据的声明就是 bug。**
- 动手前先开 issue：改协议/威胁模型/安全声明用 `RFC:` 前缀，纯实现用 `TASK:`，并且说清"什么算完成"。
- 任何防护声明必须标明落在哪一级：A（电气）、B（硬件逻辑）、C（操作系统）。C 级的东西不许叫"物理"。
- 禁用措辞：军工级、无法破解、100% 安全、AI 免疫。把只有软件的控制叫"物理"，是一票否决项。K0（明文盘）配置不许被描述成安全方案。
- 协议常量只许存在 `protocol.rs` 一处，主机与固件用 `include!` 共用；固件错误路径必须断开关，**固件里出现 `panic!` 就是缺陷**；新增 `msg_type` 要 RFC + 测试向量 + 固件版本门。
- 安全修复必须附"修之前会失败"的测试。
- 硬件贡献：KiCad 设计文件 + 制造输出；关键器件**双供应商** BOM 并说明缺货时怎么办；信号完整性声明必须给眼图/插入损耗/带时基说明的示波器截图；机械件给 STEP、打印件给 STL 与打印参数。
- 中英文需并行维护：改了一边没改另一边，至少开个 issue 说明做不到。
- 图表优先 Mermaid（GitHub 能渲染、评审时可 diff）；含文字的内容不要用纯图片。
- 先讲"为什么"，再讲"怎么做"；读者里有很多永远不会动手造的人。
- RFC 需要一位非作者的赞成评审才能合并；反对意见记录下来，不因为在争论中落败就删掉。任何人可以给任何 PR 加安全评审搁置，解除必须公开。
