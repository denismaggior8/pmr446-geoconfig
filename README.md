# GeoPMR446: Geographic Configuration Protocol for PMR446 Radios
## Official Standard Proposal and Protocol Specification

Welcome to the official repository for the **GeoPMR446 Geographic Configuration Protocol**. 

This repository contains the formal technical standard and protocol specification for decentralized, deterministic, and geographically aware allocation of channels and sub-audible CTCSS squelch tones on PMR446 radio equipment.

---

## 1. Introduction and Protocol Purpose

PMR446 is a license-free personal radio service used widely across Europe. Due to the limited spectral space (8 frequency channels), users frequently suffer from overlapping co-channel interference or find themselves unable to communicate because of mismatched sub-audible CTCSS squelch codes. 

**GeoPMR446** resolves these limitations. By dividing the physical world into discretized cells via **Geohashing**, calculating proximity with **Haversine geometry**, and assigning default configurations using **SHA-256 stable hashing**, compliant radio devices can instantly calculate a compatible configuration profile using only geographic coordinates (or a Maidenhead Locator).

### Key Features
- **Deterministic**: Given identical coordinates and version parameters, the protocol always yields identical configurations on any platform.
- **Database and Network Free**: No network connections or central registration databases are required. Perfect for emergency, backcountry, or small embedded microcontrollers.
- **Legacy Compatible**: Operates strictly within standard, existing analog PMR446 constraints (8 channels $\times$ 38 CTCSS tones = 304 discrete channels).
- **Prevents Transitive Propagation**: By establishing compatibility through intersection sets ($\mathcal{C}(u) \cap \mathcal{C}(v) \ne \emptyset$) rather than requiring uniform primary channels, distant cells can reuse frequencies while adjacent cells remain co-operative.

```
A (CH3/77.0) <=== Link ===> B (CH5/88.5) <=== Link ===> C (CH2/123.0)
   (Cell A)                  (Cell B)                  (Cell C)
  Set: {3/77.0, 5/88.5}     Set: {5/88.5, 3/77.0, 2/123.0}    Set: {2/123.0, 5/88.5}
  
  Link A-B compatibility: Shared (5/88.5)
  Link B-C compatibility: Shared (2/123.0)
  Link A-C compatibility: No shared configuration (Transitive propagation prevented!)
```

---

## 2. Repository File Structure

This repository is structured strictly as an **official standardization proposal**, containing detailed mathematical formulations, protocol invariants, and verified algorithmic pseudocode.

```text
.
├── AGENTS.md               # Original protocol research parameters & rules
├── README.md               # Main landing page & protocol overview (this file)
└── docs/
    ├── specification.md    # Technical Protocol Specification (mathematics & algorithms)
    └── simulation.md       # Compliance, Simulation, & Metric Guidelines
```

---

## 3. High-Level Mathematical Framework

Below are the foundational formulas of the standard. For a complete mathematical treatment, read the [Technical Protocol Specification](file:///Users/denismaggiorotto/Documents/Progetti/Personali/repos/pmr446-geoconfig/docs/specification.md).

### 3.1 Bijective Configuration Indexing
The 304 channels in the legacy analog space are defined as $C \times T$. An integer ID between 0 and 303 is assigned using:
$$f(c, t_i) = (c - 1) \times 38 + i$$

### 3.2 Spherical Haversine Distance
The geographic compatibility distance between cell centers $(lat_1, lon_1)$ and $(lat_2, lon_2)$ is determined using standard mean spherical Earth radius $R_{\text{earth}} = 6371.0088\text{ km}$:
$$a = \sin^2\left(\frac{lat_2 - lat_1}{2}\right) + \cos(lat_1)\cos(lat_2)\sin^2\left(\frac{lon_2 - lon_1}{2}\right)$$
$$d = 2 \cdot R_{\text{earth}} \cdot \arcsin(\sqrt{a})$$

### 3.3 Stable Hash Primary Allocation
To assign a default configuration to a Geohash string deterministically without checking external state, the protocol maps the cell using a SHA-256 stable hash:
$$\text{Primary}(u) = \text{SHA256}(\text{"1.0:"} + u + \text{":1.0"}) \bmod 304$$

---

## 4. Reference Pseudocode Sample: Local Profile Resolution

Standalone receivers and handheld transceiver microcontrollers compute their configuration profiles locally using the following algorithm.

```text
Algorithm: ResolveLocalProfile
Input: lat, lon, precision, radius_km, K
Output: profile_dictionary

Let cell_geohash = EncodeGeohash(lat, lon, precision)
Let primary_id = GetPrimaryByHash(cell_geohash)

If K <= 1:
    Return { geohash: cell_geohash, primary: primary_id, configurations: [primary_id] }

# Determine local search bounds using cell dimensions
Let step_lat_deg = 2.0 * cell_latitude_error
Let step_lon_deg = 2.0 * cell_longitude_error

Initialize neighbors = EmptySet()
For each (test_lat, test_lon) in localized_bounding_box_steps:
    Let gh = EncodeGeohash(test_lat, test_lon, precision)
    If gh != cell_geohash:
        Let gh_lat, gh_lon = CenterCoordinates(gh)
        Let dist = HaversineDistance(lat, lon, gh_lat, gh_lon)
        If dist <= radius_km:
            neighbors.add( (dist, gh) )

Sort neighbors by distance ascending

Initialize allocated_configs = [primary_id]
Initialize seen_configs = {primary_id}

For each (dist, gh) in neighbors:
    If length(allocated_configs) >= K:
        Break
    Let neigh_primary = GetPrimaryByHash(gh)
    If neigh_primary not in seen_configs:
        Add neigh_primary to allocated_configs
        Add neigh_primary to seen_configs

Return {
    geohash: cell_geohash,
    primary: primary_id,
    configurations: allocated_configs
}
```

---

## 5. Further Reading

- [docs/specification.md](file:///Users/denismaggiorotto/Documents/Progetti/Personali/repos/pmr446-geoconfig/docs/specification.md): Technical details of geohash parsing, Haversine, primary allocators, coloring, and neighbor expansion algorithms.
- [docs/simulation.md](file:///Users/denismaggiorotto/Documents/Progetti/Personali/repos/pmr446-geoconfig/docs/simulation.md): Complete testing frameworks, Italy bounding boxes, and compliance evaluation rules.
