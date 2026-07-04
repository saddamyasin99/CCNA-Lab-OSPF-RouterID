# CCNA-Lab-Dynamic Routing-OSPF-RouterID
CCNA Lab  , OSPF Router ID: default selection, loopback caching behavior, and manual override in a 3-area topology.
# Lab : OSPF Router ID Configuration 

## 1. Objective
Build on the multi-area OSPF design from Lab 4 and Lab 5, and study **how OSPF assigns and manages its Router ID (RID)** — covering:
- The rule IOS uses to auto-select a Router ID.
- Why adding a loopback interface *after* OSPF starts does not automatically update the RID.
- How to manually and permanently assign a Router ID, and why that's the recommended practice.

## 2. Topology
Same 3-router, single-process (`ospf 200`), 3-area design used in Labs 4/5:

```
        Area 10                    Area 0                     Area 20
   [R-1] ---Se0/1/0--- 200.1.50.0/30 ---Se0/1/0--- [R-2] ---Se0/1/1--- 200.1.60.0/30 ---Se0/1/1--- [R-3]
     |Gi0/0/0                              |Gi0/0/0                                    |Gi0/0/0
192.168.10.0/24                       192.168.20.0/24                            192.168.30.0/24
```

- **R-1** — internal router, Area 10 only.
- **R-2** — **ABR** (Area Border Router), touches Area 10, Area 0 (backbone), and Area 20.
- **R-3** — internal router, Area 20 only.

## 3. IP Addressing Table

| Router | Interface | IP Address | Area | Purpose |
|---|---|---|---|---|
| R-1 | Gi0/0/0 | 192.168.10.3/24 | 10 | LAN |
| R-1 | Se0/1/0 | 200.1.50.1/30 | 10 | WAN to R-2 |
| R-1 | Loopback0 | 1.1.1.1/8 | 10 | RID source |
| R-2 | Gi0/0/0 | 192.168.20.3/24 | 0 | LAN |
| R-2 | Se0/1/0 | 200.1.50.2/30 | 10 | WAN to R-1 |
| R-2 | Se0/1/1 | 200.1.60.1/30 | 20 | WAN to R-3 |
| R-2 | Loopback0 | 2.1.1.1/8 | 0 | RID source |
| R-3 | Gi0/0/0 | 192.168.30.3/24 | 20 | LAN |
| R-3 | Se0/1/1 | 200.1.60.2/30 | 20 | WAN to R-2 |
| R-3 | Loopback0 | 3.1.1.1/8 | 20 | RID source |

## 4. Configuration Walkthrough

### 4.1 Base Interface Configuration (all routers)
Standard `hostname`, `interface`, `ip address`, `no shutdown` on Gi0/0/0 and the relevant Serial interface(s). Minor CLI slips were caught and corrected live during the session (a mistyped `inerface`, an incomplete `ip address` entry) — normal part of hands-on practice.

### 4.2 Initial OSPF Enablement (before loopbacks existed)
```
router ospf 200
 network <WAN subnet> 0.0.0.3 area <area>
 network <LAN subnet> 0.0.0.255 area <area>
```
Run on R-1 (area 10), R-2 (area 10 + area 0, later corrected from a typed `area0`), and R-3 (area 20). At this stage, **no Loopback0 existed on any router**, so OSPF had only physical interfaces to choose a Router ID from.

### 4.3 Default Router ID Observed
`show ip ospf` on each router confirmed the auto-selected RID, based on the **highest active interface IP at the time the process started**:

| Router | Active Interfaces at Startup | → Default Router ID |
|---|---|---|
| R-1 | 192.168.10.3, 200.1.50.1 | **200.1.50.1** |
| R-2 | 192.168.20.3, 200.1.50.2, 200.1.60.1 | **200.1.60.1** |
| R-3 | 192.168.30.3, 200.1.60.2 | **200.1.60.2** |

