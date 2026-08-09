# PMR446 — Geographic Configuration Algorithm

## Purpose

Develop a deterministic, geographic configuration algorithm for PMR446 radios.

The algorithm assigns a set of PMR446 radio configurations to a geographic location so that nearby stations are likely to share at least one compatible radio configuration, while geographically distant stations can reuse different configurations.

The system must be:

- deterministic;
- geographically aware;
- based on Geohash;
- independent of any central database;
- compatible with legacy PMR446 equipment;
- easy to implement on small devices or embedded systems;
- independently reproducible by different implementations.

The initial target is a maximum nominal communication radius of **10 km**.

This project is an algorithmic research/prototyping project. The implementation must include simulation and verification tools rather than assuming that a particular allocation strategy is mathematically optimal.

---

# 1. PMR446 Radio Model

The algorithm MUST initially operate on the legacy/common PMR446 configuration space:

- 8 PMR446 channels;
- 38 standard CTCSS tones;
- CTCSS only;
- no DCS;
- no proprietary signaling;
- no digital modes.

The complete configuration space therefore contains:

**8 × 38 = 304 configurations.**

A configuration is represented as:

```text
(channel, ctcss_tone)
```

For example:

```text
(CH1, 67.0)
(CH3, 77.0)
(CH7, 123.0)
```

The implementation MUST treat channel and CTCSS as separate dimensions.

Do not assume that channel number and CTCSS number are interchangeable.

---

# 2. CTCSS Tone Set

The initial CTCSS set MUST contain the following 38 standard tones:

```text
67.0
71.9
74.4
77.0
79.7
82.5
85.4
88.5
91.5
94.8
97.4
100.0
103.5
107.2
110.9
114.8
118.8
123.0
127.3
131.8
136.5
141.3
146.2
151.4
156.7
162.2
167.9
173.8
179.9
186.2
192.8
203.5
210.7
218.1
225.7
233.6
241.8
250.3
```

The list MUST be represented as a stable ordered constant.

The implementation MUST NOT silently add:

- DCS codes;
- non-standard CTCSS values;
- digital signaling;
- proprietary tones.

---

# 3. Geographic Input

The primary geographic input is a geographic coordinate:

```text
latitude
longitude
```

The system MUST also provide a Maidenhead Locator interface because Maidenhead is commonly used by radio amateurs.

The implementation should support:

```text
Maidenhead locator → latitude/longitude → Geohash
```

The internal geographic representation MUST be based on latitude/longitude and Geohash rather than directly using Maidenhead characters as a spatial grid.

Example:

```text
JN45AB
    ↓
latitude / longitude
    ↓
Geohash
    ↓
geographic cell
```

The algorithm MUST NOT assume that Maidenhead cells have uniform metric dimensions.

---

# 4. Geohash

Geohash is used to discretize the geographic space.

The implementation MUST support configurable Geohash precision.

The initial simulation SHOULD use approximately 5-character Geohashes.

However, Geohash precision MUST NOT be hard-coded into the algorithm.

It should be possible to run simulations with different precisions, for example:

```text
4
5
6
7
```

The selected precision must be treated as an algorithm parameter.

---

# 5. Radio Range Model

The initial nominal radio range is:

```text
10 km
```

This is a mathematical simulation parameter, NOT a guarantee of actual PMR446 coverage.

Real PMR446 coverage depends on:

- terrain;
- antenna height;
- buildings;
- vegetation;
- transmitter power;
- receiver sensitivity;
- interference;
- Fresnel clearance;
- regulatory limitations.

The simulator MUST therefore describe 10 km as a configurable theoretical compatibility radius.

The default value is:

```text
radio_radius_km = 10.0
```

The implementation MUST allow this value to be changed.

Examples:

```text
5 km
10 km
15 km
20 km
```

---

# 6. Geographic Graph

The geographic space must be represented as a graph.

Each Geohash cell is a node.

Two cells are connected when their representative geographic points are within the configured radio radius.

