# Week 03 — EtherChannel (CCNP ENCOR)

A reproducible **Cisco Modeling Labs (CML2)** exercise for Link Aggregation Control Protocol (LACP), EtherChannel member health, 802.1Q trunks, and Rapid PVST+ path selection.

> **Scope:** The supplied YAML records a starting CML topology and saved switch configurations. The failure scenarios below are **tasks to introduce**; the YAML does not preconfigure those failures. Tested results cited below are observations from one IOL-L2 lab and may differ with image/version or configuration changes.

## Files

- [CML2 lab YAML](./ChatGPT___CCNP_EtherChannel__WK03_Labs.yaml) — importable CML topology.
- [Command Line Reference (Excel)](./05%20%E2%80%94%20Command%20Line%20Reference.xlsx) — configuration, verification, and troubleshooting commands. Excel may need downloading to view/edit locally.

## Prerequisites

- Cisco Modeling Labs with an available **IOL-L2 / ioll2-xe** switch image (the original study used IOS XE **17.15.1**).
- Enough licensed node capacity to import a four-node topology; only **three nodes are physically connected** in the YAML.
- Familiarity with trunking, VLANs, STP, and basic Cisco IOS CLI.
- Use an isolated lab, not production switches. Take a snapshot or save a known-good configuration before each fault.

## Topology


<img width="1448" height="1086" alt="Week 03 - Cisco Study Lab Network Topology" src="https://github.com/user-attachments/assets/48fdc647-de41-471a-ac2d-0afd14e9ac1e" />


### Physical interface map

| Side A | Side B | Function |
| --- | --- | --- |
| SW1 Ethernet0/0 | SW2 Ethernet0/0 | Po1 member #1 — LACP active/active at baseline |
| SW1 Ethernet0/2 | SW2 Ethernet0/2 | Po1 member #2 — LACP active/active at baseline |
| SW1 Ethernet0/1 | SW3 Ethernet0/0 | Independent 802.1Q trunk |
| SW2 Ethernet0/1 | SW3 Ethernet0/1 | Independent 802.1Q trunk |

**Fourth node:** `TEST SWITCH` is **disconnected** in the exported topology. It is an optional spare/test switch, not part of the three-switch EtherChannel/STP path. Its saved hostname is a placeholder (`inserthostname-here`). You may leave it off or remove it if you only want the three-node scenario.

### Starting settings present in the YAML

- SW1: `spanning-tree mode rapid-pvst`; VLAN 10 priority **4096** and VLAN 20 priority **8192**.
- SW2: `spanning-tree mode rapid-pvst`; VLAN 10 priority **8192** and VLAN 20 priority **4096**.
- SW3: `spanning-tree mode rapid-pvst`; VLAN 10 cost **30** on Ethernet0/0 and cost **10** on Ethernet0/1.
- SW1/SW2: `Port-channel1` and physical members Ethernet0/0, Ethernet0/2 use trunking and allowed VLANs **10,20**; both physical members are `channel-group 1 mode active`.
- All three connected switches use 802.1Q trunks on the listed independent links.

**VLAN availability check:** The exported running-configuration text does **not** explicitly contain global `vlan 10` / `vlan 20` declarations. After import, verify `show vlan brief` and whether the VLANs are active on every relevant switch. If absent, create them before testing. A VLAN appearing in a trunk's **allowed list** does not by itself mean that the VLAN exists and is active.

## Import and baseline verification

1. In CML2, import `ChatGPT___CCNP_EtherChannel__WK03_Labs.yaml` and confirm the four nodes and four links above.
2. Start SW1, SW2, and SW3. The disconnected TEST SWITCH is optional.
3. Verify VLAN 10 and VLAN 20 exist and are active as required. Add missing VLANs with `vlan 10` and `vlan 20` in global configuration mode.
4. Confirm the intended initial LACP active/active and trunk configurations on SW1 and SW2.
5. On SW2, inspect:

   ```text
   show etherchannel summary
   show lacp internal
   show lacp neighbor
   show interfaces trunk
   show spanning-tree vlan 10
   show spanning-tree vlan 20
   ```

**Expected baseline (conditional on VLANs and image):** Po1 reports Layer 2/in use (`SU`) and Ethernet0/0 and Ethernet0/2 are both bundled (`P`). In the recorded Week 03 lab, SW2 selected Po1 as its VLAN 10 root port at root-path cost **56**; SW2 Ethernet0/1 was Alternate/Blocking for VLAN 10. Treat numeric STP costs as **observed values**, not guaranteed cross-platform constants.

