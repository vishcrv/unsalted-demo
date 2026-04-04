# Password Hash Attack & Prevention

Interactive desktop tool that runs real attacks against real hashed databases and shows — with live numbers — why unsalted SHA-256 is broken and how salt, key-stretching, and pepper fix it.


<img width="1365" height="767" alt="image" src="https://github.com/user-attachments/assets/1fecec5d-ed2e-4286-8069-cff0f66c8864" />


---

## Table of Contents

- [What It Shows](#what-it-shows)
- [Setup](#setup)
- [Understanding the UI](#understanding-the-ui)
  - [Application Window](#application-window)
  - [Left Sidebar — Configuration Panel](#left-sidebar--configuration-panel)
  - [Center — Scrolling Log](#center--scrolling-log)
  - [Status Bar](#status-bar)
- [Features — Step by Step](#features--step-by-step)
  - [Run Attack](#1-run-attack)
  - [Run Full Comparison](#2-run-full-comparison)
  - [Apply Prevention](#3-apply-prevention)
  - [Show Graphs](#4-show-graphs)
  - [Breach Simulator](#5-breach-simulator)
  - [Live Crack Demo](#6-live-crack-demo)
  - [Export CSV](#7-export-csv)
  - [Clear Log](#8-clear-log)
- [Controls Reference](#controls-reference)
- [The Four Hashing Systems](#the-four-hashing-systems)
- [The Math](#the-math)
- [Files](#files)
- [Production Note](#production-note)

---

## What It Shows

| System | Attacker success rate | Why |
|---|---|---|
| Plain SHA-256 | ~80-100% | One table cracks every account instantly |
| + Salt | 0% | Unique salt per user makes the table useless |
| + Salt + Stretching | 0% | Table still useless, every guess costs K x more |
| + Salt + Pepper | 0% | Server-side 256-bit secret — 2^256 harder to brute-force |

---

## Setup

Python 3.10+ required.

```bash
pip install -r requirements.txt
python gui.py
```

Or install manually:

```bash
pip install customtkinter matplotlib numpy
python gui.py
```

No additional configuration is needed. Salts, peppers, and databases are all generated on-the-fly in memory.

---

## Understanding the UI

### Application Window

The app uses a Bloomberg-terminal-inspired dark theme with monospace fonts throughout. The window is split into three zones:

```
+--------------------+--------------------------------------+-----------------+
|   CONFIGURATION    |           SCROLLING LOG              |     STATS       |
|                    |                                      |                 |
|   Num Users        |   Attack output, formulas,           |   unsalted  %   |
|   Dict Size        |   verdicts, per-test results         |   salted    %   |
|   Num Tests        |                                      |   stretched %   |
|   Reuse %  ----    |   +------------------------------+   |   peppered  %   |
|   Stretch K ---    |   |  Live graph (per test)       |   |                 |
|                    |   |  Unsalted  -- red            |   |                 |
|   ACTIONS          |   |  Defences  -- green          |   |                 |
|   -----------      |   +------------------------------+   |                 |
|   run attack       |                                      |                 |
|   run full comp.   |                                      |                 |
|   apply prev.      |                                      |                 |
|   show graphs      |                                      |                 |
|   breach sim.      |                                      |                 |
|   live crack demo  |                                      |                 |
|   export csv       |                                      |                 |
|   clear log        |                                      |                 |
+--------------------+--------------------------------------+-----------------+
|  [STATUS INDICATOR]            [PROGRESS BAR]                               |
+-----------------------------------------------------------------------------+
```


### Left Sidebar — Configuration Panel

The sidebar is where you control all experiment parameters and trigger actions.

<img width="236" height="353" alt="image" src="https://github.com/user-attachments/assets/6a58e486-cc7d-4a04-bbed-8d3eb6ae6074" />
<img width="234" height="310" alt="image" src="https://github.com/user-attachments/assets/100820af-a0d9-4c5c-a287-e23b2a010cd0" />


<!-- Replace with a screenshot of the left sidebar -->

**Settings section (top):**

| Control | Default | Range | What it does |
|---|---|---|---|
| Num Tests | 25 | 1-100+ | Number of rounds in a full comparison run |
| Dict Size | 2000 | any positive int | Size of the attacker's password wordlist |
| Reuse % | 0.70 | 0.0-1.0 | Fraction of users picking passwords from the top 20% of the dictionary. Higher = more clustering = worse unsalted results |
| Stretch K | 10 | 1-500 (slider) | SHA-256 iterations per password guess. Each +1K costs the attacker K x more compute but costs the defender only ~1 ms more per login |

**Action buttons (below settings):**

Eight buttons that each trigger a different feature. All long-running operations run in background threads so the GUI stays responsive. Details on each button are in the [Features](#features--step-by-step) section below.

---

### Center — Scrolling Log

The main output area. All results, formulas, verdicts, and status messages appear here as color-coded text:

| Tag Color | Meaning |
|---|---|
| Purple | Section headers |
| Amber/Yellow | Mathematical formulas and warnings |
| Green | Successful defence (0% cracked) |
| Red | Failed defence / vulnerability detected |
| Blue | Informational messages |
| Gray | Dimmed secondary info |

At the bottom of the log area, a **live graph** is embedded. During a full comparison run, it updates after every test round:
- **Red line** — unsalted success rate per test
- **Green line** — average defence success rate (always 0%)
- **Shaded area** — the gap between them (the "vulnerability window")

<img width="652" height="209" alt="image" src="https://github.com/user-attachments/assets/60d17237-e540-4dc4-a5d9-8c07ec22ecde" />


---

### Status Bar

At the bottom of the window:

- **Left:** Status indicator dot + text
  - `IDLE` — ready for input
  - `RUNNING` — attack in progress
  - `COMPARING` — full comparison running
  - `VULNERABLE` — unsalted system compromised
  - `SECURE` — defences held
- **Right:** Progress bar showing % completion during comparison runs

<img width="91" height="32" alt="image" src="https://github.com/user-attachments/assets/aae9207e-e467-4ec2-9184-9857e566ca5e" />
<img width="70" height="30" alt="image" src="https://github.com/user-attachments/assets/892dbc3d-206f-4c47-a6c8-10261d22ced3" />

---

## Features — Step by Step

### 1. Run Attack

**What it does:** Runs a quick unsalted-only rainbow table attack as a baseline.

**How to use:**
1. Set your desired **Dict Size** in the sidebar
2. Click **Run Attack**
3. Watch the log — it builds a rainbow table, attacks the database, and reports time + success rate

**What you'll see:**
- Rainbow table build time
- Number of accounts cracked
- Success rate (typically 80-100%)

<img width="708" height="248" alt="image" src="https://github.com/user-attachments/assets/9f5580e9-5695-444b-9d7c-accdf1a6e3b3" />


---

### 2. Run Full Comparison

**What it does:** The main experiment. Runs `num_tests` rounds, each with randomised parameters (N = 50-500 users, reuse = 50-90%, dict size = 50-100% of base). All four hashing systems are attacked with the same rainbow table each round. The live graph and stats panel update in real time.

**How to use:**
1. Set **Num Tests** (default 25), **Dict Size**, **Reuse %**, and **Stretch K**
2. Click **Run Full Comparison**
3. Watch the log fill with per-round results and the live graph build up
4. Click **Stop** at any time to interrupt early

**What you'll see per round:**

```
  +== TEST 07  N=213  |D|=1847  reuse=68% ========================
  |  [UNSALTED]   W = 1847 x T_h = 0.018s   SR=79%  !
  |  [SALTED]     W = 213x1847xT_h = 3.9s  (x213)   SR=0%  ok
  |  [STRETCHED]  W = 213x1847x10xT_h = 39s  (x2130) SR=0%  ok
  |  [PEPPERED]   W ~ 4.87e+72 years                  SR=0%  ok
  +================================================================
```

**After all rounds complete:**
- Summary statistics are logged
- The stats panel on the right shows final averages
- Data is ready for [Show Graphs](#4-show-graphs) and [Export CSV](#7-export-csv)

<img width="727" height="475" alt="image" src="https://github.com/user-attachments/assets/c8e64e5b-d90b-4ddd-8ef3-c20b36f0706e" />



<img width="809" height="337" alt="image" src="https://github.com/user-attachments/assets/97eb33cc-98f1-4ebe-b6c5-6c64dd9fbe7d" />

---

### 3. Apply Prevention

**What it does:** Prints the mathematical work formulas with your current slider/input values substituted in, plus salt/pepper key-space sizes and a recommendation block. No computation runs — this is a quick reference.

**How to use:**
1. Adjust your settings to the desired values
2. Click **Apply Prevention**
3. Read the output — it shows exactly how much harder each defence makes the attacker's job with your current numbers

<img width="682" height="397" alt="image" src="https://github.com/user-attachments/assets/9b80be40-7aac-456c-b8c5-ab4ae64210e3" />
<img width="637" height="393" alt="image" src="https://github.com/user-attachments/assets/8eb3a94b-408c-46db-9240-df281900ff68" />



---

### 4. Show Graphs

**What it does:** Opens a matplotlib window with six publication-quality graphs. Requires a completed comparison run first.

**How to use:**
1. Run a full comparison first
2. Click **Show Graphs**
3. A new window opens with tabbed/tiled graphs

**The six graphs:**

| Graph | What it shows |
|---|---|
| **Success Rate Comparison** | Bar chart — average success rates across all four systems |
| **Work Formula Validation** | Predicted vs actual attacker work (validates the math) |
| **CIA Properties** | Confidentiality, Integrity, Authentication scores per system |
| **Latency Overhead** | Per-test login cost overhead introduced by each defence |
| **Security Improvement** | Percentage improvement of each defence over unsalted |
| **Work vs Dict Size** | How attacker cost scales with dictionary size, with clustering analysis |

<img width="1064" height="650" alt="image" src="https://github.com/user-attachments/assets/0c8f22fe-0e30-45c7-afd8-e44e5f4629a2" />

<img width="1063" height="653" alt="image" src="https://github.com/user-attachments/assets/5c20243e-6c8d-4802-b109-a5c4c5a25a93" />

<img width="1067" height="652" alt="image" src="https://github.com/user-attachments/assets/4566256c-d32e-4c93-a315-c4bccf06bd30" />



---

### 5. Breach Simulator

**What it does:** Visualises what a stolen database actually looks like — unsalted vs salted, side by side. This makes the clustering problem viscerally obvious.

**How to use:**
1. Click **Breach Simulator**
2. A display appears showing two databases

**What you'll see:**

- **Left (unsalted):** Users with the same password share the exact same hash. Identical rows are highlighted, showing how cracking one hash exposes every account that reused that password.
- **Right (salted):** Every row has a unique salt and a unique hash, even when passwords are identical. No clustering. Each account must be attacked individually.

<img width="1143" height="520" alt="image" src="https://github.com/user-attachments/assets/80c69b8e-9f89-4d3e-ab40-a7c4e7bde5fa" />


---

### 6. Live Crack Demo

**What it does:** The most interactive feature. You type in any password, and the tool walks through five attack phases in real time using actual computed hashes — showing exactly why unsalted fails and defences succeed.

**How to use:**
1. Click **Live Crack Demo**
2. A popup appears — type any password and hit Enter
3. A 900x600 animated window walks through five phases

**The five phases:**

| Phase | What happens | What you see |
|---|---|---|
| 1. Build Table | Builds a real rainbow table including your password | Dictionary words being hashed, total build time |
| 2. Unsalted Attack | Looks up your password's hash in the table | Hash found in < 1 ms — **CRACKED** |
| 3. Salted Attack | Looks up your salted hash in the table | Hash NOT found — lookup fails, **SAFE** |
| 4. Stretched Attack | Looks up your stretched hash in the table | Hash NOT found, K x cost explained, **SAFE** |
| 5. Peppered Attack | Looks up your peppered hash in the table | Hash NOT found, 2^256 keyspace explained, **SAFE** |

Ends with a four-line verdict comparing all systems.

<img width="536" height="189" alt="image" src="https://github.com/user-attachments/assets/f5af5c2b-5781-4d10-acc9-047e086fa0ab" />
<img width="952" height="677" alt="image" src="https://github.com/user-attachments/assets/e5826022-a779-4621-9e95-5c9fbc606a8e" />
<img width="950" height="676" alt="image" src="https://github.com/user-attachments/assets/297531b8-f745-433c-b368-08a80f6d0dce" />
<img width="950" height="676" alt="image" src="https://github.com/user-attachments/assets/68186060-5bde-4738-82cf-732b30296342" />
<img width="948" height="677" alt="image" src="https://github.com/user-attachments/assets/7e33897a-121d-4961-bc4c-3ab1630f8c17" />
<img width="712" height="621" alt="image" src="https://github.com/user-attachments/assets/d535ecd7-65e0-45a5-8a25-871c4ec3defc" />
<img width="618" height="396" alt="image" src="https://github.com/user-attachments/assets/f18a350b-b78e-467c-a4a5-a6846f053c30" />


---

### 7. Export CSV

**What it does:** Saves experiment results to a CSV file for use in presentations, reports, or further analysis.

**How to use:**
1. Run a full comparison first
2. Click **Export CSV**
3. A file is saved with one row per test round

**CSV columns:**

| Column | Description |
|---|---|
| `test_id` | Round number |
| `num_users` | Users generated that round |
| `dict_size` | Dictionary size used |
| `unsalted_sr_%` | Unsalted success rate |
| `salted_sr_%` | Salted success rate |
| `stretched_sr_%` | Stretched success rate |
| `peppered_sr_%` | Peppered success rate |
| `unsalted_time_s` | Attack time in seconds |

---

### 8. Clear Log

**What it does:** Wipes all output from the scrolling log and resets the live graph. Use this between experiments for a clean slate.

---

## Controls Reference

| Control | Default | What it controls |
|---|---|---|
| Num Tests | 25 | Rounds in a full comparison |
| Dict Size | 2000 | Attacker's wordlist size |
| Reuse % | 0.70 | Fraction of users picking from the top 20% of passwords. Higher = more clustering = worse unsalted result |
| Stretch K | 10 | SHA-256 iterations per password guess. Every +1K costs the attacker K x more compute, costs the defender ~1 ms more per login |

---

## The Four Hashing Systems

This tool compares four progressively stronger password storage strategies using the same database and the same attacker dictionary:

### Unsalted SHA-256 (Vulnerable)

```
stored = sha256(password)
```

The attacker precomputes a rainbow table `{sha256(word): word}` once. Every user hash is a single O(1) lookup. Database size is irrelevant — 100 users or 10 million, same cost.

**Result:** 80-100% of accounts cracked.

### Salted SHA-256 (Defence Level 1)

```
stored = (random_salt, sha256(salt || password))
```

Each user gets a unique 128-bit (16-byte) random salt. The rainbow table becomes useless because the same password produces a different hash for every user. The attacker must now redo `|D|` hashes per user.

**Result:** 0% cracked. Attacker cost multiplied by N.

### Salted + Key-Stretched (Defence Level 2)

```
stored = (salt, sha256^K(salt || password))
```

Same as salted, but SHA-256 is applied K times in a chain. Legitimate login costs ~1 ms extra. The attacker pays K x more per guess per user.

**Result:** 0% cracked. Attacker cost multiplied by N x K.

### Salted + Peppered (Defence Level 3)

```
stored = (salt, sha256(pepper || salt || password))
```

A 256-bit secret (the pepper) is stored only on the application server, never in the database. Even if the database is stolen, the attacker cannot verify any guess without the pepper. 2^256 ~ 10^77 possible keys to exhaust.

**Result:** 0% cracked. Practically infeasible even with infinite compute.

> **Note:** If the server itself is compromised, the pepper is exposed and security falls back to the salted case.

---

## The Math

### Symbols

| Symbol | Meaning |
|---|---|
| `\|D\|` | Dictionary size — number of candidate passwords the attacker has |
| `N` | Number of user accounts in the target database |
| `T_h` | Time to compute one SHA-256 hash (measured as `build_time / \|D\|`) |
| `K` | Stretch iterations — how many times SHA-256 is applied per guess |
| `W` | Attacker work — total compute cost to crack passwords |
| `SR` | Success rate — percentage of accounts recovered |

### Work Formulas

```
Unsalted   W = |D| x T_h
Salted     W = N x |D| x T_h
Stretched  W = N x |D| x K x T_h
Peppered   W = 2^256 x N x |D| x T_h
```

**Unsalted:** The attacker hashes the dictionary once, then every user is cracked at O(1) per lookup. N does not appear — database size is irrelevant to attacker cost.

**Salted:** The rainbow table is worthless. Each user has a unique 128-bit salt, so the attacker must redo `|D|` hashes per user. N becomes a direct multiplier.

**Stretched:** Same as salted, but every single guess costs K hashes instead of one (SHA-256 is applied in a chain). Legitimate login: ~1 ms overhead at K=1000. Attacker: 1000x more work per user.

**Peppered:** A 256-bit secret stored only on the server (never in the DB) is prepended before hashing. Without it, the attacker cannot verify any guess. 2^256 ~ 10^77 keys to exhaust. Practically infeasible even with infinite compute. Note: if the server itself is compromised, the pepper is exposed and security falls back to the salted case.

**Success rate:**

```
SR = (cracked / N) x 100
```

---

## Files

| File | Role |
|---|---|
| `gui.py` | Desktop application — all UI, threading, and orchestration |
| `main.py` | Prints launch instructions |
| `sha256_manual.py` | Pure-Python SHA-256 (no hashlib, intentionally slow for measurable timings) |
| `database_generator.py` | Synthetic user database generator with configurable password reuse |
| `hash_systems.py` | Unsalted and salted hashing functions |
| `preventions.py` | Key-stretching and pepper hashing functions |
| `attackers.py` | Rainbow table builder and attack functions |
| `comparison_runner.py` | Four-way experiment orchestrator |
| `graph_generator.py` | Six matplotlib publication-quality graphs |
| `metrics.py` | Success rate and work estimation helpers |
| `export_results.py` | CSV export |

Full technical reference (every function, constant, and internals): [`x_devdoc.md`](x_devdoc.md)

---

## Tech Stack

| Component | Technology |
|---|---|
| Language | Python 3.10+ |
| GUI Framework | customtkinter + tkinter |
| Graphs | matplotlib + numpy |
| Hashing | Hand-rolled SHA-256 (pure Python, intentionally slow) |
| Randomness | `os.urandom()` for salts and peppers |
| Timing | `time.perf_counter()` for high-resolution measurements |

---

## Production Note

This project uses a hand-rolled SHA-256 for visibility into timing. Do not use plain SHA-256 for passwords in production. Use **Argon2id** (`argon2-cffi`), **bcrypt**, or **scrypt** : all are memory-hard, include built-in salting, and are designed to remain hard as hardware improves.
