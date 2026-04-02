# Multi-Area OSPF, Route Redistribution, and Inter-VLAN Routing

## 1. Aim
To design, configure, and verify a large-scale enterprise topology consisting of 6 routers and 12 end-hosts (VPCs). The network integrates Inter-VLAN Routing (Router-on-a-Stick), Multi-Area OSPF (Area 0 and Area 1), RIPv2, and two-way Route Redistribution to achieve full end-to-end connectivity across disparate routing domains.

## 2. Topology Overview
The topology consists of a 6-router linear backbone connecting three distinct logical domains:

* **OSPF Area 1 (Branch):** R1 and R2. 
    * *Feature:* R1 handles Inter-VLAN routing for PC1 (VLAN 10) and PC2 (VLAN 20).
* **OSPF Area 0 (Backbone):** R3 and R4. 
    * *Feature:* R2 acts as the Area Border Router (ABR), passing summary routes between Area 1 and Area 0.
* **RIPv2 Domain (Legacy):** R5 and R6.
    * *Feature:* R6 handles Inter-VLAN routing for PC11 (VLAN 30) and PC12 (VLAN 40).
* **Redistribution Point:** R4 acts as the Autonomous System Boundary Router (ASBR), injecting RIP routes into OSPF and OSPF routes into RIP.

## 3. Theory & Mechanics

### Multi-Area OSPF
In large networks, a single OSPF area causes the Link-State Database (LSDB) to become excessively large, draining CPU and memory. By dividing the network into areas (Area 0 and Area 1), routers only calculate the SPF algorithm for their local area. Area Border Routers (ABRs) summarize traffic between these areas.

### Route Redistribution
Routers running different protocols (OSPF and RIP) cannot natively share routing tables because they use different metrics (Cost vs. Hop Count). Route redistribution is configured on the ASBR to translate these metrics. 
* RIP injected into OSPF appears as an **External Type 2 (O E2)** route.
* OSPF injected into RIP requires a manually defined "seed metric" (e.g., Hop Count 2) so RIP understands how far away the routes are.

## 4. Configuration Highlights (The Translators)

**R2 (Area Border Router - ABR):**
Interfaces in different OSPF areas.
> `R2(config-router)# network 195.120.150.128 0.0.0.31 area 1`
> `R2(config-router)# network 195.120.150.160 0.0.0.31 area 0`

**R4 (Autonomous System Boundary Router - ASBR):**
Injecting routes between protocols.
> `R4(config)# router ospf 1`
> `R4(config-router)# redistribute rip subnets`
> `R4(config)# router rip`
> `R4(config-router)# redistribute ospf 1 metric 2`

## 5. Verification & Showcase (Proof of Execution)

To demonstrate the success of this Mega-Lab to the examiner, the following commands verify the three core objectives:

### A. Verifying Inter-VLAN Routing
* **Execution:** Ping from PC1 (VLAN 10) to PC2 (VLAN 20) on Island 1.
* **Command:** `PC1> trace 195.120.150.18`
* **Inference:** The packet hits the R1 sub-interface gateway (`.1`) before routing down to the separate VLAN, proving Layer 3 separation on the switch.

### B. Verifying Multi-Area OSPF
* **Execution:** Check the routing table on R1 (inside Area 1).
* **Command:** `R1# show ip route ospf`
* **Inference:** Routes belonging to Area 0 (like the R3-R4 link) will appear with the code **`O IA`** (OSPF Inter-Area). This proves R2 is successfully summarizing between Area 0 and Area 1.

### C. Verifying Route Redistribution
* **Execution:** Check the routing tables at the extreme ends of the network (R1 and R6).
* **Command (on R1):** `R1# show ip route ospf`
* **Inference:** The RIP subnets (from R5/R6) appear as **`O E2`** (OSPF External Type 2) routes, proving R4 injected them into OSPF.
* **Command (on PC1):** `PC1> trace <PC12_IP_Address>`
* **Inference:** An end-to-end trace from the OSPF VLAN domain, through the backbone, crossing the ASBR, and terminating in the RIP VLAN domain proves full topology convergence.
