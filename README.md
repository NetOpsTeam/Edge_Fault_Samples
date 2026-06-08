# Edge Network Fault Scenarios Dataset

This document summarizes six representative edge network fault scenarios designed for fault diagnosis, root-cause analysis, KPI correlation modeling, and alarm reasoning in edge computing and 5G-oriented network environments. The scenarios cover both hard failures and gray failures, including hardware degradation, congestion propagation, routing convergence, control-plane anomalies, and application-visible performance degradation.

The corresponding edge network fault dataset is being prepared for public release. Due to ongoing data processing, consistency checks, annotation calibration, scenario verification, documentation improvement, and quality assurance, the dataset has not yet been open-sourced. We are also reviewing metric naming, alarm semantics, time-series alignment, metadata completeness, and possible anonymization requirements to ensure that the released dataset is reliable, reproducible, and easy to use. We will open-source the dataset as soon as these preparation steps are completed.

## Scenario 1: Thermal-Induced Slow Hardware Fault

This scenario describes a slow hardware fault caused by rising device temperature. At the beginning, the data-plane device operates normally. As the computing workload increases, the CPU temperature rises and briefly crosses a warning threshold, triggering a minor temperature alarm. The temperature then returns to normal after the workload decreases.

Later, when the workload remains high for a longer period and cooling becomes ineffective, the line-card CPU temperature rises again and triggers thermal throttling. The forwarding rate becomes limited, causing increased latency and packet loss for traffic passing through the faulty node. However, the fault is not severe enough to trigger a keepalive timeout, so no active/standby switchover occurs.

Typical alarm evolution:

- `MINOR: System Temperature Over Threshold`: early warning, while KPI values are still normal.
- `INFO: System Temperature Normal`: temperature returns to the normal range.
- `MAJOR: Thermal Throttling Engaged`: hardware protection is activated and forwarding performance is degraded.

This scenario is useful for evaluating whether a diagnostic model can distinguish an early thermal warning from a true slow hardware fault and identify the affected traffic path without relying on device-down alarms.

## Scenario 2: Congestion Backpressure in a Lossless Edge Network

This scenario models congestion backpressure in a lossless Ethernet-like edge network where Priority Flow Control (PFC) is enabled. A downstream device, represented as `Device_NJ`, becomes slow first. Its egress traffic rate decreases while its ingress traffic remains high, causing buffer usage to exceed the congestion threshold. `Device_NJ` starts sending PFC frames upstream.

The congestion then propagates hop by hop:

- `Device_NJ` slows down and sends PFC frames.
- `Device_MD` is pressured by `Device_NJ` and starts buffering traffic.
- `Device_TX` is pressured by `Device_MD`.
- `Device_CA2`, as the ingress-side source node, continues receiving high traffic while its outgoing traffic is suppressed, causing severe packet drops.

Key symptoms include high buffer utilization, large divergence between receive and transmit rates, high PFC frame counts, and severe packet loss at the source-side ingress node. The root cause, however, is not `Device_CA2`; it is the downstream congestion source at `Device_NJ`.

This scenario is designed to test whether an agent can trace congestion backward through PFC propagation and distinguish the root congestion point from downstream symptoms.

## Scenario 3: Link Break and Routing Reconvergence

This scenario describes a physical link break between `node7` and `node8`. In the original topology, `node7` acts as a relay node with two neighbors, `node5` and `node8`. When the `node7-node8` link fails, `node7` becomes a stub node connected only to `node5`, and transit traffic can no longer pass through it.

Before the failure, traffic may follow a path such as:

```text
... -> node5 -> node7 -> node8 -> node9 -> ...
```

After routing convergence, the traffic is redirected to an alternate path such as:

```text
... -> node5 -> node6 -> node10 -> node9 -> ...
```

Expected metric behavior includes a sharp traffic drop on `node7`, reduced traffic on `node8`, and increased traffic on the backup-path nodes such as `node6` and `node10`. During the convergence period, transient packet loss, latency spikes, CPU spikes, and temporary PFC activity may occur. After convergence, packet loss should return to normal, while the backup path may show higher steady-state utilization, buffer usage, latency, and temperature.

This scenario evaluates the ability to recognize link-down events, correlate route changes with traffic redistribution, and separate transient convergence effects from persistent congestion.

