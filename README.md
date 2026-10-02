# Crimson Guard Transport Ship in Ambient Traffic

<p align="center">
  <img src="https://img.shields.io/badge/OpenGOAL-Mod-blue.svg" alt="OpenGOAL Mod">
  <img src="https://img.shields.io/badge/Game-Jak%202-orange.svg" alt="Game">
  <img src="https://img.shields.io/badge/AI--assisted-Modding-purple.svg" alt="AI Assisted">
</p>

---

> [!NOTE]
> This mod moved from the `jak2/features/transport-ag/traffic` branch of [whozghiar/jak-project](https://github.com/whozghiar/jak-project) to this repository. Earlier releases stay installable from the launcher catalog.

## 📖 Overview
Integrates transport-v, an authentic Crimson Guard troop transport gunship into Haven City's ambient high-altitude traffic lanes, pilotable by Jak with a functional turret, chasing during alerts, and hovering to drop squads.

- **Target Game:** Jak 2
- **Repository:** [`whozghiar/jak2-mod-transport-ag-traffic`](https://github.com/whozghiar/jak2-mod-transport-ag-traffic)

## ✨ Key Features
- **Ambient High-Altitude Gunship:**  Dual-hull troop transport navigating city flight lanes with seated pilot and minimap icon.
- **Player Hijacking & Turret Controls:**  Leap onto the hull to eject the guard, take the helm, and fire the nose turret (R1).
- **Alert Pursuits & Troop Drop:**  Pursues Jak during city alerts, locks altitude in place, opens rear hatch, and drops invulnerable guards.
- **Persistent Turret & Realistic Crash:**  Turret child process synchronized with the traffic pool, with LOD, and unlocked tumble physics upon fatal damage.

## 🚀 Step-by-Step Guide to Run the Mod

### 1. Select the Active Game
Make sure your environment is targeting Jak 2:
```bash
task set-game-jak2
```

### 2. Binary Compilation
- **Status:** Required (Layer 1 & Layer 2 — Decompiler & Runtime)
- **Details:** Compiles the runtime, compiler, and decompiler required for asset extraction:
```bash
task build-release-game
task build-release-decomp
```

### 3. Asset Extraction
- **Status:** Custom extraction required (Layer 2)
- **Details:** Re-run extraction to process custom assets and modified decompiler configuration:
```bash
task extract
```

### 4. Launch the Game
Run the game natively:
```bash
task boot-game
```
*(Or iterate fast via the OpenGOAL REPL using `task repl`, then hot-reload with `(mi)` and `(r)`).*

### 5. Enable the Mod (OFF by default)
This mod ships **disabled** — a fresh install has stock Haven City traffic (no
transport gunship) and stock Crimson Guard behaviour. Open the in-game
Mods menu with **L3 + SELECT** (works in retail boot, no debug mode required):

```
Mods ▸ transport-ag-traffic ▸ Enable
```

The choice persists across level reloads and **takes effect immediately**
(ambient traffic pools are recycled in real-time via `'kill-all` and `'spawn-all`,
so troop transports appear without having to reload the city). Turn it off to
restore vanilla traffic immediately.

## 🎥 Demonstration Video
[![Demonstration Video](https://img.youtube.com/vi/MnqnybexhSA/maxresdefault.jpg)](https://youtu.be/MnqnybexhSA)

▶️ **[Watch the demonstration video on YouTube](https://youtu.be/MnqnybexhSA)**

## 📖 Technical Documentation
For the complete technical breakdown, architecture, and developer notes, refer to:
- 📄 [`docs/modding/current_mod/transport_traffic_readme.md`](docs/modding/current_mod/transport_traffic_readme.md)

---
*(AI-assisted)*