Note the pattern: on every router, the **Serial (WAN) interface** won out over the Gigabit LAN interface, simply because `200.1.x.x` is numerically higher than `192.168.x.x`. This is IOS comparing raw IP values, not any notion of interface "importance."

### 4.4 Loopback0 Added and Advertised
```
interface loopback0
 ip address X.1.1.1 255.0.0.0
!
router ospf 200
 network X.0.0.0 0.255.255.255 area <area>
```
Configured as `1.1.1.1/8` (R-1, area 10), `2.1.1.1/8` (R-2, area 0), `3.1.1.1/8` (R-3, area 20).

**Result:** `show ip ospf` was re-checked, and the Router ID on every router was **unchanged** — still `200.1.50.1`, `200.1.60.1`, and `200.1.60.2` respectively. This is the core lesson of the lab:

> The OSPF Router ID is selected **once**, when the process first comes up, and is then **cached in memory**. It is not automatically re-evaluated just because a loopback (or any higher IP) appears afterward. Only a `reload` or `clear ip ospf process <id>` forces re-selection.

### 4.5 Router ID Manually Assigned
```
router ospf 200
 router-id X.1.1.1
```
Set on each router to match its own loopback address:

| Router | Manually Assigned Router ID |
|---|---|
| R-1 | 1.1.1.1 |
| R-2 | 2.1.1.1 |
| R-3 | 3.1.1.1 |

Every router immediately displayed the same IOS console warning after this command:
```
Reload or use "clear ip ospf process" command, for this to take effect
```
This confirms that **manually setting** the RID has exactly the same restart requirement as the automatic default — a running OSPF process never silently changes its own Router ID.

## 5. Verification

### 5.1 `show ip ospf`
Used on all three routers to confirm the process name (`ospf 200`), current RID, and area/interface counts. R-2's output specifically confirmed its **ABR role**, showing separate area sections (Area 10, Area 0, Area 20 depending on stage) each with the correct interface count — validating the multi-area design carried over from Lab 4/5.

### 5.2 `show ip route`
Confirmed full end-to-end reachability once all `network` statements and Loopback0 interfaces were advertised:
- `O` routes for directly-adjacent-area routes.
- `O IA` (inter-area) routes for the far router's LAN and loopback, correctly routed via R-2 as the ABR.
- Each router's own Loopback0 shown as a locally connected `/32` (via the `L` code) inside its advertised `/8` supernet.

This confirms the OSPF network design remained fully functional throughout every Router ID transition — Router ID changes affect *identification* of the router in the OSPF domain (LSAs, neighbor tables), not the routing calculation itself.

## 6. Troubleshooting Notes From the Session
- `ip address 2` (incomplete command) — corrected by re-entering the full `ip address` statement.
- `inerface serial0/1/1` typo — corrected to `interface`.
- `network 192.168.20.0 0.0.0.255 area0` — missing space rejected by IOS, corrected to `area 0`.
- All of the above are exactly the kind of real-world CLI slips worth documenting — they show the debugging process, not just the "clean" final config.

## 7. Key Takeaways
1. **Default Router ID rule:** highest loopback IP at OSPF startup, else highest active physical interface IP.
2. **Router ID is cached** — it will not update on its own after a loopback is added or an interface IP changes; only `reload` or `clear ip ospf process` forces re-election.
3. **Manual `router-id`** configuration has the identical restart requirement — it is not applied to a live process.
4. **Best practice:** always assign the Router ID manually, ideally matching a Loopback0 address, so the OSPF domain has stable, memorable, and predictable Router IDs regardless of physical interface changes.
5. This lab's addressing, area design, and ABR role are unchanged from Lab 4/5 — only the Router ID behavior is new, showing how this concept layers cleanly onto a design already in place.

## 8. Conclusion
Lab 6 isolates and confirms one of the most commonly misunderstood details of OSPF: the Router ID is not a "live" property — it's a cached identifier chosen once, changeable only through deliberate manual configuration and a process restart. Anchoring each router's Router ID to its Loopback0 address is the production-standard fix, and this lab reproduces exactly why that standard exists.
