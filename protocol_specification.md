# PMR446 GeoConfig: Technical Protocol Specification

This document serves as the formal technical specification, mathematical description, reference algorithmic pseudocode, and compliance verification standard for **PMR446 GeoConfig**.

---

# 1. PMR446 Radio Model & Configuration Space

Compliant implementations MUST operate strictly within the legacy/common PMR446 configuration space:
- 8 PMR446 frequency channels (12.5 kHz spacing).
- 38 standard CTCSS tones.
- No DCS, digital modes, or proprietary signaling.

The complete configuration space $\mathcal{S}$ is:
$$\mathcal{S} = C \times T \quad \text{where} \quad |\mathcal{S}| = 8 \times 38 = 304$$

A configuration is represented as:
$$s = (\text{channel}, \text{ctcss-tone})$$

### 1.1 Frequency Channels ($C$)
Let $C$ be the set of standard PMR446 channels:
$$C = \{1, 2, 3, 4, 5, 6, 7, 8\}$$

The physical carrier frequencies corresponding to each channel number are defined as:
- **CH1**: 446.00625 MHz
- **CH2**: 446.01875 MHz
- **CH3**: 446.03125 MHz
- **CH4**: 446.04375 MHz
- **CH5**: 446.05625 MHz
- **CH6**: 446.06875 MHz
- **CH7**: 446.08125 MHz
- **CH8**: 446.09375 MHz

### 1.2 CTCSS Tone Set ($T$)
The CTCSS set $T$ MUST be represented as a stable, ordered constant containing the following 38 standard sub-audible tone squelch frequencies in Hertz:
```text
[
  67.0,  71.9,  74.4,  77.0,  79.7,  82.5,  85.4,  88.5,  91.5,  94.8,
  97.4, 100.0, 103.5, 107.2, 110.9, 114.8, 118.8, 123.0, 127.3, 131.8,
  136.5, 141.3, 146.2, 151.4, 156.7, 162.2, 167.9, 173.8, 179.9, 186.2,
  192.8, 203.5, 210.7, 218.1, 225.7, 233.6, 241.8, 250.3
]
```

### 1.3 Stable Configuration Encoding
Every configuration is bijectively mapped to a stable integer ID in the range $0 \dots 303$:
$$\text{config-id}(c, t_i) = (c - 1) \times 38 + i$$

The decoding inverse function is defined as:
$$\text{config-from-id}(\text{id}) = \left( \lfloor \text{id} / 38 \rfloor + 1, ~ T_{\text{id} \bmod 38} \right)$$

This mapping MUST remain stable and immutable across all version implementations.

---

# 2. Geographic Input and Spatial Modeling

### 2.1 Coordinate System
The standard geographic datum is WGS-84. Coordinates are represented as:
- Latitude: $[-90.0, 90.0]$
- Longitude: $[-180.0, 180.0]$

### 2.2 Maidenhead Locator Interface
When receiving a Maidenhead Locator, the system MUST decode it to its exact coordinate center-point before discretization.
- **Fields (character pair 1)**: Subdivides the globe into $18 \times 18$ cells of $20^\circ \text{ longitude} \times 10^\circ \text{ latitude}$.
- **Squares (digit pair 2)**: Subdivides each field into $10 \times 10$ cells of $2^\circ \text{ longitude} \times 1^\circ \text{ latitude}$.
- **Subsquares (character pair 3)**: Subdivides each square into $24 \times 24$ cells of $5'\text{ longitude} \times 2.5'\text{ latitude}$.

### 2.3 Geohash Discretization
Geohashing is used to discretize continuous geographic space. Precision $P$ (character length) is treated as an algorithm parameter and MUST NOT be hard-coded.
- Default precision for nominal calculations is $P=5$ ($\approx 4.9\text{ km} \times 4.9\text{ km}$ at the equator).

### 2.4 Distance Metric
To compute physical distance $d$ between coordinates, implementations MUST use the **Haversine formula**:
$$a = \sin^2\left(\frac{\phi_2 - \phi_1}{2}\right) + \cos(\phi_1)\cos(\phi_2)\sin^2\left(\frac{\lambda_2 - \lambda_1}{2}\right)$$
$$c = 2 \arcsin(\sqrt{a})$$
$$d = R_{\text{earth}} \times c$$
where $\phi_1, \phi_2$ are latitudes, $\lambda_1, \lambda_2$ are longitudes, and the Earth radius is $R_{\text{earth}} = 6371.0088\text{ km}$. Do not use latitude/longitude Euclidean projections.

---

# 3. Geographic Graph and Compatibility Invariants

### 3.1 Geographic Graph Connectivity
The discrete geographic space is modeled as an undirected graph $G = (V, E)$ where:
- $V \subset \mathcal{H}_P$: A set of Geohash cells at precision $P$.
- $E$: Edges between cell centers within nominal radio radius:
  $$E = \{ (u, v) \in V \times V \mid \text{dist}(\text{center}(u), \text{center}(v)) \le R_{\text{radio}} \}$$
- Default radius parameter: $R_{\text{radio}} = 10.0\text{ km}$ (fully configurable).