For the first implementation, the center point of each Geohash cell may be used as its representative point.

The graph is:

```text
G = (V, E)
```

where:

```text
V = Geohash cells

E = pairs of cells whose geographic distance <= radio radius
```

The distance calculation MUST use a proper geographic distance function, preferably Haversine distance.

Do not calculate radio compatibility using differences in latitude/longitude alone.

---

# 7. Core Compatibility Requirement

The fundamental requirement is:

For two geographic cells A and B:

```text
distance(A, B) <= radio_radius
```

then:

```text
configurations(A) ∩ configurations(B) != ∅
```

In words:

> Any two geographically compatible cells must share at least one radio configuration.

This is the primary invariant that the simulator must verify.

---

# 8. Primary Configuration

Every geographic cell MUST have exactly one PRIMARY configuration.

The primary configuration is the default configuration presented to the user.

Example:

```text
Cell: JN45AB-equivalent Geohash

PRIMARY:
CH3 / CTCSS 77.0
```

Important:

**Nearby cells MUST NOT necessarily have the same primary configuration.**

In fact, the algorithm SHOULD generally attempt to assign different primary configurations to adjacent cells.

The following is explicitly allowed:

```text
Cell A:
PRIMARY = CH3 / 77.0

Cell B:
PRIMARY = CH5 / 88.5
```

even if A and B are only a few kilometres apart.

The compatibility requirement is instead satisfied by their complete configuration sets.

---

# 9. Configuration Set

Each cell has:

```text
configuration_set(cell)
```

The set contains:

```text
PRIMARY
+
NEIGHBOUR configurations
```

For example:

```text
Cell A

PRIMARY:
CH3 / 77.0

NEIGHBOURS:
CH7 / 67.0
CH2 / 94.8
CH6 / 123.0
```

Therefore:

```text
configuration_set(A) =
{
    CH3 / 77.0,
    CH7 / 67.0,
    CH2 / 94.8,
    CH6 / 123.0
}
```

The number of configurations per cell MUST be configurable.

The simulator MUST test different values of:

```text
K = 1
K = 2
K = 3
K = 4
...
```

where K includes the primary configuration.

The objective is to determine the smallest practical K that satisfies the geographic compatibility constraint.

---

# 10. Why Primary Configurations Must Not Be Shared Automatically

Do NOT implement the rule:

```text
distance(A, B) <= 10 km
    →
primary(A) == primary(B)
```

This creates a transitive propagation problem.

For example:

```text
A --10km-- B --10km-- C
```

would imply:

```text
primary(A) = primary(B)
primary(B) = primary(C)
```

and therefore:

```text
primary(A) = primary(C)
```

even if A and C are significantly more distant than the intended communication radius.

Instead, compatibility must be established through configuration-set intersection.

The desired relationship is:

```text
A ∩ B != ∅
```

rather than:

```text
PRIMARY(A) == PRIMARY(B)
```

---

# 11. Primary Assignment

Primary assignment is a graph-coloring-like problem.

Adjacent cells SHOULD receive different primary configurations whenever possible.

The 304 available configurations provide a large palette.

The implementation should use a deterministic graph coloring strategy.

The exact algorithm is not prescribed.

Candidate approaches include:

- greedy graph coloring;
- deterministic DSATUR;
- constrained coloring;
- deterministic pseudo-random ordering;
- hybrid graph-coloring strategies.

The algorithm MUST be deterministic.

Given the same:

```text
geographic input
algorithm version
parameters
```

it MUST produce the same primary assignments.

---

# 12. Determinism

The algorithm MUST NOT depend on:

- database state;
- network services;
- random entropy;
- current time;
- machine-specific state;
- iteration order of unordered data structures.

If pseudo-randomization is useful, it MUST use a deterministic seed derived from stable algorithm inputs.

For example:

```text
seed =
hash(
    algorithm_version,
    geohash,
    configuration_space_version
)
```

The exact seed strategy may be chosen by the implementation.

