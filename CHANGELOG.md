```
                               .:---==--------==--:.
                         .-----.....................-----..                            _  _ __  __   _   ___
                     :-=--...............................:-==-.                       | \| |  \/  | /_\ | _ \
                 .---:.:..:.....::--=------------::......:....----.                   | .` | |\/| |/ _ \|  _/
              .-=-:..-..:.:---==-------===-=--------==---..:....:.--:.                |_|\_|_|  |_/_/ \_\_|
           :---....-..::---:---------=======::::--------:---::..:.:..=--.             / __(_)___ __| |_ __ _
       ..---:.:..:.:--:::::----:-------=****=-=-----:---::::::--:.:.:..----.          \__ \ / -_|_-<  _/ _` |
     :---::..:..---::::::::--:::-----+##*++*##+----::::--::::::.:---..:..::--:.       |___/_\___/__/\__\__,_|
  .---..:..:.:--:.........:-:::::---+**illmob**----:::::::..........--:::..:.:=-:
 :--...:.:--::.............:::.::---=**+*++*+**---:::::::..............:--..::.:=-:
:--:.::-----...............::::::::-:=***++***=---:::::::..............------:..:--:
:-:::=------:..............:::::::::----=++=----::::::::...............:-----==::--:
 .:-====--::::=---:..........:::::::::--------:::::::::.........:-----:-:--=-===-..
        ..:-===--:-:-=-::.....::::::::::::::::::::::::.....---=-:-:+-===:..
               .:-===::::-=---..:-::::::::::::::::-:.:----=::--===-..
                    .-+==::::::----------:-----------::::::+==:.
                       .:=+-=::::::::::::::::::::::::::-==-.
                           .:===-=---:::::::::---====--.
                                  .:===------=--..
```

# Nmap Siesta — Changelog