### 3.2 The Core Compatibility Invariant (Symmetric Standby-Calling)
For any two geographically adjacent cells connected in the graph, the following must hold:
$$(u, v) \in E \implies \text{Primary}(v) \in \mathcal{C}(u) \quad \text{and} \quad \text{Primary}(u) \in \mathcal{C}(v)$$
This guarantees that operators in adjacent cells can always contact each other by tuning to the other's designated primary calling channel.

### 3.3 Rule 1: No Transitive Propagation
To avoid transitive channel locking across long ranges, **adjacent cells are forbidden from having the same primary configuration**:
$$\text{Primary}(u) \ne \text{Primary}(v) \quad \text{where } (u, v) \in E$$

---

# 4. Reference Algorithms

### 4.1 Geohash Encoding (`EncodeGeohash`)
```text
Input: latitude, longitude, precision
Output: geohash_string

Let BASE32 = "0123456789bcdefghjkmnpqrstuvwxyz"
Let lat_min = -90.0, lat_max = 90.0
Let lon_min = -180.0, lon_max = 180.0

Initialize geohash = []
Let total_bits = precision * 5
Initialize current_val = 0

For bit_idx from 0 to total_bits - 1:
    If bit_idx is even:
        Let mid = (lon_min + lon_max) / 2
        If longitude >= mid:
            current_val = (current_val << 1) | 1
            lon_min = mid
        Else:
            current_val = (current_val << 1) | 0
            lon_max = mid
    Else:
        Let mid = (lat_min + lat_max) / 2
        If latitude >= mid:
            current_val = (current_val << 1) | 1
            lat_min = mid
        Else:
            current_val = (current_val << 1) | 0
            lat_max = mid

    If (bit_idx + 1) mod 5 == 0:
        Append BASE32[current_val] to geohash
        current_val = 0

Return concatenate(geohash)
```

### 4.2 Primary Configuration Tessellation (`GetPrimaryByTessellation`)
Compliant devices MUST map Geohashes to their primary configuration using this local, graph-free, modular shift pattern. This guarantees adjacent cells receive different primary configuration IDs.
```text
Input: geohash_str
Output: configuration_id (integer 0 to 303)

Let lat, lon, lat_err, lon_err = DecodeGeohash(geohash_str)
Let lat_height = 2.0 * lat_err
Let lon_width = 2.0 * lon_err

Let Y = Round((lat + 90.0) / lat_height)
Let X = Round((lon + 180.0) / lon_width)

Let color_id = (Y * 17 + X) mod 304
If color_id < 0:
    color_id = color_id + 304

Return color_id
```

### 4.3 Local Profile Expansion (`ResolveLocalProfile`)
Computes the complete local profile set $\mathcal{C}(u)$ of size $K$ without building a global regional graph.
```text
Input: lat, lon, precision, radius_km, K
Output: profile_dictionary { geohash, primary, configurations }

Let cell_geohash = EncodeGeohash(lat, lon, precision)
Let primary_id = GetPrimaryByTessellation(cell_geohash)

If K <= 1:
    Return { geohash: cell_geohash, primary: primary_id, configurations: [primary_id] }

Let lat_step = radius_km / 111.1
Let cos_lat = cos(radians(lat))
Let lon_step = lat_step / max(0.01, cos_lat)

Let cell_lat, cell_lon, lat_err, lon_err = DecodeGeohash(cell_geohash)
Let step_lat_deg = 2.0 * lat_err
Let step_lon_deg = 2.0 * lon_err

Let lat_bound = ceiling(lat_step / step_lat_deg) + 1
Let lon_bound = ceiling(lon_step / step_lon_deg) + 1

Initialize neighbors = EmptySet()

For i from -lat_bound to lat_bound:
    For j from -lon_bound to lon_bound:
        Let test_lat = lat + i * step_lat_deg
        Let test_lon = lon + j * step_lon_deg
        
        Let gh = EncodeGeohash(test_lat, test_lon, precision)
        If gh != cell_geohash:
            Let gh_lat, gh_lon = CenterCoordinates(gh)
            Let dist = HaversineDistance(lat, lon, gh_lat, gh_lon)
            If dist <= radius_km:
                neighbors.add( (dist, gh) )

# Sort neighbors ascending by distance, and alphabetically by geohash for stable ties
Sort neighbors

Initialize allocated_configs = [primary_id]
Initialize seen_configs = {primary_id}

For each (dist, gh) in neighbors:
    If length(allocated_configs) >= K:
        Break
    Let neigh_primary = GetPrimaryByTessellation(gh)
    If neigh_primary not in seen_configs:
        Add neigh_primary to allocated_configs
        Add neigh_primary to seen_configs

Return {
    geohash: cell_geohash,
    primary: primary_id,
    configurations: allocated_configs
}
```

