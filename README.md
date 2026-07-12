# Experiment Design

**Chemist's compass — an experiment designer.** A Python toolkit for designing, screening, and scheduling chemistry experiments. It generates Design of Experiments (DOE) trial matrices, then runs each candidate through a four-stage pipeline—Theory → Safety → Trend ranking—so you can see which runs are physically plausible, which pass hazard checks, and which to try first. Includes a desktop GUI for real reaction systems (polymers, small molecules) and a **cookie-oven sandbox** for learning the pipeline without touching real chemistry.

---

## How to read this README

This file is organized for **two audiences**. You do not need to read it top to bottom—use the guide below and skip whatever does not apply to you.

Sections are tagged in headings:

| Tag | Who it is for |
|-----|----------------|
| **Everyone** | Safety rules and shared concepts—all readers should see these |
| **Everyday users** | Students, chemists, and newcomers using the GUI or learning the workflow |
| **Programmers** | Developers installing from source, calling the API, or extending the code |
| **Reference** | Lookup material either audience may need occasionally |

### Suggested path — everyday users

1. [Safety notice](#safety-notice--read-this-first-everyone) — **Everyone** (required)
2. [What this program does](#what-this-program-does-everyday-users) — **Everyday users**
3. [Cookie-oven sandbox](#cookie-oven-sandbox-everyday-users) — story and state diagram only; skip the programmer subsection
4. [The four-layer pipeline](#the-four-layer-pipeline-everyone) — **Everyone**
5. [Using the GUI](#using-the-gui-everyday-users) — **Everyday users**

**Safe to skip:** Quick start (developers), Command-line examples, Project layout, Design principles, Development

**Optional:** Run `python examples/demo.py` if you want to see the cookie-oven tutorial in the terminal (no chemistry knowledge required).

### Suggested path — programmers

1. [Safety notice](#safety-notice--read-this-first-everyone) — **Everyone**, especially the programmer bullets
2. [Quick start (developers)](#quick-start-developers) — **Programmers**
3. [Cookie-oven sandbox](#cookie-oven-sandbox-everyday-users) — read **For programmers**; skim or skip the baking story
4. [The four-layer pipeline](#the-four-layer-pipeline-everyone) — **Everyone** (layer contracts)
5. [Command-line examples](#command-line-examples-programmers) — **Programmers**
6. [Project layout](#project-layout-reference) — **Reference**
7. [Design principles](#design-principles-reference) and [Development](#development-programmers) — **Programmers**

**Safe to skip:** [Using the GUI](#using-the-gui-everyday-users) if you only care about the library API and tests (the GUI is a thin client over the same layers).

---

## Safety notice — read this first (Everyone)

> **This software is for planning and education only. It is not a substitute for laboratory safety training, institutional review, or professional chemical judgment.**

- **Do not try any reactions described by this program at home.** The examples (polymerizations, initiators, elevated temperatures, and similar) are meant for trained personnel in properly equipped labs—not kitchens, garages, or personal workspaces.
- **An “Allow” or “Caution” gate does not mean a trial is safe to run.** Outputs are heuristic annotations. A blocked trial is a strong signal to stop; an allowed trial still requires your own risk assessment.
- **Missing or wrong hazard input can silently under-state risk.** Always verify GHS H-codes and P-codes from official safety data sheets (SDS).

**Programmers — read these too:**

- Treat Safety and Theory layers as **advisory logic**, not authorization.
- Do not wire this tool into automated lab equipment, unsupervised execution, or any system that could start a physical experiment without human review.
- Hazard data entered via the API or GUI is only as good as your sources.

If you are new to lab work, use the **[cookie-oven sandbox](#cookie-oven-sandbox-everyday-users)** to learn how the pipeline behaves before entering real chemical names in the GUI.

---

## What this program does (Everyday users)

If you are planning lab work and want help structuring trials instead of guessing one condition at a time, this program:

1. **Builds a trial matrix (DOE)** from the factors you care about—temperature, reaction time, monomer ratios, and so on.
2. **Checks each trial against simple theory rules** (causality, thermodynamics, mechanism context) and flags ones that do not make physical sense.
3. **Annotates safety** using GHS hazard information you provide for reactants, plus process heuristics (extended heating, initiator handling, and similar).
4. **Ranks the survivors** into a suggested run order when you have more cleared trials than you can run immediately.

Nothing here replaces a qualified chemist or your institution’s safety review. It is a structured worksheet that keeps process context, hazard awareness, and scheduling logic in one place.

---

## Cookie-oven sandbox (Everyday users)

The cookie oven is a **deliberately fake chemistry system** so you can learn the four-layer pipeline without hazardous materials. Think of it as a tutorial level before working with real reactions in the GUI.

*Programmers: technical details are in the subsection at the end of this section.*

### The story

You are baking cookies. Each experiment proposes a **state change**—for example, turning **Dough** into a **Cookie**. You adjust three oven settings:

| Setting | Example levels |
|---------|----------------|
| **Oven temperature** | low / medium / high |
| **Time** | short / medium / long |
| **Oven mode** | ON / OFF / COLD |

The program generates many combinations (a DOE matrix), then asks:

- **Theory:** Can dough realistically become a cookie this way? (You cannot un-bake a cookie. A cold oven cannot bake without heat.)
- **Safety:** Is this bake window reasonable, or are we likely to burn the batch?
- **Trend:** If several bakes are viable, which should we try first based on past results?

No food, no chemicals, no lab coat required—just a mental model for how the software thinks.

### The states and rules

Cookies move through a small **state machine** with only forward paths:

```
Dough → Baking Dough → Cookie → Burnt Cookie
                              ↘ Cooled Cookie
```

Trying to go backward (e.g. Cookie → Dough) fails the theory filter, the same way many real reactions are irreversible. The theory layer also checks that you actually turned the heat on when baking is required.

**Try it (optional, safe on any computer):**

```bash
python examples/demo.py
```

### For programmers

- **Module:** `experiment_design/cookie_oven.py`
- **CLI demo:** `python examples/demo.py`
- **Key types:** `CookieState`, `FORWARD_EDGES`, `build_oven_doe_space()`, `oven_trend_kernel()`
- **Sandbox detection:** `is_sandbox_transition()` — when both `initial_state` and `target_state` are cookie-oven labels, theory uses the baking graph and baking-specific safety heuristics. Reverse oven transitions (e.g. Cookie → Dough) are **blocked in the Theory layer**. Real chemistry transitions skip that graph and use mechanism-aware rules instead; some invalid DOE inputs for real reactions are caught in the GUI at **Generate DOE** (see [Using the GUI](#using-the-gui-everyday-users)).
- **Tests:** `tests/test_oven_sandbox.py` exercises the full pipeline on oven scenarios (valid bake, cold-oven adversarial, reverse transition, overbake, and more).

The sandbox shares the **same pipeline API** as real chemistry (`run_oven_pipeline`, `SafetyFilter`, `TheoryScoringEngine`, `TrendKernel`), which makes it useful for integration tests and for understanding layer contracts before extending the GUI.

---

## Quick start (Programmers)

**Requirements:** Python 3.10+

```bash
git clone <your-repo-url>
cd experiment_design
pip install -e ".[dev,gui]"
pytest
python examples/demo.py          # safe sandbox — good first run
python examples/gui_prototype.py   # real chemistry GUI — lab planning only
```

| Command | Purpose |
|---------|---------|
| `python examples/demo.py` | Cookie-oven sandbox pipeline (CLI) — **recommended first run** |
| `python examples/gui_prototype.py` | Desktop app for real reaction systems (CustomTkinter) |
| `python examples/chemistry_hazard_demo.py` | GHS hazard input demo (CLI) |
| `pytest` | Run the test suite |

Install without the GUI: `pip install -e ".[dev]"`

---

## The four-layer pipeline (Everyone)

Trials move through the same stages in the GUI, the cookie-oven demo, and the chemistry examples:

```
DOE  →  Theory filter  →  Safety filter  →  Trend ranking
```

| Layer | What it answers |
|-------|-----------------|
| **DOE** | What factor combinations should we try? |
| **Theory** | Is this transition physically plausible given mechanism, temperature, time, and optional structure data? |
| **Safety** | What is the hazard picture (GHS signal word, gate, notes) for this trial? |
| **Trend** | Among cleared trials, which should we run first? |

Each layer has an **input table** and an **output table**. Downstream layers only see trials that passed upstream filters (except Safety, which annotates all theory-passed trials; blocked trials are labeled but not forwarded to Trend).

---

## Using the GUI (Everyday users)

Launch:

```bash
python examples/gui_prototype.py
```

The window title is **Chemist's compass -- an experiment designer**. A cookie icon appears in the title bar when `cookie.ico` is found (project root, `examples/`, or next to a packed `.exe`).

**Reminder:** Only use this for reactions you are qualified to plan in a proper laboratory. Do not treat the GUI as permission to run experiments at home.

### Tab 1 — Experimental design (DOE)

1. Enter **reactants** and a **product** to define the chemical transition.
2. Set fixed process details: mechanism, initiator, solvent, default temperature, and related fields.
3. Add **DOE factors** (e.g. `Reaction time`, `Temperature`) and their levels, then submit each factor.
4. Click **Generate DOE** to fill the output trial matrix.

   **Generate DOE checks** (messagebox warnings; generation is blocked until fixed):

   | Problem | Example |
   |---------|---------|
   | No transition — reactants and product overlap | Reactants `HPMAA, CHMI`, product `HPMAA` |
   | Reverse polymerization — polymer as reactant, monomer as product | Mechanism: chain-growth; reactants `Polystyrene`, product `Styrene` |

   Matching is case-insensitive. These checks run only when you click **Generate DOE**, not while you type.

5. Click **Send to Theory** (or use **Run full workflow**) to continue.

Supported modes include a compact 9-trial basic design and a fuller 27-trial design.

### Tab 2 — Theory filter

- Trials arrive automatically from DOE with process context attached.
- Optionally enable **structure-guided ΔG** and attach reactant/product structure files for advanced checks.
- Click **Run theory filter** to label each trial `ALLOW` or `BLOCK`.

### Tab 3 — Safety filter

1. Submit **species hazard data** for reactants, solvents, and other materials (hazard classes, H-codes, P-codes). A link to Sigma-Aldrich product search is provided as a reminder when codes are missing.
2. Click **Run safety filter**. Outputs show the **gate** (Allow / Caution / Block), **GHS signal word**, and explanatory notes.
3. Cleared trials are forwarded to Trend automatically.

### Tab 4 — Trend ranking

- Click **Run trend ranking** to order safety-cleared trials by priority (`NOW`, `SOON`, `LATER`, and so on).
- The kernel uses your DOE factor names and any response history you add later; it does not reject trials—that happened upstream.

**Tip:** Use **Run full workflow** on the DOE tab to execute all four layers in sequence.

---

## Command-line examples (Programmers)

### Programmatic DOE only

```python
from experiment_design import (
    ExperimentSpace,
    generate_full_factorial,
    generate_orthogonal_subset,
    print_experiment_plan,
    set_seed,
)

set_seed(42)

space = ExperimentSpace({
    "Temperature": (60, 90),
    "Reaction time": ["6 hrs", "12 hrs", "24 hrs"],
})

full = generate_full_factorial(space)
subset = generate_orthogonal_subset(space, strength=2, seed=42)

print_experiment_plan(subset)
```

### Declaring hazards in code

```python
from experiment_design.hazard_input import ReactionSystemBuilder
from experiment_design.safety import SafetyFilter

experiment = {
    "reaction_system": (
        ReactionSystemBuilder()
        .add("reactants", "HPMAA", hazard_classes=["flammable"], h_codes=["H225"])
        .build()
    ),
    "mechanism": "10. Chain-growth polymerization",
    "Temperature": "75 °C",
    "Reaction time": "6 hrs",
    "initial_state": "HPMAA, CHMI, HEMA",
    "target_state": "p(HPMAA-CHMI-co-HEMA)",
}

result = SafetyFilter().evaluate(experiment)
print(result["signal_word"], result["gate"])
```

See `examples/chemistry_hazard_demo.py` for a longer walkthrough.

---

## Project layout (Reference)

```
experiment_design/
├── experiment_design/          # Core library
│   ├── space.py                # Experiment factor definitions
│   ├── orthogonal.py           # Full-factorial & orthogonal subset generation
│   ├── theory_scoring.py       # Theory filter engine
│   ├── thermo_constraints.py   # Thermodynamic constraint checks (cookie graph)
│   ├── material_direction.py   # Polymer↔monomer direction heuristics (GUI)
│   ├── safety.py               # GHS-aware safety annotation
│   ├── process_safety.py       # Thermal / duration process heuristics
│   ├── hazard_input.py         # User hazard declarations (CLI & GUI)
│   ├── ghs.py                  # GHS H-code catalog & signal words
│   ├── trend.py                # Trial priority ranking
│   ├── mechanisms.py           # Reaction mechanism families
│   ├── initiators.py           # Polymerization initiator rules
│   ├── experiment_context.py   # Merging DOE factors into layer context
│   └── cookie_oven.py          # Sandbox demo domain
├── examples/
├── examples/
│   ├── gui_prototype.py        # Main desktop application ("Chemist's compass")
│   ├── demo.py                 # Cookie-oven CLI demo
│   └── chemistry_hazard_demo.py
├── cookie.ico                  # In-app window title-bar icon (cookie.png fallback)
├── icon.ico                    # Packed .exe file icon (PyInstaller --icon=icon.ico)
├── tests/                      # Pytest suite
└── pyproject.toml
```

**Icons (two roles):**

| File | Purpose |
|------|---------|
| `cookie.ico` / `cookie.png` | **Runtime** — title-bar icon while the GUI is open. Searched in project root, `examples/`, and next to a packed `.exe` (or PyInstaller `_internal/`). |
| `icon.ico` | **Build time** — Windows executable icon (Explorer, taskbar shortcut). Pass to your packer, e.g. PyInstaller `--icon=icon.ico`. Not read by the app at runtime. |

---

## Design principles (Reference)

- **Deterministic and reproducible** — seeded DOE generation; stable trend tie-breaking.
- **Modular layers** — each stage is usable on its own via the Python API.
- **Separation of concerns** — theory decides plausibility; safety annotates hazards; trend schedules; none of them silently overrides the others.
- **Real chemistry and a teaching sandbox** — the GUI targets real reaction systems; the cookie oven remains available for demos and tests.
- **Safety-first framing** — gates and signal words inform humans; they do not authorize unattended or home use.
- **Standard library first** — core logic has no ML dependency; the GUI optionally uses CustomTkinter.

---

## Development (Programmers)

```bash
pip install -e ".[dev,gui]"
pytest          # 150+ tests
```

When changing layer behavior, add or update tests under `tests/`. Key integration tests cover the cookie-oven sandbox, real polymer trials, GHS hazard input, process thermal rules, trend ranking stability, GUI DOE transition warnings (`test_no_transition_warning.py`, `test_material_direction.py`), and icon path resolution (`test_gui_icon_paths.py`).

---

## License (Reference)

MIT — see `pyproject.toml`.
