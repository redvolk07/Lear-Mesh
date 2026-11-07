# LEAR-Mesh — Claude Project Memory

> **Purpose:** Persistent project context for Claude Code, Claude Projects, or any Claude conversation. Place this file at the repository root as `CLAUDE.md` for Claude Code, or upload it to a Claude Project's knowledge files. **Source status:** Compiled from the October 2026 `LEAR-Mesh_deck.pptx` and the attached 2025 *Sensors* article. This is a proposed design and evaluation plan, **not a record of finished implementation or measured LEAR-Mesh results**.

## 1. Project identity and mission

- **Name:** LEAR-Mesh (Link-quality and Energy-Aware Routing Mesh).
- **Project title:** Infrastructure-Independent LoRa-Based Self-Healing Mesh Communication System.
- **Setting:** Final-year Electronics and Communication Engineering (ECE) major project.
- **Objective:** Design, build, and evaluate a small gateway-free, self-healing LoRa mesh carrying **phone-to-phone text** over **BLE-connected handhelds** and **solar/battery-powered fixed relays**.
- **Operating constraint:** No cellular service, Wi-Fi infrastructure, Internet, cloud, LoRaWAN gateways, or network servers required for message transport.
- **Essential distinction:** **LoRa PHY ≠ LoRaWAN ≠ custom LoRa mesh.** LEAR-Mesh builds its own MAC and routing over LoRa PHY; it is **not** a LoRaWAN implementation.
- **Motivation:** Communication during disasters, infrastructure failure, and in remote areas without cellular coverage.

## 2. Proposed seven-device prototype

| Role | Count / IDs | Function |
| --- | --- | --- |
| Fixed solar + battery relay | 5: N1–N5, `0x0001`–`0x0005` | Form multi-hop backbone; discover neighbours, forward packets, maintain routes, self-heal, monitor power |
| Handheld BLE/LoRa edge | 2: HA `0x0101`, HB `0x0102` | Pair with phones over BLE; attach to best mesh relay with backup; originate/receive messages; normally do not relay for others |
| Smartphone | 2 | Chat UI, contacts, message history, delivery status; BLE central; no LoRa radio connection |

`0xFFFF` is broadcast. Nominal path:

`Phone A ⇄ BLE ⇄ HA ⇄ LoRa ⇄ N1 ⇄ N2 ⇄ N4 ⇄ N5 ⇄ LoRa ⇄ HB ⇄ BLE ⇄ Phone B`

There are **five LoRa hops** from HA to HB on this representative path. Alternative paths include `N1–N3–N4–N5` and a marginal `N3–N5` link. These are **illustrative test topologies**, not a claim of validated real-world connectivity or range. Distinguish **single-hop RF range** from **end-to-end multi-hop reach**.

## 3. Hardware and radio assumptions

- **Baseline all seven nodes:** ESP32-S3 microcontroller + Semtech SX1262 LoRa transceiver; Heltec WiFi LoRa 32 V3 / Wireless Stick V3 class development boards.
- **Fixed node:** ~6 V, 3–6 W solar panel; CN3791-class MPPT Li-ion charger; protected 18650 cell; 3.3 V regulation; battery/voltage sensing (optional INA219); ~5 dBi omnidirectional antenna.
- **Handheld:** ESP32-S3 + SX1262, NimBLE, optional OLED + button, 1000–2000 mAh rechargeable Li-ion pouch, USB-C charge, ~3 dBi antenna.
- **Future hardware possibilities:** nRF52840 + SX1262 for improved power efficiency; STM32WLE5 for future low-power fixed nodes (external BLE if needed).
- **Initial LoRa PHY:** SF7, 125 kHz BW, coding rate 4/5, 8-symbol preamble, CRC on; TX power and frequency configurable. SF adjustments are not part of the initial adaptive routing design.
- **Regional legality:** The deck illustrates an India-oriented 865–867 MHz band. **Verify current national frequency, EIRP, duty-cycle/LBT, antenna, and equipment requirements from authoritative local sources before transmission.** Do not copy the 915 MHz settings of the Ecuador reference paper into an India deployment without checking rules.

## 4. LEAR routing algorithm — proposed, not validated

LEAR is a compact **DSDV-style, hop-by-hop distance-vector** design. It maintains destination/next-hop/cost/sequence state and seeks reliable, energy-balanced, low-congestion paths without route-request flooding.

Per-link cost:

```text
C(i,j) = ETX(i,j) × f_E(j) × f_Q(j)
PathCost(path) = Σ C(i,j) for links along path
ETX(i,j) = 1 / (d_f × d_r)
f_E(j) = 1 + 2 × max(0, (0.5 − SoC_j) / 0.5)^2
f_Q(j) = 1 + q_j / Q_max
```

Where:
- `d_f`, `d_r`: forward and reverse delivery probabilities.
- `ETX`: calculated from an 8-HELLO-sequence sliding window plus hop-ACK outcomes; proposed EWMA coefficient `0.2`; cap ETX at `10`.
- `SoC_j`: neighbouring node state-of-charge fraction in `[0,1]`; **no energy penalty at or above 50% SoC**, rising below 50%.
- `q_j`: neighbour-reported queue occupancy; `Q_max`: queue capacity.
- SNR: proposed link-admission gate (roughly 3–5 dB above SF-specific demodulation threshold) and ETX seeding; **RSSI is not the primary path metric**.
- **Hysteresis:** only switch when `newCost < 0.9 × currentCost` (at least 10% improvement).
- **Loop control:** destination sequence numbers, split horizon with poison reverse, feasible-successor test, TTL and duplicate suppression. Confirm compatibility and loop-freedom of these mechanisms during implementation rather than assuming the combination is proven.

**Only an illustrative worked example, not data:** primary `N1–N2–N4–N5` cost `3.4`, alternative `N1–N3–N4–N5` cost `3.6`, marginal two-hop route `N1–N3–N5` cost `4.0`. When N2 reaches 15% SoC, the primary cost is illustrated as `4.48` and energy-aware routing should prefer the alternative if it passes hysteresis.

## 5. Discovery, reliability and self-healing — design targets

