# PMR446 GeoConfig — Geographic Configuration Algorithm Design

This document describes the design, mathematics, and implementation details of the **PMR446 GeoConfig Geographic Configuration Algorithm**.

## 1. PMR446 Configuration Space

Standard legacy/analog PMR446 radios operate on a set of 8 designated frequency channels in the $446.0 - 446.1\text{ MHz}$ band and support standard Continuous Tone-Coded Squelch System (CTCSS) tones to isolate communications from interference.

### Configurations
A single radio configuration is defined as a discrete tuple of:
$$\text{config} = (\text{channel}, \text{ctcss\_tone})$$

Where:
- $\text{channel} \in [1, 8]$
- $\text{ctcss\_tone} \in \text{CTCSS\_TONES}$ (a static, ordered set of 38 standard CTCSS tones from $67.0\text{ Hz}$ to $250.3\text{ Hz}$).

The complete configuration space contains:
$$8 \times 38 = 304 \text{ distinct configurations}$$

### Stable Configuration ID Mapping
To enable fast, stable, and deterministic operations, configurations are mapped bijectively to integer IDs from $0$ to $303$:
$$\text{id} = (\text{channel} - 1) \times 38 + \text{tone\_index}$$

Where $\text{tone\_index}$ is the 0-indexed position of the CTCSS tone in the static 38-tone list.

---

## 2. Geographic Modeling

### Coordinate Representation
Coordinates are processed in standard decimal degrees Latitude $[-90.0, 90.0]$ and Longitude $[-180.0, 180.0]$ (WGS-84 datum).

### Maidenhead Locator Parsing
The system converts Maidenhead Locators (e.g. `JN45AB` or `JN45`) into WGS-84 coordinates by parsing the alphanumeric locator fields:
- **Field (pair 1)**: Subdivides the globe into $18 \times 18$ grid cells of $20^\circ \text{ lon} \times 10^\circ \text{ lat}$.
- **Square (pair 2)**: Subdivides each field into $10 \times 10$ grid cells of $2^\circ \text{ lon} \times 1^\circ \text{ lat}$.
- **Subsquare (pair 3)**: Subdivides each square into $24 \times 24$ grid cells of $5'\text{ lon} \times 2.5'\text{ lat}$ ($2/24^\circ \times 1/24^\circ$).

The coordinate is mapped to the exact geometric center-point of the highest precision square defined by the locator.

### Geohash Discretization
To discretize continuous physical locations into deterministic area buckets (cells), coordinates are encoded into **Geohashes** using binary interval partitioning.
- The default simulation uses precision 5 (characters), which corresponds to a grid box of approximately $4.9\text{ km} \times 4.9\text{ km}$ at the equator.
- The precision $P$ is fully configurable to support different grid spatial scales.

### Distance Metric (Haversine)
The great-circle distance between two cell center-points $(lat_1, lon_1)$ and $(lat_2, lon_2)$ is computed using the **Haversine formula** to account for Earth's spherical curvature:
$$a = \sin^2\left(\frac{\Delta \phi}{2}\right) + \cos(\phi_1)\cos(\phi_2)\sin^2\left(\frac{\Delta \lambda}{2}\right)$$
$$c = 2 \arcsin(\sqrt{a})$$
$$d = R_{\text{earth}} \times c$$
Where $R_{\text{earth}} = 6371.0088\text{ km}$.

---

## 3. Geographic Graph

A set of Geohash cells $V$ is represented as an undirected graph $G = (V, E)$.
An edge $e = (u, v)$ is established between cells $u$ and $v$ if and only if the physical distance between their geometric centers is within the configured radio radius $R$:
$$E = \{ (u, v) \in V \times V \mid \text{haversine}(u, v) \le R_{\text{radio}} \}$$

### Efficient Graph Construction
To avoid $O(N^2)$ distance calculations when building graphs for large numbers of cells, a **2D Spatial Grid Bucketing** index is used:
1. Divide the space into uniform rectangular buckets of size $D_{\text{lat}} \times D_{\text{lon}}$ where:
   - $D_{\text{lat}} = R_{\text{radio}} / 111.1$ degrees.
   - $D_{\text{lon}} = D_{\text{lat}} / \cos(\text{max\_abs\_lat})$ degrees.
2. For each cell, its bucket indices are computed.
3. Neighbors are queried only from the cell's own bucket and the 8 surrounding buckets.
This guarantees all neighbors within $R_{\text{radio}}$ are captured in $O(N)$ average time.

---

## 4. Configuration Allocation Algorithm

### The Core Invariant (Symmetric Standby-Calling)
Any two geographically adjacent cells (connected in the graph) must support mutual direct standby-calling:
$$\forall (u, v) \in E \implies \text{Primary}(v) \in \mathcal{C}(u) \quad \text{and} \quad \text{Primary}(u) \in \mathcal{C}(v)$$

This guarantees that either operator can immediately tune to and call the other on their designated standby frequency.

### Rule 1: No Transitive Propagation
To avoid a chain reaction where adjacent cells are forced to have identical configurations across long distances, **nearby cells do not share the same Primary configuration**.
$$\text{Primary}(u) \ne \text{Primary}(v) \quad \text{is mathematically guaranteed.}$$
Instead, compatibility is established because each cell's set contains its neighbors' unique primary standby configurations:
$$\mathcal{C}(u) = \{ \text{Primary}(u) \} \cup \{ \text{Primary}(v) \mid (u, v) \in E \}$$

### Primary Assignment Strategies
1. **2D Coordinate-Based Modular Tessellation Grid**:
   Maps any Geohash cell of precision $P$ to its discrete integer coordinates $(X, Y)$ and applies a modular shifting pattern:
   $$\text{Primary}(u) = (Y \cdot 17 + X) \bmod 304$$
   This is local, graph-free, and mathematically guarantees that adjacent cells never share the same primary configuration.
2. **Alphabetical Greedy Graph Coloring**:
   Colors the graph $G$ using the 304-color palette in a stable, alphabetical sorting of Geohashes. This is a global simulation alternative that also ensures local channel differentiation.

### Compatibility Allocation Strategy
To expand configuration sets up to size $K$ to satisfy the standby-calling invariant:
1. Initialize $\mathcal{C}(u) = \{ \text{Primary}(u) \}$ for all $u$.
2. For each cell $u \in V$:
   - Identify all its adjacent neighbor cells $v$ where $(u, v) \in E$.
   - Sort these neighbors by geographic distance (ascending), using Geohash alphabetical order as a stable tie-breaker.
   - For each neighbor $v$ in sorted order:
     - If $|\mathcal{C}(u)| < K$:
       - Add $\text{Primary}(v)$ to $\mathcal{C}(u)$ (if not already present).
     - Else:
       - Stop adding. The remaining further-out neighbors remain incompatible (unsatisfied) for this capacity limit $K$.