### 4.4 Alphabetical Greedy Graph Coloring (`AllocatePrimaryColoring`)
Used by simulators to construct and optimize primary assignments across a global bounded region.
```text
Input: nodes (set of geohashes), adjacency (map of node to neighbor set)
Output: primary_assignments (map of node to config_id)

Let sorted_nodes = SortAlphabetically(nodes)
Initialize primary_assignments = EmptyMap()

For each node in sorted_nodes:
    Let neighbor_colors = EmptySet()
    For each neighbor in adjacency[node]:
        If neighbor in primary_assignments:
            neighbor_colors.add(primary_assignments[neighbor])
            
    Let assigned_color = -1
    For color_id from 0 to 303:
        If color_id not in neighbor_colors:
            assigned_color = color_id
            Break
            
    If assigned_color == -1:
        assigned_color = GetPrimaryByTessellation(node)
        
    primary_assignments[node] = assigned_color

Return primary_assignments
```

---

# 5. Simulation and Metric Guidelines

### 5.1 The Italy Bounding Box (IT-BBOX-01)
Simulators MUST support the standard geographical test benchmark `IT-BBOX-01`:
- **Min Latitude**: $35.5^\circ \text{ N}$
- **Max Latitude**: $47.1^\circ \text{ N}$
- **Min Longitude**: $6.6^\circ \text{ E}$
- **Max Longitude**: $18.5^\circ \text{ E}$

### 5.2 Metrics Framework
Every simulation run MUST calculate:
- **Cell Count ($N$)**: Total unique Geohash vertices.
- **Edge Count ($M$)**: Total unique connected edges ($\le R_{\text{radio}}$).
- **Maximum Degree ($\Delta(G)$)**: Maximum neighbor count.
- **Average Degree ($\bar{d}(G)$)**: $2M/N$.
- **Uncovered Edges ($U_K$)**: $\Big| \big\{ (u, v) \in E \mid \mathcal{C}(u) \cap \mathcal{C}(v) = \emptyset \big\} \Big|$ for a given $K$.
- **Satisfied Link % ($P_K$)**: $(1 - U_K / M) \times 100\%$.
- **Minimum Successful K ($K_{\text{successful}}$)**: $\min \{ K \in \mathbb{N} \mid U_K = 0 \}$.

### 5.3 Performance Optimizations
For large-scale simulations, O(N²) geographic graph comparisons are prohibited. Simulators SHOULD use a **2D Spatial Grid Bucketing** index of step size $D_{\phi} = R_{\text{radio}} / 111.1$ and $D_{\lambda} = D_{\phi} / \cos(\phi_{\max})$ to ensure $O(N)$ average lookup times.

### 5.4 JSON Export Schema
To ensure scientific reproducibility, simulation results exported to JSON MUST validate perfectly against the standard JSON schema:
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "PMR446 GeoConfig-SimulationResult",
  "type": "object",
  "required": [
    "algorithm",
    "version",
    "geohash_precision",
    "radio_radius_km",
    "configuration_count",
    "primary_strategy",
    "graph_metrics",
    "minimum_successful_k",
    "k_steps"
  ],
  "properties": {
    "algorithm": { "type": "string", "const": "PMR446 GeoConfig" },
    "version": { "type": "string" },
    "geohash_precision": { "type": "integer", "minimum": 1 },
    "radio_radius_km": { "type": "number", "minimum": 0.1 },
    "configuration_count": { "type": "integer", "const": 304 },
    "primary_strategy": { "type": "string", "enum": ["coloring", "tessellation"] },
    "graph_metrics": {
      "type": "object",
      "required": [
        "number_of_cells",
        "number_of_edges",
        "max_node_degree",
        "avg_node_degree",
        "primary_configurations_used"
      ],
      "properties": {
        "number_of_cells": { "type": "integer" },
        "number_of_edges": { "type": "integer" },
        "max_node_degree": { "type": "integer" },
        "avg_node_degree": { "type": "number" },
        "primary_configurations_used": { "type": "integer" }
      }
    },
    "minimum_successful_k": { "type": ["integer", "null"] },
    "k_steps": {
      "type": "array",
      "items": {
        "type": "object",
        "required": [
          "k",
          "uncovered_edges",
          "percentage_satisfied",
          "max_config_set_size",
          "avg_config_set_size"
        ],
        "properties": {
          "k": { "type": "integer" },
          "uncovered_edges": { "type": "integer" },
          "percentage_satisfied": { "type": "number" },
          "max_config_set_size": { "type": "integer" },
          "avg_config_set_size": { "type": "number" }
        }
      }
    }
  }
}
```

---

# 6. Verification & Property-Based Testing

Automated testing suites MUST verify the following invariants:
1. **Bijective Mapping**: `config_from_id(config_id(c, t))` matches `(c, t)` exactly for all 304 positions.
2. **Determinism**: Identical parameter arrays (inputs, precision, radius) yield byte-identical primary and configuration set assignments.
3. **Standby-Calling Coverage**: For every edge where $d(A, B) \le R_{\text{radio}}$, check that $\mathcal{C}(A) \cap \mathcal{C}(B) \ne \emptyset$ for $K \ge K_{\text{successful}}$.
4. **Local/Global Convergence**: Profile lists generated locally via `ResolveLocalProfile` must be mathematically equivalent to profile lists queried from the global geographical graph.
