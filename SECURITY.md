# Security Policy

[中文要点](#中文要点)

## Reporting

This project has no bug bounty and no vendor. It has a maintainer and an issue tracker.

- **Ordinary findings:** open a public issue. Most of what matters here is design-level and benefits from daylight.
- **Findings that could lead someone to lose data right now** (e.g. a firmware bug that arms the switch without presence, a protocol flaw that allows a grant to be forged): contact the maintainer privately first via the address listed in `SECURITY_CONTACT` on the repository profile, and allow 30 days before disclosure.

Please include:

1. Which tier the flaw breaks (A / B / C) — see [README §3.2](../README.md#32-what-physical-is-doing-in-that-word).
2. The exact hardware or firmware version, and the configuration (K0 or K1).
3. Reproduction steps, with the measurement or capture that demonstrates it.

---

## What the project promises

| Promise | Detail |
|---|---|
| Acknowledgement | Within 7 days |
| Public credit | Offered for every accepted report, including design-level ones |
| Honest severity | The maintainer will not downplay a finding because it makes the design look bad |
| Documentation fix | A confirmed design flaw is edited into the threat model, not just patched in firmware |
| No silent retroactive claims | If a claim was wrong, the README keeps a note saying it was wrong |

---

## What the project will not do

- Ship firmware without a reproducible build recipe.
- Publish a security claim that has not been tested by an attempt to defeat it (see [roadmap, definition of done](../docs/05-roadmap.md#definition-of-done-for-a-security-claim)).
- Hide a known limitation in a wiki page while the README stays clean.

---

## Known limitations you should not report as vulnerabilities

These are documented gaps, not bugs:

1. **Approved-window abuse** — a compromised host can write garbage inside an approved byte budget. Bounded by design; see residual risk R1.
2. **Read confidentiality on a live host** — reads are open by design.
3. **Coercion** — duress mode is a delay, not a shield.
4. **Fault injection** — v0.1 has no countermeasures beyond brown-out detection.
5. **K0 mode** — a clear-text drive behind the gate is defeated by removing the drive. This is stated in the README.

Reporting one of these is still welcome as an RFC — but it will be answered with "yes, that is in the threat model", not treated as a new finding.

---

## 中文要点

- 本项目没有奖金、没有厂商，只有一个维护者和一个 issue 区。
- **普通发现**：开公开 issue。这里多数问题都是设计层面的，见光反而更好。
- **可能让人立刻丢数据的发现**（例如固件在没有物理在场的情况下合上开关、协议允许伪造授权）：先私下联系仓库主页 `SECURITY_CONTACT` 里的地址，给 30 天再公开。
- 报告请包含：破坏了哪一级（A/B/C）、确切的硬件或固件版本与 K0/K1 配置、复现步骤与用来证明它的测量或抓包。
- 承诺：7 天内回应；每个被采纳的报告都给公开署名（包括设计层面的）；不因为"显得设计难看"而降低严重性；确认的设计缺陷要**改写威胁模型**，不只是打个固件补丁；曾经的错误声明不悄悄删掉，README 里保留"这里曾经写错了"的记录。
- 不做：发布没有可复现构建配方的固件；发布没有被"击败尝试"验证过的安全声明；把已知限制藏在 wiki 里而 README 保持干净。
- **不要当成漏洞上报的已知限制**：批准窗口内的滥用、活主机上的读机密性、胁迫、故障注入、K0 模式。这些写在威胁模型里。把它们提成 RFC 仍然欢迎，但回答会是"是的，这在威胁模型里"，而不是当成新发现。