**A stealth-scheduling fork of [Nmap](https://github.com/nmap/nmap)**

[![Base](https://img.shields.io/badge/base-Nmap%207.98-blue?style=flat-square)](https://nmap.org)
[![Fork](https://img.shields.io/badge/fork-Siesta-orange?style=flat-square)](#)
[![Author](https://img.shields.io/badge/by-illmob-red?style=flat-square)](#)
[![License](https://img.shields.io/badge/license-NPSL-green?style=flat-square)](LICENSE)

</div>

---

## 📋 Overview

**Nmap Siesta** is built on top of upstream Nmap `7.98` (commit [`f87c5e20b`](https://github.com/nmap/nmap/commit/f87c5e20b)) and adds a complete **scan scheduling and stealth-timing system** without modifying the core scanning logic. The scheduling hooks only gate *whether* probes are sent — never *how* they're constructed.

| Metric | Value |
|:--|--:|
| Files modified | 11 |
| Files added | 8+ |
| Lines added | ~2,900 |
| Lines removed | ~490 |

---

## 🏷️ Version: 7.98 Siesta

> **Released:** February 2026  
> **Base:** Nmap 7.98SVN (upstream master)

---

### 🎨 Branding & Identity

- **Version renamed** from `7.98SVN` → `7.98 Siesta` across both `nmap` and `ncat` binaries

- **Custom ASCII art banner** displayed on every scan launch — skull logo with "NMAP Siesta" branding and author credit

- **Version display** now reads:
  ```
  Nmap 7.98 Siesta By: illmob and ( https://nmap.org )
  ```

<details>
<summary>📁 Files changed</summary>

| File | Change |
|:--|:--|
| `nmap.h` | `NMAP_SPECIAL` → `" Siesta"` |
| `ncat/ncat.h` | `NCAT_VERSION` → `"7.98Siesta"` |
| `nmap.cc` | Version display string updated |
| `nmap_siesta_art.h` | **NEW** — ASCII art banner header (31 lines) |
| `nmap.cc` | Banner printed to stdout before scan starts |

</details>

---

### ⏰ Scan Scheduling Engine — *The Big One*

A complete **scan pause / resume / throttle system** allowing Nmap to operate within defined time windows, with multiple layers of stealth-oriented timing controls.

#### New CLI Option

```
--schedule-file <filename>    YAML config for scan pause/resume scheduling
```

> If `schedule.yaml` exists in the current directory, it is loaded **automatically**.

#### Features at a Glance

| # | Feature | Description |
|:-:|:--|:--|
| 1 | **Blackout Windows** | Recurring (day-of-week) or one-shot (specific date/time) periods where scanning pauses completely |
| 2 | **Adaptive Throttle** | Slow scan rate during restricted hours instead of full pause |
| 3 | **Human-Like Rhythms** | Burst + rest patterns to mimic organic network traffic |
| 4 | **Randomized Micro-Pauses** | Random delays between probe bursts for IDS/IPS evasion |
| 5 | **CIDR-Based Rules** | Host/network-specific pause and throttle rules |
| 6 | **Calendar Awareness** | Weekday / weekend / specific-date scheduling |
| 7 | **Hot-Reload** | Auto-reloads config if changed during a running scan |
| 8 | **Timezone Support** | Evaluate schedule windows in the target's timezone |

#### Core Engine Integration

The scheduler is woven into all probe-sending paths to ensure **zero packets leak** during blackout windows:

```
ultra_scan() main loop
  ├── pauseIfNeeded()          ← blocks until blackout ends
  ├── throttleIfNeeded()       ← adjusts timing if in throttle window
  │
  ├── doAnyNewProbes()
  │   ├── [blackout guard]     ← returns immediately if in blackout
  │   ├── applyRhythm()        ← burst/rest pattern
  │   └── applyMicroPause()    ← random inter-probe delay
  │
  ├── doAnyRetryStackRetransmits()
  │   └── [blackout guard]
  │
  ├── doAnyPings()
  │   └── [blackout guard]
  │
  ├── doAnyOutstandingRetransmits()
  │   └── [blackout guard]
  │
  └── service_scan()
      ├── pauseIfNeeded()
      └── launchSomeServiceProbes()
          └── [blackout guard]  ← returns 0 probes if in blackout
```

#### Interactive Runtime Key

Press **`s`** during a running scan to display schedule status:
- Active/inactive windows
- Next pause/resume time
- Current throttle state

The `?` help menu has been updated to document this key.

#### Schedule Auto-Loading

```
Priority: --schedule-file argument  >  auto-detect schedule.yaml in CWD
```

If no `--schedule-file` is given but a `schedule.yaml` exists in the working directory, it is loaded automatically with an informational message.

<details>
<summary>📁 New files</summary>

| File | Lines | Purpose |
|:--|--:|:--|
| `nmap_schedule.h` | 288 | Class definition, data structures, enums |
| `nmap_schedule.cc` | 1,158 | Full implementation — YAML parser, time matching, blackout enforcement, throttle logic, micro-pauses, rhythm engine |
| `schedule.yaml` | 296 | Full reference config with all features documented |
| `stealth_schedule.yaml` | 176 | Pre-configured stealth profile (business-hours blackouts, conservative timing) |
| `docs/SCHEDULE_README.md` | ~700 | Comprehensive documentation |

</details>

<details>
<summary>📁 Modified files</summary>

| File | Change |
|:--|:--|
| `NmapOps.h` | Added `NmapScheduler scheduler`, `schedule_file`, `schedule_enabled` members |
| `NmapOps.cc` | Initialize schedule fields in `NmapOps::Initialize()` |
| `nmap.cc` | `--schedule-file` option registration, parsing, auto-detection, and startup enforcement |
| `scan_engine.cc` | Blackout guards in all 4 probe-sending functions + main scan loop |
| `service_scan.cc` | Blackout guards in `launchSomeServiceProbes()` and `service_scan()` |
| `nmap_tty.cc` | `s` key handler, help text update, schedule enforcement on keyboard poll |

</details>

---

### 🔧 Build System

- **`Makefile.in`** — Added `nmap_schedule.cc` / `.h` / `.o` to `SRCS`, `HDRS`, and `OBJS` lists

- **`liblinear/Makefile`** — Added `liblinear.a` static library target:
  ```makefile
  liblinear.a: linear.o newton.o blas/blas.a
      cp blas/blas.a liblinear.a
      ar r liblinear.a linear.o newton.o
      ranlib liblinear.a
  ```

- **`ncat/configure`** — Regenerated autoconf script (autotools regeneration, not manual edits)

---

### 📦 Deployment & Distribution

| File | Purpose |
|:--|:--|
| `nmap-deploy.sh` | **NEW** (230 lines) — Build & package script for creating portable Nmap Siesta tarballs |
| `nmap-portable/` | **NEW** — Pre-built portable distribution with `nmap`, `ncat`, all data files, NSE scripts, schedule configs, and `run-nmap.sh` launcher |
| `nmap-7.98-siesta-portable.tar.gz` | **NEW** — Packaged portable tarball ready to deploy |

---

### 📊 Diff Summary

```diff
 Makefile.in        |    6 ±
 NmapOps.cc         |    2 +
 NmapOps.h          |    7 +
 liblinear/Makefile |    5 +
 ncat/configure     | 1158 ±±±±±±±±±±±±±±±±±±±±±±±±
 ncat/ncat.h        |    2 ±
 nmap.cc            |   40 ±
 nmap.h             |    2 ±
 nmap_tty.cc        |   13 +
 scan_engine.cc     |   27 +
 service_scan.cc    |    8 +
 11 files changed, 781 insertions(+), 489 deletions(-)
```

**New files not tracked by upstream:**

```
 nmap_schedule.cc          |  1158  ████████████████████
 nmap_schedule.h           |   288  █████
 nmap_siesta_art.h         |    31  █
 schedule.yaml             |   296  █████
 stealth_schedule.yaml     |   176  ███
 nmap-deploy.sh            |   230  ████
 docs/SCHEDULE_README.md   |  ~700  ████████████
 nmap-portable/            |    —   (pre-built distribution)
```

---

### 🏗️ Architecture Notes

The design principle behind Nmap Siesta is **non-invasive augmentation**:

1. **No core scan logic was modified** — the scheduling system only gates whether probes are sent, never how packets are constructed or interpreted

2. **All hooks are guarded** — every integration point checks `o.schedule_enabled && o.scheduler.isActive()` before executing, ensuring zero overhead when scheduling is disabled

3. **Graceful degradation** — if no `schedule.yaml` is present and `--schedule-file` is not used, Nmap behaves identically to upstream

4. **The YAML parser is self-contained** — no external YAML library dependency; lightweight key-value parsing built into `nmap_schedule.cc`

---
