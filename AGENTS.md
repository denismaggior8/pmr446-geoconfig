# PMR446 GeoConfig: AI Coding Agent Guidelines & Core Rules

This document outlines the architectural guidelines, codebase directory rules, and system instructions for AI coding agents working on **PMR446 GeoConfig**.

---

## 1. Directory Structure & File Invariants

AI agents MUST respect the decoupled separation of concerns within the repository:

- **[`README.md`](file:///Users/denismaggiorotto/Documents/Progetti/Personali/repos/pmr446-geoconfig/README.md) (Human-facing)**: Focuses on quickstart guidelines, high-level project introductions, and standard operating procedures (SOP) for radio operators in the field.
- **[`protocol_specification.md`](file:///Users/denismaggiorotto/Documents/Progetti/Personali/repos/pmr446-geoconfig/protocol_specification.md) (Technical specification)**: The single source of truth for the protocol, mathematical formulas, algorithms, test bounding boxes, and JSON schemas.
- **[`AGENTS.md`](file:///Users/denismaggiorotto/Documents/Progetti/Personali/repos/pmr446-geoconfig/AGENTS.md) (Agent instructions - this file)**: Workspace system rules and constraints for AI agents.
- **`web/` (Interactive Client Applications)**:
  - **[`index.html`](file:///Users/denismaggiorotto/Documents/Progetti/Personali/repos/pmr446-geoconfig/web/index.html)**: Main frontend, mobile-responsive dashboard. Uses pure vanilla CSS, Leaflet JS, and Google Fonts.
  - **[`manifest.json`](file:///Users/denismaggiorotto/Documents/Progetti/Personali/repos/pmr446-geoconfig/web/manifest.json)**: Progressive Web App (PWA) manifest properties.
  - **[`sw.js`](file:///Users/denismaggiorotto/Documents/Progetti/Personali/repos/pmr446-geoconfig/web/sw.js)**: Stale-while-revalidate service worker for dynamic offline tile and assets caching.

---

## 2. Core Constraints & Developer Guidelines

When editing the visual explorer or protocol libraries, agents MUST verify:

### 2.1 Tooltip Centering Override (Important CSS/JS Invariant)
To prevent timing-based double-shifting or zero-width render offset bugs on both PC and mobile displays, Leaflet's native position subtraction for centered tooltips has been bypassed.
1. The JS prototype method `L.Tooltip.prototype._setPosition` must NOT subtract widths/heights for `direction === 'center'`.
2. The CSS stylesheet must apply relative translation offsets directly:
   ```css
   .leaflet-tooltip-center,
   .cell-tooltip,
   .active-cell-tooltip {
       margin: 0 !important;
       translate: -50% -50% !important;
   }
   ```
   Do not revert or modify these CSS classes without ensuring positioning calculations remain device-agnostic.

### 2.2 PWA Service Worker Management
- If editing local resource files, verify that both root `./` and `index.html` remain cached in the `ASSETS` cache list inside `web/sw.js`.
- Always check that any navigation fallback in `sw.js` routes to `index.html` during offline mode.

### 2.3 Mathematical Implementation Integrity
All programmatic translations of formulas (Haversine, Geohash parsing, 2D modular color shifting, and local neighborhood expansions) must converge with the official reference pseudocode and bounds detailed in **[`protocol_specification.md`](file:///Users/denismaggiorotto/Documents/Progetti/Personali/repos/pmr446-geoconfig/protocol_specification.md)**.