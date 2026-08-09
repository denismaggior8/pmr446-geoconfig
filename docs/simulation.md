# PMR446 GeoConfig: Simulation and Compliance Validation Guidelines
## Protocol and Metrics Specification
**Version:** 1.0.0  
**Status:** Proposal / Draft Standard  

---

## Abstract

This specification defines the official guidelines for simulating, validating, and verifying standard-compliant implementations of the **PMR446 GeoConfig Geographic Configuration Protocol**. It details geographic test areas, graph density evaluations, optimization metrics, and standard formats for JSON data exchange.

---

## 1. Objectives of Simulation

Because the PMR446 GeoConfig protocol is decentralized and operates on a limited channel space, compliance validation requires testing geographic connectivity at scale. 

Simulations are designed to:
1. **Verify Invariant Compliance**: Ensure that every compatible radio link within a simulated region has at least one common channel configuration in its endpoints' configuration sets.
2. **Determine Spectral Bounds**: Evaluate the minimum configuration-set size ($K$) required to satisfy the geographic constraints of a specific topography.
3. **Analyze Spatial Density**: Profile graph degree properties (average/maximum degrees) to assess channel collision probability.

---

## 2. Standard Test Areas

Compliant simulation engines MUST support standard geographical test benches.

### 2.1 The Italy Bounding Box (IT-BBOX-01)
The primary standard test area is the **Italy Bounding Box (IT-BBOX-01)**, which represents a highly dense, elongated horizontal/vertical landscape with variable maritime and alpine boundaries.

The bounding coordinates are defined as:
- **Minimum Latitude**: $35.5^\circ \text{ N}$
- **Maximum Latitude**: $47.1^\circ \text{ N}$
- **Minimum Longitude**: $6.6^\circ \text{ E}$
- **Maximum Longitude**: $18.5^\circ \text{ E}$

```
                  47.1° N (Northern Alps)
             +-----------------------+
             |                       |
             |                       |
  6.6° E     |         ITALY         |     18.5° E
  (West)     |      (IT-BBOX-01)     |     (East)
             |                       |
             |                       |
             +-----------------------+
                  35.5° N (Lampedusa)
```

### 2.2 Simulation Cell Generation
To conduct a test, compliant simulators must generate node distributions using either:
1. **Regular Geographic Grid**: Vertices are generated uniformly at specific steps (e.g., $\Delta_{\text{lat}} = 0.2^\circ$, $\Delta_{\text{lon}} = 0.2^\circ$) inside `IT-BBOX-01`.
2. **Pseudo-Random Distribution**: Vertices are sampled uniformly across `IT-BBOX-01` using a deterministic seed (default seed = `42`) to produce exactly $N$ unique Geohash cells.

---

## 3. Metrics and Evaluation Protocol

For any test run, the simulation engine MUST calculate and output the following metrics.

### 3.1 Graph Metrics
Before evaluating channel allocations, the simulator must calculate the following metrics:
- **Cell Count ($N$)**: The number of unique Geohash vertices.
- **Edge Count ($M$)**: The number of unique links where physical distance $\le R_{\text{radio}}$.
- **Maximum Degree ($\Delta(G)$)**: The maximum number of neighbors adjacent to any cell.
- **Average Degree ($\bar{d}(G)$)**:
  $$\bar{d}(G) = \frac{2M}{N}$$

### 3.2 Allocation Invariant Metrics (per K)
For increasing set sizes $K = 1, 2, 3 \ldots 8$, the simulator evaluates:
- **Uncovered Edges ($U_K$)**:
  $$U_K = \Big| \big\{ (u, v) \in E \mid \mathcal{C}(u) \cap \mathcal{C}(v) = \emptyset \big\} \Big|$$
- **Satisfied Link Percentage ($P_K$)**:
  $$P_K = \left( 1 - \frac{U_K}{M} \right) \times 100\%$$
- **Average Set Size ($\text{Set}_{\text{avg}}$)**: The average number of active configurations allocated per cell (which should be $\le K$).

### 3.3 The Minimum Successful K Search
The simulation MUST run iteratively starting at $K=1$. The search stops at the first $K$ where the compatibility invariant is fully satisfied. This value is recorded as:
$$K_{\text{successful}} = \min \{ K \in \mathbb{N} \mid U_K = 0 \}$$

---

## 4. Standard Export Format (JSON)

To guarantee scientific reproducibility and allow cross-implementation verification, compliant simulators must be able to export results to a JSON file matching the following schema.

### 4.1 JSON Schema

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
    "primary_strategy": { "type": "string", "enum": ["coloring", "hash"] },
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