- Periodic HELLO: **30 s ±10% jitter** in normal operation.
- Hop-by-hop ACK, capped retries (up to **3 retries per hop** in deck's forwarding flow), CAD/LBT, random backoff, queued delivery, duplicate cache and message hold buffer.
- Proposed ACK timeout: `ToA(DATA) + ToA(ACK) + ~300 ms` (roughly ~0.6 s at SF7 under design assumptions, **not** a measured figure).
- **SUSPECT:** 2 ACK failures, 1 missed HELLO, or ETX spike.
- **DOWN:** retries exhausted or 3 missed HELLOs (~90 s for idle failure detection).
- State progression as appropriate: `NORMAL → SUSPECT → LINK_DOWN → LOCAL_REPAIR / REROUTE → REJOIN` (or `UNREACHABLE`).
- On failure: invalidate bad neighbour; promote a feasible backup and resend held packets, or propagate `RERR` upstream and seek updated route; hold packets until timeout, then report `FAILED`.
- Rejoin: 3 consecutive HELLOs and symmetric-link check; return to recovered path only if sufficiently cheaper (10% hysteresis).
- **Recovery targets only:** 3–6 s for local backup promotion (e.g. N2 fails); 6–12 s for wider RERR rerouting (e.g. N4 fails). These are **hypotheses**, not measured results. Timing in idle vs traffic-active failures differs.

## 6. Packet and protocol design

Custom stack: application/chat (phone) → BLE bridge (handheld) → messaging/end-to-end ACK → LEAR routing → MAC (CAD/LBT, hop-ACK, backoff, duty budget) → SX1262 LoRa PHY. Fixed relays only need mesh networking and low-level layers; no BLE chat component.

- **Frame:** 15-byte common header, up to 48-byte payload, optional 4-byte MIC. Maximum **67 bytes only when optional MIC is present** (15+48+4). MIC is **not enabled in the baseline**, so do not assume baseline mesh security.
- Header contains network, packet type/flags, source/destination, previous/next hop, message ID, TTL/hop count, fragment and length fields.
- Source + message ID suppress duplicates; previous/next hop support forwarding and ACKs.
- Proposed packet types: `DATA`, `HELLO`, `ACK` (per-hop), `E2E_ACK`, `RERR`, `ATTACH`, `RSOL` (route solicitation).
- Text: UTF-8; 48 B per fragment; up to 3 fragments / 144 B of fragment payload in the proposed design.
- Delivery state at app: `QUEUED → DELIVERED` or `FAILED`; per-hop ACK ≠ end-to-end delivery acknowledgement.
- Deck's **calculated**, not measured, airtime for a 67 B frame at BW125/CR4/5: ~123 ms SF7, 226 ms SF8, 411 ms SF9, 739 ms SF10, 1561 ms SF11, 2957 ms SF12. Based on Semtech time-on-air equation, preamble 8, CRC on.
- Future: authenticated encryption and key management; do not label unencrypted LoRa traffic confidential or secure.

## 7. BLE and mobile-app plan

- Handheld is **BLE GATT peripheral** (NimBLE); phone is **central**.
- Characteristics: `TX` phone→handheld for text + destination; `RX` handheld→phone for received text/delivery notices; `STATUS` handheld→phone for attachment, SoC, queue and link conditions.
- Intended BLE Secure Connections, OLED passkey pairing, and negotiated higher ATT MTU.
- Mobile chat app: node-ID contacts, SQLite message history, delivery ticks/status; Android Chrome Web Bluetooth may be a prototype shortcut (**not supported on iOS Safari**).
- Handheld has a mesh attachment/edge client and normally does not forward third-party traffic.

## 8. Solar/energy-aware roles — design defaults

| Power mode | SoC | Planned behaviour |
| --- | --- | --- |
| NORMAL | ≥50% | Relay normally; HELLO every 30 s; `f_E=1` |
| ECONOMY | 20–50% | Increased routing penalty; HELLO ~60 s |
| CRITICAL | 10–20% | Avoid relay traffic; HELLO ~120 s |
| LEAF-ONLY | <10% | Origin/destination traffic only, no forwarding for others; HELLO ~300 s |

Mode thresholds are intended to have ±5% hysteresis. **Battery/energy figures are estimates:** 100 mW average→2.4 Wh/day; with 3 peak-sun-hours and 60% efficiency, 3–6 W panel is proposed; nominal ~11 Wh 18650 may theoretically give ~4 days at assumed load. Verify actual TX/RX/idle current, charger/regulator losses, usable capacity and solar availability experimentally.

## 9. Research evidence and novelty boundary

**Closest primary paper:** Arregui Almeida et al., *Gateway-Free LoRa Mesh on ESP32: Design, Self-Healing Mechanisms, and Empirical Performance*, **Sensors** 25(19):6036 (2025), DOI: **10.3390/s25196036**. Attached PDF: `sensors-25-06036-v2.pdf`.

That paper tested a five-node ESP32-S3/SX1262 gateway-free mesh with HELLO, RSSI-age neighbour scoring, hop-ACK, LBT, three retransmission attempts, and failure rerouting; it used a compile-time logical neighbour whitelist, SF7/125 kHz/4/5 at **915 MHz in Ecuador**, and did **not** measure energy or implement production security. **Its measured figures are not LEAR-Mesh results:**

| Published paper scenario | Average PDR | Other |
| --- | --- | --- |
| Controlled / ideal | 95% | 8 retransmissions per repetition (average) |
| Intermediate-node failure | 90% | PRR 88.33%; 15 retransmissions per repetition (average) |
| Urban | 75% | 12 retransmissions per repetition (average) |

The paper's 15 s ACK wait is **not** LEAR-Mesh's proposed airtime-derived ACK timeout.

Other sources covered in the slide-deck bibliography: Wong et al. (2024) LoRa mesh survey; Macaraeg et al. (2020) BLE-connected emergency text with modified AODV; Hong et al. (2022) HBEE energy-efficient LoRa mesh; De Couto et al. (2003) ETX; Meshtastic managed flooding, MeshCore project docs, AODV/DSR/OLSR/RPL, and several 2024–2026 preprints. Treat community docs as practitioner evidence, preprints as preprints, simulation results as simulation results. Verify incomplete reference metadata before academic submission.

**Research gap/contribution:** Not "first mesh," "invented ETX," or "new energy-aware routing." Intended contribution is a **measured integration** of BLE phone text + gateway-free LoRa backbone + composite ETX/battery/queue metric + solar-aware relay roles + local failure recovery, evaluated on real hardware against appropriate baselines. The literature gap is based on a non-exhaustive search and requires IEEE Xplore/Scopus verification before claims of novelty.

## 10. Experimental plan (planned; no LEAR results yet)

| ID | Test | Comparisons / main outputs |
| --- | --- | --- |
| E0 | Link characterization by SF; BLE latency | RSSI, SNR, PDR, timing |
| E1 | Phone-to-phone over 5 LoRa hops | End-to-end delivery, per-hop delay, hop count |
| E2 | Controlled marginal link / min-hop trap | LEAR vs min-hop DV, RSSI-greedy, managed flooding; PDR, delay, overhead |
| E3 | Shut down N2 (local backup) and N4 (wider reroute); buffer on/off | Recovery time, within-60-s delivery; ~10 runs intended |
| E4 | Attenuator-induced link degradation | Route migration and flapping |
| E5 | Inject low SoC at N2; energy term coefficient 0 vs 2 | Route redistribution; Jain's forwarding fairness |
| E6 | Measure overhead, mJ/message, 3–7-day solar log | Energy and SoC behaviour |
| E7 | Recover/rejoin offline relay | Rejoin time and flapping |
| E8 (optional) | SimPy with 20–50 simulated nodes | Scaling trends only, not physical scalability proof |

Method: target **at least 100 messages × 5 repetitions per case**, interleave cases, report means and 95% CIs when justified, synchronize logs from one PC clock if possible, use controlled topology (whitelist/attenuators) **plus field tests**, separate detection/repair time, measure PDR, end-to-end delay, throughput, retries, recovery, control airtime/overhead, mJ per *delivered* message, and fairness. Baseline comparisons must use consistent PHY, traffic, topology and instrumentation.

**16-week schedule (adjustable):** literature/requirements → hardware/BLE → MAC/HELLO/ETX → LEAR and ACK → healing/queues → solar → phone app/integration → experiments (~weeks 12–14) → report (~14–15) → demonstration/viva (~16).

## 11. Implementation guidance and source-of-truth rules

Proposed software stack: **C/C++**, SX1262 radio driver (e.g., RadioLib), ESP32-S3 FreeRTOS, NimBLE for handhelds, nonvolatile configuration, serial logging. Suggested modules: `radio`, `packet_codec`, `mac`, `neighbor_manager`, `etx`, `routes`, `forwarder`, `acks`, `retry_queue`, `healing_fsm`, `power_manager`, `ble_gatt` (handheld), `diagnostics`, `config`.

**Status as of the supplied October 2026 deck:** an architectural proposal and research/evaluation plan. The materials do **not** substantiate completed LEAR-Mesh firmware, seven assembled units, a finished mobile app, or measured LEAR-Mesh outcomes. Ask for or inspect current code/logs before describing any subsystem as implemented.

When assisting:

1. Use this memory as the **design baseline**, not as proof of implementation. New explicit user decisions supersede old proposed details; track such changes.
2. Mark statements as **proposed**, **calculated**, **illustrative**, **published by another study**, or **measured on LEAR-Mesh**. Never blur these categories or fabricate data.
3. Preserve **custom LoRa PHY mesh, 5 fixed + 2 handheld topology, BLE edge, LEAR ETX×energy×queue, backup/local healing** unless asked to revise architecture.
4. Check airtime, half-duplex scheduling, ACK collisions, duty-cycle/legality, memory, power budget, loop prevention and protocol overhead when recommending changes.
5. Explain assumptions, units, equations, test methodology and limitations; produce modular, buildable firmware and realistic lab experiments when requested.
6. Critically assess technical flaws and disagreements rather than blindly confirming them; separate source-grounded details from new recommendations.
7. For citations, consult the actual paper and deck; avoid inventing authors, DOIs, benchmark numbers, regulatory details or field results.
8. Keep answers suitable for an ECE major project: precise, implementation-oriented, academically honest, reproducible.

## 12. Source files / continuity

- `LEAR-Mesh_deck.pptx` — 26 slides (October 2026): primary **project specification**, architecture, figures, calculations, reference list and test plan.
- `sensors-25-06036-v2.pdf` — published **reference paper**, 31 pages; DOI above.

If the attachments are absent in a future Claude session, this file is the portable context; **do not claim to have opened or verified missing source files**. Request files only when a task needs deeper source-specific evidence.

### Future session log (update as work progresses)

- **Current confirmed implementation status:** Unknown; do not infer progress from the design deck.
- **Confirmed hardware purchased/assembled:** Unknown.
- **Repository/code version:** Unknown.
- **Measured experiments / datasets:** None supplied for LEAR-Mesh.
- **Open decisions:** Exact board revisions and pin mapping; legally compliant operating configuration; message buffer sizing/timeouts; complete route-table protocol; true power measurements; cross-platform app implementation; precise test instrumentation.

When implementation or experiments occur, add a dated entry here distinguishing *decision*, *implemented*, *tested*, *result*, and *next action*; remove outdated statements without rewriting historical results.