## Fault injection exercises

**Exercise 1 — LACP active/passive**

- On **SW2 Ethernet0/2**, configure `channel-group 1 mode passive` while SW1 Ethernet0/2 remains **active**.
- Predict the member flag before checking.
- **Expected:** active/passive can negotiate; Ethernet0/2 remains bundled (`P`). Return SW2 Ethernet0/2 to **active** when finished.

**Exercise 2 — LACP passive/passive**

- On **both SW1 and SW2 Ethernet0/2**, set `channel-group 1 mode passive`; leave Ethernet0/0 active/active on both switches.
- Inspect `show etherchannel summary`, `show lacp internal`, `show lacp neighbor`, and `show spanning-tree vlan 10`.
- **Observed in the Week 03 CML lab:** SW2 Po1 remained `SU` via Ethernet0/0 (`P`), Ethernet0/2 became suspended (`s`), and LACP partner information was absent/zero for Ethernet0/2. The VLAN 10 Po1 root-path cost increased **56 → 100**, while the alternate route via SW3 cost **130** and remained less preferred.
- **Recover:** Restore `channel-group 1 mode active` on Ethernet0/2 **on both switches** and verify both members return to `P`.

**Exercise 3 — Member compatibility / native VLAN**

- Introduce a **lab-only** native VLAN override on **SW2 Ethernet0/2** (`switchport trunk native vlan 99`), leaving the Po1/other member baseline unchanged.
- **Observed in the Week 03 CML lab:** Ethernet0/2 suspended while Ethernet0/0 kept Po1 operational. IOS warned VLAN 99 was not found; no VLAN 99 was created.
- **Recover:** On SW2 Ethernet0/2, `no switchport trunk native vlan`. Verify the member bundles again and check trunks and STP.

**Exercise 4 — Full bundle outage and STP alternate path**

- Administratively shut **both** SW2 Po1 members. Inspect Po1 and the VLAN 10 root port.
- **Observed:** Po1 was down (`SD`) and SW2 switched to Ethernet0/1 as Root/Forwarding for VLAN 10 with total cost **130**.
- **Recover:** `no shutdown` both members and verify Po1 `SU`, both `P`, and the preferred root path is restored.

## Verification and troubleshooting checklist

- **Logical vs. physical state:** `show etherchannel summary` — `Po1(SU)` does not prove *all* intended members are healthy.
- **Negotiation evidence:** `show lacp internal` for local mode/state; `show lacp neighbor` for learned partner information.
- **Configuration comparison:** `show running-config interface Ethernet0/0`, `Ethernet0/2`, and `Port-channel1` on both ends.
- **Trunk eligibility:** `show interfaces trunk` — distinguish VLANs **allowed**, **allowed and active**, and **STP forwarding/not pruned**.
- **Loop-free topology:** `show spanning-tree vlan 10` and `show spanning-tree vlan 20` to inspect root port, cost, roles, and states.
- **Root cause evidence:** `show logging` and member-interface error counters as needed.
- **Data-plane verification:** If you attach test endpoints, verify actual bidirectional traffic; `Po1(SU)`, MAC entries, and STP roles alone do not prove application reachability.

### Troubleshooting guardrails

Do **not** assume (1) passive always fails, (2) a healthy Port-channel means all members are bundled, (3) an allowed VLAN is necessarily forwarding, or (4) a four-member 1 Gbps EtherChannel allows one TCP connection to exceed a single member's speed. Conventional EtherChannel hashing keeps a given flow on one member, while different flows may use aggregate link capacity.

## Scope and limitations

- The preserved YAML is an **exported CML starting state**, not an automated test harness or evidence of all exercises having been performed on a fresh import.
- The Week 03 tests verified EtherChannel and STP operational status in CML. **No measured convergence time, end-to-end host ping, or single-TCP-flow throughput benchmark** was recorded.
- The three-switch setup is the meaningful network topology; the fourth TEST SWITCH is disconnected.
- Interface capabilities and commands, including `lacp min-links`, depend on the Cisco platform/image. In the original IOL-L2 image, `lacp ?` under Po1 exposed `max-bundle` but not `min-links`.

## Learning goal

Explain why an EtherChannel remains operational with one bad member; isolate LACP and trunk compatibility faults from STP decisions; verify recovery with **both** control-plane/operational output and, when endpoints exist, data-plane traffic.
