# Week 05 — OSPF (CML2)

## Lab assets
- `Week_05_OSPF_clean.yaml` — sanitized CML2 import candidate with four `iol-xe` routers and four physical links.
- Original `Week_05_OSPF.yaml` and actual topology screenshot are preserved in the private Week 05 Google Drive study folder, not published here.

## Purpose
Practice single-area OSPF (process 1, area 0), point-to-point neighbor formation, route costs, Equal-Cost Multipath (ECMP), CEF forwarding, controlled interface failover, and IP SLA reachability monitoring.

## Address plan
| Link | A | B |
| --- | --- | --- |
| R1–R2 | R1 Et0/0 10.0.12.1/30 | R2 Et0/0 10.0.12.2/30 |
| R1–R3 | R1 Et0/1 10.0.13.1/30 | R3 Et0/0 10.0.13.2/30 |
| R2–R4 | R2 Et0/1 10.0.24.1/30 | R4 Et0/0 10.0.24.2/30 |
| R3–R4 | R3 Et0/1 10.0.34.1/30 | R4 Et0/1 10.0.34.2/30 |
| Destination | R4 Loopback0 | 172.16.100.1/32 |

## Final intended OSPF costs
R1 Et0/0 = 10 (default on this lab image), R1 Et0/1 = 5, R2 Et0/1 = 20, R3 Et0/1 = 25, R4 loopback = 1 (expected lab behavior). The intended R1-to-R4 loopback metric is **31 via R2 and 31 via R3**.

## Network assurance
On R1, IP SLA operation 10 sends ICMP Echo probes to R4 Loopback0, sourced from R1 10.0.13.1, every second with timeout 500 ms and threshold 400 ms. The threshold was deliberately set below the timeout to allow scheduling on the observed IOS XE version.

## Important distinctions
- The source CML2 lab was configured and tested live on IOS XE IOL 17.15.1. The cleaned version was generated from the observed final topology and selected configurations; **its successful fresh import and boot have not been independently retested**.
- The cleaned export omits device-generated certificate chains, system/PKI boilerplate, unrelated services and accumulated canvas annotations. It is not a byte-identical copy.
- Evidence of FULL adjacencies, installed ECMP next hops, and post-failure reachability was observed on the original live lab, not on this sanitized copy.
- IP SLA cumulative counts with one-second probing do **not** establish an exact OSPF convergence time or a three-second application recovery SLA.
- For a baseline single-path routing exercise, change R3 Ethernet0/1 cost from 25 to 40, giving R1's R3 path metric 46 while the R2 path remains 31.

## Reproduce
Import the YAML into Cisco Modeling Labs 2, ensure an available `iol-xe` image, start the nodes, and verify with:
```text
show ip ospf neighbor
show ip ospf interface brief
show ip route 172.16.100.1
show ip cef 172.16.100.1 detail
show ip sla summary
```
Do not treat expected outputs as observed results until validated on the imported lab.
