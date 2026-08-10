# PMR446 GeoConfig: Decentralized Spatial Channel-Allocation for PMR446 Radios

Welcome to **PMR446 GeoConfig** — a modern, decentralized, and geographically aware standard operating protocol and visual explorer for analog PMR446 radio equipment.

The goal of this project is to map any physical coordinate on Earth to a stable, compatible set of channel and squelch tone configurations without relying on central registries, database lookups, or network synchronization. 

---

## 1. What is PMR446 GeoConfig?

**PMR446** (Private Mobile Radio, $446\text{ MHz}$) is a license-free personal radio service used widely across Europe for personal, professional, recreational, and emergency communication. However, because it operates on a limited set of shared frequencies, operators frequently suffer from co-channel interference or find themselves unable to establish communication due to misaligned squelch settings (CTCSS tones).

**PMR446 GeoConfig** solves this. By dividing the physical world into discrete cells via **Geohashing**, calculating proximity with **Haversine geometry**, and assigning default configurations using **stable modular tessellation**, compliant radio devices or operators can instantly calculate a compatible configuration profile using only geographic coordinates (or a Maidenhead QTH Locator).

### Key Features
- 🌐 **Infrastructure Free**: Requires no internet, databases, or central coordination. Perfect for emergency, downtime, and off-grid scenarios.
- 📻 **Legacy Compatible**: Works 100% with standard analog PMR446 equipment using the 304 legacy configurations (8 channels $\times$ 38 CTCSS tones).
- 🧩 **Prevents Congestion & Propagation**: By establishing compatibility using overlapping local channel sets rather than enforcing uniform channels everywhere, distant operators can reuse the same frequencies while adjacent areas remain fully cooperative.
- 📱 **Mobile-First Visual Explorer**: A responsive, PWA-installable map app that operates completely offline on your smartphone or PC.

---

## 2. The Golden Operating Rules (SOP)

To allow radio operators to establish communication completely ad-hoc and without prior coordination, PMR446 GeoConfig introduces two standard operating procedures:

### Rule 1: The Standby (Monitoring) Rule
Every active operator **MUST** set their radio receiver to monitor their local cell's **Primary Standby** configuration by default.
> *Think of this as your geographical calling "inbox." Your radio remains quiet from static and distant noise, but any nearby operator can reach you by calling your cell's designated Primary frequency.*

### Rule 2: The Outbound Calling Rule
To contact operators in the same cell, use **your** current **Primary** configuration.
To call operators in a nearby cell, temporarily tune your transmitter and receiver to **their** cell's **Primary** configuration, call them, and wait for them to answer.
> *Once contact is successfully established, the two operators could stay in the current frequency/tone or agree to move to any other frequency/tone*

```
   [Cell A]                                                     [Cell B]
   Primary Standby: CH3 / 77.0 Hz                               Primary Standby: CH5 / 88.5 Hz
   Compatible Sets: {3/77.0, 5/88.5}                            Compatible Sets: {5/88.5, 3/77.0}
   
   Operator A is listening on CH3/77.0                          Operator B is listening on CH5/88.5
   
   To Call B: Operator A temporarily tunes to CH5/88.5, initiates the call, and B answers.
   To Call A: Operator B temporarily tunes to CH3/77.0, initiates the call, and A answers.
```

---

## 3. Repository File Structure

The project has been simplified to separate human-oriented quickstart and operating details from raw technical implementation specifics:

```text
.
├── README.md               # Unified human-oriented landing page & user guide (this file)
├── AGENTS.md               # Unified agent-oriented mathematical specification & algorithms
└── web/                    # Standalone Visual Specification Explorer (Web App)
    ├── index.html          # Interactive responsive map dashboard
    ├── manifest.json       # PWA Application manifest (Installable Web App)
    ├── sw.js               # Service worker for offline use & tile caching
    └── icon.svg            # Custom modern vector PWA logo
```

---

## 4. The Interactive Visual Explorer

To allow engineers, designers, and hobbyists to explore the protocol's spatial channel allocations globally in real-time, we provide an interactive, browser-based **Visual Explorer**.

### Accessing the Explorer
- 💻 **Offline Local Access**: Open the file **[`web/index.html`](file:///Users/denismaggiorotto/Documents/Progetti/Personali/repos/pmr446-geoconfig/web/index.html)** directly in any modern web browser.
- 🚀 **Live Hosted Access**: Accessible worldwide via GitHub Pages directly from the repository's main domain.

### Key Explorer Features
- **Map Interaction**: Click or tap anywhere on the dark-theme worldwide map to select a physical coordinate.
- **Auto-Computation**: Instantly displays the location's Maidenhead Locator (QTH), Geohash cell, primary configuration, and its local neighborhood compatibility graph.
- **Parameters Customization**: Drag the sidebar range sliders to see how Geohash precision, radio radius, and set size ($K$) dynamically modify allocations on-the-fly.
- **Mobile Optimized Layout**: Includes responsive bottom navigation tabs, touch gestures, and collapsible cards-based stats tailored for smartphone screens.
- **PWA Capabilities**: Save the page directly to your phone's home screen as a standalone application. Maps and tiles are cached dynamically by our stale-while-revalidate service worker for offline use in the field!

---

## 5. Practical Quickstart for Radio Operators

Here is a step-by-step example of how to use PMR446 GeoConfig in practice:

1. **Find Your Location**: Check your GPS or Maidenhead Locator (e.g. `JN45AB`).
2. **Consult the Explorer**:
   - Open the web app and search for `JN45AB` or click your coordinate.
   - It will display your cell's **Primary Standby**: e.g., `CH3 / CTCSS 77.0 Hz`.
   - Set your PMR446 transceiver's Channel to **3** and CTCSS tone to **77.0 Hz**. Keep it on standby.
3. **Contacting a Neighbor**:
   - If you want to contact a station in a nearby town located in cell `JN45AC` (whose Primary is, say, `CH5 / CTCSS 88.5 Hz`), tune your radio to **Channel 5, CTCSS 88.5 Hz**.
   - Press Push-To-Talk (PTT) and make your call: *"This is Station A calling JN45AC. Do you copy?"*
   - Once they answer, agree to switch back to your primary channel (`CH3 / 77.0 Hz`) or any other mutually compatible configuration in your sets to keep their calling channel clear.

---

## 6. Scientific Reproducibility & Contributing

For AI coding assistants, compilers, embedded systems engineers, and programmatic developers, please consult **[`AGENTS.md`](file:///Users/denismaggiorotto/Documents/Progetti/Personali/repos/pmr446-geoconfig/AGENTS.md)**. It contains the official, strict mathematical specification, standard JSON export schemas, regional test area bounding boxes (such as `IT-BBOX-01`), and complete algorithmic pseudocode for full independent reproducibility.