---

# 13. Neighbour Configuration Assignment

After primary configurations have been assigned, the algorithm must ensure geographic compatibility.

For every graph edge:

```text
A -- B
```

the algorithm must verify:

```text
configuration_set(A) ∩ configuration_set(B) != ∅
```

If the intersection is empty, one or more configurations must be added to one or both cells, subject to the maximum K value.

A simple initial strategy is:

```text
configuration_set(A) += PRIMARY(B)
```

or:

```text
configuration_set(B) += PRIMARY(A)
```

when this does not exceed K.

More advanced optimization strategies may be implemented later.

---

# 14. Optimization Objective

The implementation SHOULD optimize the following objectives in order:

### Hard constraint

Every geographically compatible pair MUST have at least one common configuration.

### Primary optimization

Minimize:

```text
K
```

the number of configurations assigned to each cell.

### Secondary optimization

Minimize:

```text
total number of configurations used
```

### Tertiary optimization

Maximize geographic reuse of configurations.

### Additional optimization

Avoid unnecessary sharing between geographically distant cells where practical.

The optimization algorithm MUST NOT sacrifice the hard compatibility constraint.

---

# 15. Simulation

The project MUST include a simulator.

The simulator should be able to generate a geographic test area and construct the corresponding graph.

At minimum it should support an Italy-sized geographic bounding box.

A real geographic polygon is preferable for future versions, but the initial implementation may use a bounding box.

The simulator MUST report:

```text
number of geographic cells
number of graph edges
maximum node degree
average node degree
number of primary configurations used
```

For each tested K it MUST report:

```text
K
number of uncovered edges
percentage of satisfied edges
maximum configuration-set size
average configuration-set size
```

Example:

```text
K = 1
covered: 72.4%

K = 2
covered: 98.1%

K = 3
covered: 100.0%
```

The exact results are expected to depend on the geographic sampling and algorithm.

Do not hard-code expected numerical results.

---

# 16. Minimum-K Search

The simulator MUST automatically test increasing values of K.

For example:

```text
K = 1
K = 2
K = 3
K = 4
K = 5
...
```

The first K satisfying:

```text
uncovered_edges == 0
```

should be reported as:

```text
minimum_successful_K
```

This value is an empirical result of the selected algorithm and simulation model.

It MUST NOT be presented as a universal mathematical minimum unless proven.

---

# 17. Configuration Encoding

Every configuration should have a stable integer ID.

There are 304 IDs:

```text
0 ... 303
```

A simple deterministic mapping may be:

```text
id = (channel - 1) * 38 + tone_index
```

This results in:

```text
0..37    = CH1
38..75   = CH2
...
266..303 = CH8
```

The implementation MUST provide conversion functions:

```text
config_id(channel, tone)
```

and:

```text
config_from_id(id)
```

The mapping MUST remain stable within a major algorithm version.

---

# 18. API

The implementation should expose a simple high-level API.

Example:

```python
profile = get_radio_profile(
    latitude=45.0703,
    longitude=7.6869
)
```

or:

```python
profile = get_radio_profile_from_locator("JN45AB")
```

The result should contain at least:

```python
{
    "geohash": "...",
    "primary": {
        "channel": 3,
        "ctcss": 77.0
    },
    "configurations": [
        {
            "channel": 3,
            "ctcss": 77.0
        },
        {
            "channel": 7,
            "ctcss": 67.0
        }
    ]
}
```

The API should also expose algorithm metadata:

```python
{
    "algorithm": "GeoPMR446",
    "version": "1.0",
    "geohash_precision": 5,
    "radio_radius_km": 10.0
}
```

---

# 19. CLI

Provide a command-line interface.

Example:

```bash
geopmr446 JN45AB
```

Output:

```text
GeoPMR446 v1.0

Locator: JN45AB
Geohash: ...

Primary:
  CH3 / CTCSS 77.0 Hz

Compatible configurations:
  CH3 / CTCSS 77.0 Hz
  CH7 / CTCSS 67.0 Hz
  CH2 / CTCSS 94.8 Hz
```

