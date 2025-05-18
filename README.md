# Project Zircon: Distributed Storage Cluster

## Membership

Membership in Project Zircon is based on contribution to the cluster's resources. The goal of this cluster is to provide safe and reliable backup to all memebers. Understand that this network is support by home networks with the goal of shared storage. Here are some basic requirements for members:
1. Members are not allowed to use this service for illegal services or activities. 
2. Members buckets maximum size will be limited, do not exceed those limits.
3. Limit transfer speeds to 100MiB/s. Limit network traffic to less than 10TB per month.

In addition, all members are expected to adhere to the following guidelines:  

### Contribution Requirements

To be an active member, individuals or entities must fulfill one of the following:

*   **Contribute Storage:** Provide raw storage capacity to the cluster through dedicated nodes or by expanding existing ones.
*   **Borrow Storage:** Arrange to utilize storage allocated to another member, with mutual agreement.

For Example:
> **Storage Contribution Formula:** The calculation is straightforward - take the maximum amount of data you wish to store and multiply it by 2.5. This represents the raw storage you need to provide, borrow, or donate.
> 
> *Example:* If a member provides 25TB of raw storage, they would be allocated a maximum bucket size of 10TB.
> 
> *Rationale:* When data is stored on the network, it is replicated so that there are two copies distributed. In the event of a node failure, data will be automatically re-replicated to maintain redundancy while the cluster undergoes repair. This approach ensures your data remains protected during failure scenarios, and includes overhead so that none of the nodes are ever 100% full.

### Public Allocation

A public ledger will be maintained to track each member's contributions. Their allocated storage usage will be visible on the bucket size limits.

*   **Storage Allocation:** Members will be allocated storage based on their contributions.
*   **Exceeding Allocation:** If a member using more storage than their allocated amount will have their API access keys switched to read-only mode.
*   **Public Endpoints:** Public endpoints are hosted on public cloud which charge $20 per 1TB of network traffic. If you use the public endpoint you will need to cover your network costs.

### Responsibilities for Storage Providers

Members who contribute storage by providing nodes have the following responsibilities:

*   **Node Uptime:** Storage providers must make a best-effort attempt to ensure their contributed nodes remain online and functional.
*   **Network Performance:** The node should have a low latency connection to the NAT and internet.
    * **Ping:** Less than 50ms to ensure responsive connections
    * **Bandwidth:** Minimum 150MiB/s for efficient upload and download
    * **Data Cap:** Minimum 5 TB monthly data allowance
*   **Node Performance:** The node must provide reasonable data speeds and low latency.
    * **Read:** Atleast 500 MiB/s
    * **Write:** Atleast 250 MiB/s
    * **Data Storage:** XFS, ZFS, BRTFS configured as a pool, volume or individual disks. You must ensure that your stated capacity and disk space align, ie - don't allow other applications to consume the capacity you are sharing with the pool.
    * **Metadata Storage:** ZFS, BTRFS configured as a pool or volume. This will store the config, node data, database, and database snapshots. Supports either LMDB or SQLite and needs to be backed up.

### Data Retention Policy

*   **Uncovered Data:** Any data stored on the cluster that is not covered by a member's active storage contribution will be subject to deletion after a period of three (3) months.
*   **Node Failure:** In the event of a catastrophic node failure, data that is not covered by sufficient contributions may also be subject to loss, as per the cluster's overall resilience and replication strategy.

## Network Summary

Project Zircon is a collaborative, distributed storage cluster designed for resilience and large-scale data management. It operates on a 2x replication factor, meaning every piece of active data (`A`) is stored twice across the cluster (`A * 2`). The cluster is designed to tolerate the failure of any **single largest node**, requiring enough total raw storage to reconstitute the data from that failed node. Members contribute raw storage capacity through dedicated nodes or expanding existing ones with hard drives or donations.

This document outlines the system architecture, storage calculations, and recovery buffer strategy.

## Storage Replication and Recovery Buffer

