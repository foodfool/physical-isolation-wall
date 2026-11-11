# 03 · Hardware

[English](#1-the-central-hardware-question) · [中文摘要](#中文摘要)

> Status: proposals, not proven designs. Component choices are starting points for someone with a scope and a VNA to confirm or replace.

---

## 1. The central hardware question

"Block writes physically" sounds like one problem. It is two, and they have different answers depending on the bus.

| Bus | Directionality at the electrical level | Can you cut "write only"? |
|---|---|---|
| **USB 2.0** (480 Mbps) | One differential pair, half-duplex; TX and RX share D+/D− | **No.** Any electrical cut kills reads too. Write gating must be done in a bridge that terminates the SCSI/BOT protocol and refuses write commands. |
| **USB 3.x SuperSpeed** (5–10 Gbps) | Separate SSTX and SSRX differential pairs, full-duplex | **Yes.** Cut SSTX (host→device) and keep SSRX live. |
| **SATA** (1.5/3/6 Gbps) | Separate TX and RX pairs per direction, full-duplex | **Yes.** Cut the host→device TX pair. This is exactly how commercial forensic write blockers are built. |
| **NVMe / PCIe** | Separate lanes per direction (Tx/Rx), full-duplex | **Yes in principle**, but PCIe link training, hot-plug, and 8–16 Gbps signalling make a hobby-grade mux impractical. Practical answer: gate the drive's power, or bridge through a small SoC. |
| **SD / microSD** | Single bidirectional bus, but the card has a **physical write-protect notch** sensed by the reader | Partial — a real mechanical interlock, but coarse and reader-dependent. |

This table drives the whole hardware plan:

- **M0 (cheapest, strongest)** avoids the problem: the drive is *entirely* offline by default, and comes online for a bounded time on physical approval. All-or-nothing, so direction does not matter.
- **M1 (recommended)** accepts the bridge: reads flow through an SBC that terminates the protocol, writes are additionally gated by a **direction-selective cut** on the SATA side.
- **M2 (hard)** builds the directional switch into a custom PCB.

---

## 2. M0 — "Air-Gap Timer"

**Semantics:** the archive drive has no power and no data connection. It does not exist as far as the host is concerned. Physical approval brings it online for *N* minutes (default 10), after which the gate cuts power and signal at the hardware level.

This is the strongest security tier available at hobby cost, and it is honest: it does not pretend to be "read-open, write-closed". Use it for an archive drive that is normally not needed.

```
                ┌──────────── Gate ────────────┐
  Host USB-A ───┤ USB2 mux (TS3USB221-class)   ├─── Archive drive (USB enclosure)
                │        ↕ control             │
                │ VBUS load switch (TPS22918)  │
                │                              │
                │  RP2040  ← button            │
                │    │     ← PN532 (13.56 MHz) │
                │    │     → 0.96" OLED        │
                │    └─────→ FRAM log          │
                └──────────────────────────────┘
```

Behaviour that matters:

- Power-on state is **signal disconnected, VBUS off**. Firmware never "opens" anything it did not receive a physical grant for.
- The mux's default (unpowered) state must be *disconnected*; verify this on the bench, because mux default states are easy to get wrong and are the usual place a design silently fails open.
- On expiry the firmware sets both the mux select and the load switch low, and only then writes the log record.
- Removing the drive requires a physical approval as well (`EJECT_REQUEST`), so a host cannot yank the enclosure's state machine.

**Approximate BOM**

| Part | Role | Qty | Note |
|---|---|---|---|
| RP2040 board (Pico-class) | Gate MCU | 1 | Dual-core; one core can busy-wait the watchdog |
| PN532 module | 13.56 MHz NFC reader | 1 | I2C or SPI; UID alone is not enough — use challenge-response |
| TS3USB221-class USB 2.0 mux | D+/D− path switch | 1 | **Verify default state = disconnected** |
| TPS22918-class load switch | VBUS gate | 1 | Low Rds(on), controlled rise time |
| SSD1306 OLED 0.96" | Proposal display | 1 | Must be driven by the gate, never mirrored from the host |
| Momentary push button + RC | Presence input | 1 | Debounce in firmware, not in hardware |
| FRAM (MB85RC-class) or NOR flash | Audit log | 1 | FRAM survives power loss without wear concerns |
| USB-C receptacle ×2, enclosure | Mechanical | — | Sealed with a tamper-evident label |

Cost target: under ¥300 in single quantities.

**Known weaknesses:** 480 Mbps through an analog mux needs care with stub length and impedance; the archive is unavailable while offline; no fine-grained control.

---

## 3. M1 — Bridged, read-open, write-gated (recommended reference)

```
   Host ──USB──► SBC (CM4 / RK3566 / Orange Pi class)
                     │  USB gadget: mass-storage LUN(s) over configfs
                     │  block layer: read-only by default
                     │
                     ├── SATA (or USB3) ──► Direction switch ──► Archive drive
                     │                          ▲
                     │                          │ drive-enable
                     └── UART (RX only) ◄── MCU (RP2040/STM32)
                                              ▲
                                    button + NFC + display + log
```

Why this works:

1. The host only ever sees a USB mass-storage device exported by the SBC. It has no path to the SBC's shell, filesystem, or firmware — the gadget exposes block storage and nothing else.
2. The SBC's block layer holds the archive **read-only** whenever no session is live, and the SBC refuses write commands at the protocol layer.
3. Independently of (2), the MCU controls a **direction switch on the SATA TX pair**. Even if the SBC's software were fully subverted by a host exploit, the physically-cut transmit pair means write data cannot reach the drive.
4. The MCU's control line is not reachable from the host: it is wired to the SBC's GPIO, and the SBC's userspace daemon is the only thing that talks to it — over a link whose direction is enforced by the wiring.

**The honest caveat:** M1's tier-A guarantee covers the SATA direction cut. The USB gadget path itself is tier-B — it depends on the SBC's Linux stack being uncompromised. To move the whole design into tier A, the archive must also be encrypted with a key held by the gate (mode K1), so that even a fully subverted SBC cannot produce a readable copy of the drive's contents without a presence event.

**Direction switch options**

| Option | Bandwidth | Difficulty | Note |
|---|---|---|---|
| Discrete high-bandwidth differential analog switch | 6 Gbps marginal | Medium | Needs careful layout; eye diagram will close if stubs are bad |
| SATA redriver with output disable | 6 Gbps | Medium-easy | Redrivers (e.g. TUSB501-class for USB3, SATA-specific equivalents) often have an output-enable pin; using it as the gate is elegant — **subject to confirming the disable state is high-Z, not "repeats garbage"** |
| Relay (reed or signal relay) | 1.5–3 Gbps | Easy | Audible click is a feature: the human hears the gate work. Not viable at 6 Gbps |
| Power gate only | n/a | Trivial | Cuts both directions; the M0 semantics applied to an internal drive |

**Open hardware dependency:** the exact SATA-redriver part that exposes a clean output-enable is not yet verified by this project. This is a first-class task for a hardware contributor and should be tracked in an issue.

---

## 4. M2 — Pass-through board

Once M1 is proven, M2 miniaturises it: a 4-layer board with an edge connector, the direction switch, a redriver per lane, the MCU, NFC front-end (ST25R3911B-class), a secure element for K1 key custody, and a tamper mesh.

Things that will bite:

- **Differential pair routing through a switch** — impedance discontinuities at the switch pads will destroy the eye at 6 Gbps. Expect to iterate with a VNA or at least a scope with a compliance mask.
- **Return path / ground stitching** — a cut in the TX pair must still leave a solid reference plane; a naive cut creates a stub antenna.
- **Inrush and hot-plug** — SATA and NVMe drives dislike abrupt power removal. The gate must sequence: quiesce, drain, cut. Firmware must not simply drop a MOSFET.
- **ESD** — the enclosure touches the drive; every external line needs protection, and protection capacitance must not ruin the signal.

---

## 5. Mechanical and tamper

| Measure | Cost | Effect |
|---|---|---|
| Tamper-evident label over a case screw | ~¥2 | Detects casual opening; useless against a careful attacker |
| Case switch (microswitch) | ~¥1 | Logs opening; can trigger key zeroisation in K1 mode |
| Conformal coating / potting | ~¥20 | Slows probing; makes repair impossible |
| Shielded enclosure | ~¥50 | Reduces EM leakage; not a side-channel cure |
| Tamper mesh on the PCB | high | Only worth it at M3 |

Design rule: **a tamper response must never be "silently continue".** Either it zeroises (K1), or it records and refuses to arm until the operator acknowledges on the device itself.

---

## 6. Provisioning port

A pogo-pin header inside the case, exposed only with the lid off. It carries UART for firmware recovery plus a one-time provisioning interface. It must be **physically inaccessible** when the case is closed — not merely "disabled in firmware", because firmware can be argued with.

---

## 7. What a hardware contributor should pick up first

1. Bench-verify that a TS3USB221-class mux defaults to *disconnected* with no control power.
2. Find a SATA redriver/mux with a verifiable high-Z output-disable, and measure insertion loss at 6 Gbps.
3. Propose a 4-layer M0 carrier board.
4. Characterise relay-based switching at 1.5 and 3 Gbps for the "audible click" variant.
5. Determine the smallest secure element that supports ECDH plus monotonic counters at hobby price.

---

## 中文摘要

**一个必须先想清楚的事实**：不同总线的"方向性"完全不同，这决定了能不能真的"只断写"。

- **USB 2.0**：只有一对半双工差分线，电气上无法只断写 → 必须靠桥接器在协议层拒绝写命令。
- **USB 3.x**：SSTX / SSRX 是分开的，**可以只切 SSTX**。
- **SATA**：TX/RX 各自独立，**可以只切主机→盘的 TX 对**，商用取证写阻断器正是这么做的。
- **NVMe/PCIe**：原理上可切，但链路训练与 8–16 Gbps 信号让业余成本下的多路开关不现实 → 务实做法是电源门控或 SoC 桥接。
- **SD 卡**：有真正的机械写保护缺口，但粒度粗、依赖读卡器。

**M0「气隙定时器」**：默认盘完全断电断线，物理确认后上线 N 分钟（默认 10），到期硬断。全有或全无，所以方向问题不存在。这是业余成本下能拿到的**最高电气等级（A 级）**，且诚实——不假装"读常开写受控"。BOM 约 ¥300 以内：RP2040 + PN532 + TS3USB221 类 mux + TPS22918 类负载开关 + OLED + FRAM。**最容易翻车的地方**：mux 在无控制电时的默认态必须是"断开"，必须实测，否则设计会静默失效为"常通"。

**M1「读常开、写受控」（推荐）**：SBC 做 USB gadget 桥接，盘在块层默认只读；同时由 MCU 独立控制 SATA TX 方向的开关。主机就算打穿 SBC 的软件，写数据也到不了盘上。**诚实的分级**：M1 的 A 级保证只覆盖 SATA 方向切断；USB gadget 这条路径仍是 B 级（依赖 SBC 的 Linux 栈不被攻破）。要让整体进入 A 级，必须叠加 K1 密钥托管。

**方向开关的四种选项**：离散高带宽差分开关（6 Gbps 勉强，布线生死）、带输出使能的 SATA redriver（优雅，但需确认禁用态是高阻）、继电器（1.5–3 Gbps，清脆的"咔哒"声反而是优点——人能听见闸门在工作）、纯电源门控（等于把 M0 语义搬到内置盘）。**"哪颗 SATA redriver 有干净的输出使能"目前未被本项目验证**，这是硬件贡献者的头号任务。

**M2 自制直通板的坑**：差分对穿过开关焊盘的阻抗不连续、切断 TX 后留下的 stub 天线、SATA/NVMe 讨厌粗暴掉电（必须"静默—泄放—切断"时序）、ESD 保护电容不能毁掉信号。

**机械与防拆**：防拆标签、壳体微动开关（可触发 K1 零化）、灌封、屏蔽壳。铁律：**防拆响应绝不能是"静默继续"**——要么零化密钥，要么记录并在设备本身上要求操作者确认后才允许重新武装。

**配给口**：壳内的 pogo pin，只在开盖时可达，**必须在物理上不可达**，而不是"固件里禁用"——固件是可以被说服的。