The CLI should also support coordinates:

```bash
geopmr446 --lat 45.0703 --lon 7.6869
```

and simulation:

```bash
geopmr446 simulate
```

Useful simulation options:

```bash
--radius 10
--precision 5
--max-k 8
```

---

# 20. Testing

The project MUST have automated tests.

Tests MUST cover:

### Configuration space

Verify:

```text
8 channels
38 CTCSS tones
304 configurations
```

### Configuration mapping

Verify that every ID maps uniquely to one configuration.

### Geographic calculations

Verify:

- Geohash encoding;
- Geohash decoding;
- center calculation;
- Haversine distance.

### Determinism

Calling the allocator twice with identical input MUST produce identical results.

### Compatibility

For every simulated edge:

```text
configuration_set(A) ∩ configuration_set(B)
```

must not be empty.

### Non-neighbour cells

The algorithm is NOT required to guarantee communication between cells outside the configured radius.

### Boundary conditions

Test:

- exact radius;
- slightly below radius;
- slightly above radius;
- Geohash cell boundaries;
- northern/southern latitude extremes used by the simulator.

---

# 21. Property-Based Testing

Where practical, use property-based testing.

Important properties include:

```text
len(CONFIGURATIONS) == 304
```

and:

```text
distance(A, B) <= radius
→ intersection(A, B) != empty
```

for every generated compatible pair.

The test suite should also verify deterministic output across repeated executions.

---

# 22. Reproducibility

A simulation result MUST be reproducible.

A simulation should expose all parameters necessary to reproduce it:

```text
algorithm version
geohash precision
radio radius
geographic bounds
configuration-space version
K
```

The simulator should optionally export its result as JSON.

Example:

```json
{
  "algorithm": "GeoPMR446",
  "version": "1.0",
  "geohash_precision": 5,
  "radio_radius_km": 10.0,
  "configuration_count": 304,
  "minimum_successful_k": 3
}
```

The numerical value of `minimum_successful_k` must be generated by the simulation, not hard-coded.

---

# 23. Performance

The initial implementation should prioritize correctness and clarity over extreme performance.

However, do NOT implement an O(N²) geographic graph construction if the number of cells becomes large.

Use a spatial index or candidate-neighbour strategy where appropriate.

Possible approaches:

- KD-tree;
- spatial hashing;
- Geohash neighbour expansion;
- latitude/longitude buckets;
- R-tree.

The final algorithm should be capable of running a country-scale simulation in a reasonable amount of time on a normal desktop computer.

---

# 24. Separation of Concerns

The project should be divided into logical modules.

Suggested structure:

```text
src/
    geopmr446/
        __init__.py
        config.py
        geohash.py
        geography.py
        graph.py
        allocator.py
        profile.py
        maidenhead.py
        cli.py
        simulator.py

tests/
    test_config.py
    test_geohash.py
    test_geography.py
    test_graph.py
    test_allocator.py
    test_determinism.py
    test_simulation.py

docs/
    algorithm.md
    simulation.md

examples/
    basic.py

pyproject.toml
README.md
AGENTS.md
```

The exact structure may be changed if a better architecture is justified.

---

# 25. Important Algorithmic Distinction

Do not confuse these three concepts:

### Geographic proximity

```text
distance(A, B) <= 10 km
```

### Primary configuration

```text
primary(A)
```

### Communication compatibility

```text
configuration_set(A) ∩ configuration_set(B) != ∅
```

They are deliberately different.

The purpose of the algorithm is to transform geographic proximity into configuration-set compatibility without forcing primary configurations to be equal.

---

# 26. Future Extensions

The implementation should be designed so that the following can be added later:

- different radio-radius models;
- terrain-aware propagation;
- elevation data;
- real Italian geographic polygons;
- other countries;
- other PMR standards;
- 16-channel PMR devices;
- additional CTCSS configurations;
- DCS;
- mobile/roaming use;
- dynamic configuration;
- weighted geographic density;
- population-based optimization;
- interference-aware optimization;
- frequency/channel reuse analysis.

