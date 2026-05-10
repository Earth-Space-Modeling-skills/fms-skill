---
name: fms
description: >
  Progressive-disclosure skill for the GFDL Flexible Modeling System (FMS),
  the Fortran software framework that underlies NOAA-GFDL atmosphere, ocean,
  and Earth-system models (MOM6, AM4, CM4, ESM4, SHiELD, SPEAR). Covers FMS
  modules (diag_manager, time_manager, mpp, fms_io, coupler), build systems
  (CMake and autotools), the diag_table format, the FMScoupler, and the
  switch to Apache 2.0 license. Foundation for any work inside a GFDL
  model.
version: 0.1.0-scaffold
tags:
  - earth-science
  - climate-model
  - framework
  - fms
  - gfdl
  - noaa
  - fortran
  - mpi
---

# FMS (Flexible Modeling System) Guide

> **FMS** = Flexible Modeling System, the GFDL Fortran framework for atmospheric, oceanic, and Earth-system models.
> Maintainer: NOAA / GFDL
> Source: https://github.com/NOAA-GFDL/FMS
> License: switching to Apache 2.0 with the 2025.04 release (see issue #1683)
> Skill author: Koutian Wu (ktwu01@gmail.com)
> Skill version: 0.1.0-scaffold

**What FMS does:** Provides the common infrastructure that GFDL models are built on, calendar/time management, MPI domain decomposition, parallel I/O, diagnostic output management (`diag_manager` + `diag_table`), grid utilities, exchange grids for component coupling, and a coupler. Used by MOM6, AM4, CM4, ESM4, SHiELD, SPEAR, LM4, and many community forks (CESM/MOM_interface, NorESM/BLOM via MOM-derived chains, etc.).

**Who this skill is for:** Anyone working **inside** a GFDL model who needs to understand FMS-specific patterns: how diagnostic output is configured, how MPI domains are set up, how files are read/written, how to add a new tracer, how to call the coupler.

---

## Quick Decision Tree

```
"What do I need?"
│
├─ 🆕 What is FMS, what models use it, what does it provide?
│  └─ Read: reference/overview.md
│
├─ 🛠️ Build FMS standalone (CMake or autotools)
│  └─ Read: reference/building.md
│
├─ 📊 Configure diagnostic output via diag_table
│  └─ Read: reference/diag-manager.md
│
├─ 🌐 Domain decomposition and parallel I/O (mpp_*, fms_io)
│  └─ Read: reference/mpp-and-io.md
│
├─ 🔗 Coupling components through FMScoupler / exchange grids
│  └─ Read: reference/coupler.md
│
├─ ⏰ Time and calendar management
│  └─ Read: reference/time-and-calendar.md
│
└─ 🐛 Common FMS errors (NetCDF, MPI, diag_table parsing)
   └─ Read: reference/debugging.md
```

---

## Repo Layout (verified from clone)

```
FMS/
├── affinity/         # CPU/thread affinity
├── amip_interp/      # AMIP SST/sea-ice interpolation
├── astronomy/        # Solar geometry
├── axis_utils/       # Coordinate axis helpers
├── coupler/          # Coupler scaffolding (used by FMScoupler)
├── data_override/    # Run-time data substitution
├── diag_manager/     # Diagnostic output engine
├── exchange/         # Exchange grids (atm-ocn, ocn-ice, etc.)
├── field_manager/    # Tracer/field registry
├── fms/              # Top-level driver helpers
├── fms2_io/          # Modern parallel I/O
├── grid_utils/       # Grid manipulation
├── mpp/              # Parallel I/O + message passing
├── time_interp/      # Time interpolation of forcing
├── time_manager/     # Calendars, dates, intervals
├── topography/       # Topography utilities
├── tracer_manager/   # Tracer registry
├── cmake/            # CMake support
├── CMakeLists.txt    # Modern CMake build (preferred)
├── configure.ac      # Autotools build (legacy, still supported)
└── docs/             # Doxygen + diag_table format docs
```

---

## Critical Rules

1. **Two build systems coexist.** CMake (`CMakeLists.txt`) is the modern path; autotools (`configure.ac`) is the legacy path still used by some downstream projects. See `CMAKE_INSTRUCTIONS.md` and `AUTOTOOLS_INSTRUCTIONS.md` for current commands.
2. **The `diag_table` is text-format and order-sensitive.** Top section is global (run name, base date), middle is file definitions, bottom is field assignments to files. See [docs/diag_table.md](https://github.com/NOAA-GFDL/FMS/blob/main/docs/diag_table.md) for the canonical reference.
3. **Use `fms2_io`, not `fms_io`, for new code.** `fms2_io` is the supported parallel I/O API; `fms_io` is being deprecated.
4. **Domains, layouts, and IO layouts are three different concepts**, domain is the physical decomposition, layout is the PE arrangement, IO layout is which subset of PEs writes. Get this wrong and you get cryptic NetCDF errors.
5. **License is changing.** With the 2025.04 release FMS moves to Apache 2.0 (was LGPL 3 before). Track [issue #1683](https://github.com/NOAA-GFDL/FMS/issues/1683) if you redistribute.

---

## Reference Index

| File | Topic |
|---|---|
| reference/overview.md | What FMS is, who uses it |
| reference/building.md | CMake and autotools |
| reference/diag-manager.md | diag_table format, fields, files |
| reference/mpp-and-io.md | Domain decomposition, parallel I/O |
| reference/coupler.md | FMScoupler, exchange grids |
| reference/time-and-calendar.md | time_manager API |
| reference/debugging.md | Common errors |

## Critical agent gotchas (Gemini-reviewed)

- **`FMScoupler` is a separate repo**, not part of FMS itself. It lives at `NOAA-GFDL/FMScoupler` and is the top-level driver for atm-ocn-ice models. The `coupler/` directory in this FMS repo is just type definitions and flux exchange interfaces.
- **`mpp` is a communication layer, not a parallel I/O layer.** Modern parallel I/O is in `fms2_io`. The legacy `mpp_io` is functionally extinct; do not write new code against it.
- **Configuration goes through `input.nml`.** Every FMS module reads its config from a namelist block (`&diag_manager_nml`, `&fms_io_nml`, `&fms_nml`, etc.) in a file conventionally named `input.nml` in the run directory.
- **`field_table` defines tracers.** A run directory needs both `diag_table` (output) and `field_table` (tracers/inputs); `field_manager` reads the latter.
- **Grid files are external NetCDF inputs.** FMS expects `mosaic.nc`, `topog.nc`, `grid_spec.nc` files referenced from the namelist; agent must locate or generate them.
- **`io_layout` controls file count, not just PE subset.** Setting `io_layout = 2,2` produces 4 output files per diag; `1,1` produces one. Mismatch with downstream tools causes silent breakage.
- **mkmf still exists.** GFDL's internal workflows often use `bin/mkmf` (legacy makefile generator) instead of CMake; check what the host model actually uses.

## Status

Scaffold (v0.1.0-scaffold). Source-grounded layout and rules verified against cloned tree, with Gemini critique pass on 2026-05-09. Operational depth being filled in.