Project Zircon maintains a **2x replication factor** for all active data (`A`). This ensures data redundancy during normal operation.

To handle the failure of the **single largest node** in the cluster, the cluster must maintain sufficient **total raw storage (`S_total`)** to absorb the data from the **single largest node (`T_largest_node`)** and allow for its data to be replicated again to restore the 2x factor across all active data.

**Calculation for Minimum Total Cluster Raw Storage (`S_required`):**
The minimum total raw storage required across all zones is calculated as:
`S_required = (A * 2) + T_largest_node`

Where:
*   `A` = Desired total active data footprint.
*   `A * 2` = Raw storage needed for 2x replication of active data.
*   `T_largest_node` = Total raw storage capacity of the single largest node in the cluster. This represents the cluster-level recovery buffer needed.

Conversely, given the current total raw storage (`S_total`) and the largest node (`T_largest_node`), the maximum supportable active data footprint (`A_max`) is:
`A_max = (S_total - T_largest_node) / 2`

The cluster's **Total Available Storage** (`S_total - (A * 2)`) must always be greater than or equal to `T_largest_node`.

## Future Cluster Plan

Based on the current total raw storage (`S_total` = 328 TB) and the largest node (nova1/nova2, `T_largest_node` = 64 TB), the maximum supportable active data footprint (`A_max`) is 132 TB.

*   **Largest Node:** `nova1`/`nova2` (64 TB each)
*   **Required Cluster-Level Buffer (for single largest node failure):** 64 TB
*   **Maximum Supported Active Data (`A_max`):** 132 TB
*   **Implied Total Used Storage (`A_max * 2`):** 264 TB
*   **Current Total Available Storage (Buffer):** 328 TB - 264 TB = 64 TB
*   **Buffer Requirement Met:** Yes (64 TB >= 64 TB)

The following table shows the distribution based on `A_max = 64 TB`:

| Zone | Node Name      | Total Raw Storage (TB) | Available Storage (TB)    | Reserve Storage (TB)  | Storage Credit (TB) |
| :--- | :------------- | :--------------------- | :------------------------ | :-------------------- | :------------------ |
| pae1 | terra          | 48                     | 38.6                      | 9.4                   | 19                  |
| pae1 | helios         | 56                     | 45.1                      | 10.9                  | 22                  |
|      | **Zone Total** | **104**                | **83.7**                  | **20.3**              | **41**              |
| sea1 | nova1          | 64                     | 51.5                      | 12.5                  | 25                  |
| sea1 | nova2          | 64                     | 51.5                      | 12.5                  | 25                  |
|      | **Zone Total** | **128**                | **103.0**                 | **25.0**              | **50**              |
| oak1 | luna           | 48                     | 38.6                      | 9.4                   | 19                  |
|      | **Zone Total** | **48**                 | **38.6**                  | **9.4**               | **19**              |
| pdx1 | eva            | 48                     | 38.6                      | 9.4                   | 19                  |
|      | **Zone Total** | **48**                 | **38.6**                  | **9.4**               | **19**              |
| ---  |                |                        |                           |                       |                     |
|      | **Total**      | **328**                | **264.0**                 | **64.0**              | **132**             |


## System Map

### High-Level Cluster Overview

```mermaid
graph TD
    subgraph "Project Zircon Storage Cluster"
        direction LR
        PAE1["Zone: pae1<br>Total: 104TB<br>Avail: 83.7TB<br>RSVD: 20.3TB"]
        SEA1["Zone: sea1<br>Total: 128TB<br>Avail: 103.0TB<br>RSVD: 25.0TB"]
        OAK1["Zone: oak1<br>Total: 48TB<br>Avail: 38.6TB<br>RSVD: 9.4TB"]
        PDX1["Zone: pdx1<br>Total: 48TB<br>Avail: 38.6TB<br>RSVD: 9.4TB"]
    end
    
    PAE1 --- SEA1
    SEA1 --- OAK1
    OAK1 --- PDX1
    PDX1 --- PAE1
    PAE1 --- OAK1
    SEA1 --- PDX1

    ClusterTotal["Total Cluster Raw Storage: 328TB<br>Total Used (A*2): 264TB<br>Total RSVD (Buffer): 64TB<br>Max Active Data (A): 132TB"]
```