## Scenario 4: MTU Black Hole

This scenario represents an MTU black hole caused by three simultaneous conditions:

- The source host sends large packets with the `DF` (Don't Fragment) bit set.
- An intermediate node, such as `Node 7`, has an egress MTU smaller than the packet size, for example due to encapsulation overhead such as VXLAN.
- ICMP `Fragmentation Needed` messages are blocked by a security gateway or firewall, such as `Node 5`, or are ignored by the source host.

As a result, `Node 7` silently drops large packets and attempts to send ICMP Type 3 Code 4 messages back to the source. Because those ICMP messages are blocked, the source never learns that it must reduce packet size. TCP sessions repeatedly time out and retransmit large packets, while useful application traffic appears to disappear.

Typical observations:

- `Node 7`: high input traffic at first, sharp output traffic drop, high packet loss, low buffer usage, low device-level latency, and a spike in `icmp_unreachable_sent`.
- `Node 5`: high `icmp_unreachable_received` and ACL/drop counters caused by blocked ICMP messages.
- `Node 8` and downstream nodes: traffic starvation, reduced CPU load, low buffer usage, and no direct packet-loss symptom.

This scenario is valuable because the device that drops packets and the policy device that completes the black-hole loop are different. A correct diagnosis must combine data-plane loss, ICMP behavior, and security-policy effects.

## Scenario 5: Memory Leak and Control-Plane Collapse

This scenario describes a memory leak on `node5`, caused by a defective critical process such as a routing daemon, monitoring agent, or control-plane service. Memory usage gradually increases from a high but tolerable level to near exhaustion. As available memory disappears, the operating system may trigger swapping or an out-of-memory condition, causing CPU utilization to surge and control-plane responsiveness to collapse.

The failure develops in four stages:

- Silent data-plane drops: neighboring nodes still believe `node5` is reachable, but traffic sent to it experiences high latency and packet loss.
- Congestion and backpressure: retransmissions increase, queues build up on adjacent links, and neighboring devices may experience tail drops.
- Adjacency flapping: `node5` fails to send routing keepalive or hello packets in time, causing neighbors such as `node4`, `node6`, and `node7` to declare adjacency loss.
- Traffic migration and secondary congestion: routing convergence redirects traffic to longer alternate paths, potentially overloading links such as `node2-node3` or `node3-node6`.

This scenario is intended to test long-horizon temporal reasoning. The root cause is not the later route change or secondary congestion, but the progressive memory exhaustion on `node5`.

## Scenario 6: Link Degradation and High BER Gray Failure

This scenario models a gray failure on the physical link between `Node5` and `Node7`. The root cause may be an aging optical module, fiber micro-bending, connector contamination, or signal-to-noise ratio degradation. The link experiences high bit error rate (BER), CRC errors, and packet loss, but it does not fully go down at Layer 1. Therefore, traffic may continue to use the degraded path instead of being automatically rerouted.

The failure evolves from silent packet loss to broader performance degradation:

- Packet loss increases on the `Node5-Node7` link, even when traffic volume does not increase.
- TCP retransmissions reduce throughput and increase application-visible latency.
- Buffers on `Node5`, `Node7`, and neighboring devices begin to build up.
- PFC counters may fluctuate or rise if flow control is enabled.
- If BFD or routing protocols are configured aggressively, the degraded link may flap, causing unstable routing and unexpected traffic shifts.

Unlike a clean link break, this gray failure is difficult to detect because connectivity remains partially available. The diagnostic focus is the abnormal correlation between stable or moderate traffic load, rising packet loss, CRC-like behavior, higher latency, and increasing congestion signals.

## Dataset Release Plan

We plan to open-source the edge network fault dataset as soon as possible. Before release, we are completing several preparation steps:

- Data processing and normalization across all scenarios.
- Time alignment between KPIs, alarms, topology changes, and fault injection points.
- Manual proofreading and correction of scenario descriptions.
- Annotation consistency checks for root causes, affected nodes, propagation paths, and alarm labels.
- Validation of metric trends against the intended network behavior.
- Documentation of topology, field definitions, sampling intervals, and usage examples.
- Packaging and reproducibility checks to make the dataset convenient for research and benchmarking.
- Review for sensitive information, naming consistency, and possible anonymization requirements.

Once these steps are complete, the dataset will be released publicly.