Do not implement these features unless required by the initial specification.

However, avoid architectural decisions that would make them impossible.

---

# 27. Important Radio Engineering Disclaimer

The algorithm does NOT guarantee that two stations will physically communicate.

The 10 km value is a design parameter used to establish a geographic compatibility graph.

Actual radio coverage depends on environmental and radio conditions.

The algorithm guarantees only the logical property:

```text
if two simulated geographic nodes are considered neighbours,
their assigned configuration sets contain at least one common
channel/CTCSS combination.
```

It does not guarantee RF propagation.

---

# 28. Versioning

The algorithm must expose a version.

Initial version:

```text
GeoPMR446 v1.0
```

Changes that modify:

- configuration mapping;
- CTCSS ordering;
- Geohash strategy;
- primary allocation;
- neighbour allocation;
- compatibility semantics;

should result in an algorithm version change.

The version must be included in exported profiles and simulation results.

---

# 29. Acceptance Criteria

The implementation is considered successful when all of the following are true:

1. It supports the 8 × 38 PMR446 configuration space.
2. It accepts latitude/longitude.
3. It accepts Maidenhead locators.
4. It converts geographic coordinates to Geohash.
5. It builds a geographic compatibility graph.
6. It uses a configurable nominal radio radius, defaulting to 10 km.
7. It assigns exactly one deterministic primary configuration per cell.
8. Adjacent cells are not forced to have the same primary configuration.
9. Each cell can have a configurable number K of total configurations.
10. The simulator finds the first K for which all compatible edges have at least one shared configuration.
11. The result is deterministic.
12. The simulator reports meaningful statistics.
13. Automated tests verify the core compatibility invariant.
14. No central database or network service is required.
15. The complete project can be reproduced from the documented algorithm and parameters.

---

# 30. Development Priority

Implement the project in the following order:

### Phase 1 — Radio model

Implement:

```text
8 channels
38 CTCSS
304 configurations
```

### Phase 2 — Geography

Implement:

```text
latitude/longitude
Maidenhead
Geohash
Haversine
```

### Phase 3 — Graph

Implement:

```text
Geohash cells
neighbour detection
10 km radius
```

### Phase 4 — Primary allocator

Implement deterministic graph coloring.

### Phase 5 — Compatibility allocator

Implement configuration-set expansion and verify:

```text
A ∩ B != ∅
```

for every compatible pair.

### Phase 6 — Optimization

Run:

```text
K = 1...
```

and determine the first successful K.

### Phase 7 — CLI/API

Expose the allocator to users.

### Phase 8 — Documentation

Document:

- algorithm;
- assumptions;
- simulation methodology;
- results;
- limitations.

---

# 31. Engineering Principle

Do not optimize prematurely.

The first implementation must be:

- simple;
- deterministic;
- testable;
- inspectable;
- mathematically verifiable.

The algorithm should produce enough intermediate data to allow a developer to answer:

> "Why did this QTH receive this channel and CTCSS combination?"

The allocation process should therefore be explainable rather than being an opaque hash function.

---

# 32. Final Goal

The final system should make it possible for a user to enter a radio amateur QTH such as:

```text
JN45AB
```

and obtain a deterministic PMR446 configuration profile such as:

```text
GeoPMR446 v1.0

Location:
JN45AB

Primary:
CH3 / CTCSS 77.0 Hz

Additional compatible configurations:
CH7 / CTCSS 67.0 Hz
CH2 / CTCSS 94.8 Hz
CH6 / CTCSS 123.0 Hz
```

Two users located within the configured geographic compatibility radius should have at least one configuration in common, while their primary configurations may remain different.

The central design principle is:

> **Geographic proximity creates configuration overlap, not necessarily identical primary configurations.**

The algorithm must use this principle throughout its implementation and optimization.