### Zone: pae1 (104TB Total)

```mermaid
graph TD
    subgraph "Zone pae1"
        direction LR
        Terra["Node: terra<br>Total: 48TB<br>Avail: 38.6TB<br>RSVD: 9.4TB"]
        Helios["Node: helios<br>Total: 56TB<br>Avail: 45.1TB<br>RSVD: 10.9TB"]
        
        ZoneStats["Zone Statistics<br>Total: 104TB<br>Avail: 83.7TB<br>RSVD: 20.3TB"]
    end
    
    Terra --- Helios
```

### Zone: sea1 (128TB Total)

```mermaid
graph TD
    subgraph "Zone sea1"
        direction LR
        Nova1["Node: nova1<br>Total: 64TB<br>Avail: 51.5TB<br>RSVD: 12.5TB"]
        Nova2["Node: nova2<br>Total: 64TB<br>Avail: 51.5TB<br>RSVD: 12.5TB"]
        
        ZoneStats["Zone Statistics<br>Total: 128TB<br>Avail: 103.0TB<br>RSVD: 25.0TB"]
    end
    
    Nova1 --- Nova2
```

### Zone: oak1 (48TB Total)

```mermaid
graph TD
    subgraph "Zone oak1"
        direction LR
        Luna["Node: luna<br>Total: 48TB<br>Avail: 38.6TB<br>RSVD: 9.4TB"]
        
        ZoneStats["Zone Statistics<br>Total: 48TB<br>Avail: 38.6TB<br>RSVD: 9.4TB"]
    end
```

### Zone: pdx1 (48TB Total)

```mermaid
graph TD
    subgraph "Zone pdx1"
        direction LR
        Eva["Node: eva<br>Total: 48TB<br>Used: 38.6TB<br>RSVD: 9.4TB"]
        
        ZoneStats["Zone Statistics<br>Total: 48TB<br>Used: 38.6TB<br>RSVD: 9.4TB"]
    end
```

### Inter-Zone Data Pathways

```mermaid
graph TD
    PAE1["pae1<br>(104TB)"] --- SEA1["sea1<br>(128TB)"]
    SEA1 --- OAK1["oak1<br>(48TB)"]
    OAK1 --- PDX1["pdx1<br>(48TB)"]
    PDX1 --- PAE1
    
    PAE1 --- OAK1
    SEA1 --- PDX1
    
```

## Footnotes and Additional Information

-  **Storage Units**: All storage capacities and usage are presented in Terabytes (TB).
-  **Total Raw Storage**: The total physical storage capacity contributed by a node or zone.
-  **Implied Available Storage**: Calculated based on the maximum supported active data (`A_max` = 132 TB) replicated twice (`A_max * 2` = 264 TB total) and distributed proportionally across nodes/zones based on their `Total Raw Storage`. This represents the usage if the cluster were fully utilized up to its maximum supportable active data limit.
-  **Implied Reserved Storage**: Calculated as `Total Raw Storage - Implied Available Storage`. The sum of this across all zones constitutes the cluster-level recovery buffer, which must be at least equal to the size of the **single largest node** (64 TB).
-  **Inter-Zone Links**: The connections depicted in the system map represent logical data pathways or primary replication relationships. The actual network topology might be more intricate.
-  **Member Contribution**: Member contributions must collectively ensure the cluster\'s `Total Raw Storage` is sufficient to meet the `S_required = (A * 2) + T_largest_node` formula for the desired active data footprint (`A`).
-  **Data Distribution**: The `client-sync` script facilitates managing data movement and backup operations between nodes and zones, contributing to the overall resilience strategy of Project Zircon.
