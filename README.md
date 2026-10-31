# WireGuard at ~5 Mbit/s on a CCR2004: interaction between packet reordering and the FreeBSD TCP stack

[Français](wireguard-ccr2004-freebsd.fr.md) | **English**

Setup: MikroTik **CCR2004-1G-12S+2XS, RouterOS 7.23.8**, connected to a Freebox Ultra, with a WireGuard tunnel to **OPNsense 26.7 hosted at OVH**. A downstream TCP test, with OPNsense as the sender, plateaued at around **5 Mbit/s**.

Cross-analysis of captures from inside and outside the VPN, together with the TCP exchanges, revealed two phenomena:

## 1. Actual packet reordering

WireGuard packet counters were in order when leaving OVH, but approximately **19% arrived after a higher counter had already been observed in the CCR’s WAN capture**, with a maximum reordering depth of nine positions. SACKs from the PC confirmed actual reordering at the TCP receiver, beyond a mere capture artifact. No transmitted TCP packet was missing when comparing captures over this window.

A strong clue: the same WireGuard UDP flow was split almost exactly between **two CCR cores**, with each subsequence remaining in order. This makes parallel receive processing a leading hypothesis, without conclusively identifying the responsible stage.

The MikroTik sniffer’s CPU metadata showed the following exact distribution across the **9,256 downstream UDP datagrams**:

| Core reported in the capture | Captured datagrams |
| --- | ---: |
| **CPU 1** | **4,627** |
| **CPU 3** | **4,629** |

When examining each core’s WireGuard data packets separately, there were **zero counter inversions on CPU 1 and zero on CPU 3**. In the combined sequence, **1,776 out of 9,255 packets** had a counter lower than the highest counter already observed.

This pattern is consistent with two parallel processing paths that each preserve their own order, but introduce reordering when their outputs are merged. On its own, it cannot distinguish between hardware, the driver, RX distribution, or software scheduling.

## 2. A detrimental response from the classic FreeBSD TCP stack

The reordering was accompanied by numerous spurious retransmissions, confirmed by DSACKs, and a congestion window frequently reduced to just a few kilobytes. Throughput collapsed despite the absence of measured packet loss in this capture.

**SACKs** identify blocks received beyond a gap in the sequence; **DSACKs** report data received more than once. A 20-second capture showed **366 TCP retransmissions and 358 ACKs containing a DSACK**, strongly supporting the presence of spurious retransmissions.

The configured congestion control algorithm was **CUBIC**, verified by `net.inet.tcp.cc.algorithm=cubic`. CUBIC adjusts the congestion window using, among other mechanisms, a cubic function of the time elapsed since a congestion event. If reordering is interpreted as packet loss, recovery actions and congestion window reductions can occur repeatedly, preventing throughput from increasing.

**CUBIC and RACK operate at different levels**: CUBIC governs the congestion window; FreeBSD’s alternative `rack` TCP stack changes, among other things, loss detection, loss recovery, and transmission management. The test therefore involved replacing **`freebsd` with `rack` while keeping CUBIC**.

## Controlled A → B → A test results

Selecting the **RACK TCP stack on the OVH sender**, while keeping CUBIC, produced the following controlled result:

| Sender TCP stack | Received throughput | Retransmissions over 30 s |
| --- | ---: | ---: |
| `freebsd` — baseline | 4.96 Mbit/s | 537 |
| `rack` | **1.31 Gbit/s** | 1,368 |
| Back to `freebsd` | 4.93 Mbit/s | 530 |

The **Cwnd reported by iperf3** increased from a few kilobytes — often **2.70 KiB** — to **8.00 MiB** with RACK. The absolute retransmission count under RACK must be considered relative to the volume transferred: **4.57 GiB versus approximately 18 MiB**, or more than 260 times as much data.

The iperf3 server was restarted between stack changes to recreate its listening socket and the test connections.

**The A → B → A test demonstrates a causal dependence on the sender’s TCP stack**, with an approximately 264-fold throughput increase. RACK changes, among other things, loss detection/recovery and transmission management; this experiment does not isolate their respective contributions.

## Scope of the fix and limitations

The validated fix applies to TCP connections terminating on the OVH OPNsense host. It does not change the TCP behavior of machines whose packets OPNsense merely routes. **The exact origin of the reordering remains to be confirmed: this result is insufficient to identify a specific RouterOS bug or to conclude that the reordering itself has been fixed.**

## Technical references

- [CUBIC — RFC 9438](https://www.rfc-editor.org/rfc/rfc9438.html)
- [RACK and Alternate TCP Stacks for FreeBSD](https://freebsdfoundation.org/our-work/journal/browser-based-edition/networking-10th-anniversary/rack-and-alternate-tcp-stacks-for-freebsd/)
