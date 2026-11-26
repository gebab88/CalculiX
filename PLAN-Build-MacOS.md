<a id="deutsch"></a>

# CalculiX auf macOS (Apple Silicon) — Build- und Install-Plan

**Deutsch** · [English](#english)

> **Status:** Schritte 1–3 ausgeführt (2026-11-26): Build erfolgreich, Smoke-Test `achtel2` identisch mit Referenz. Dafür waren zwei Anpassungen an `CMakeLists.txt` nötig (siehe unten).
> **Date:** 2026-11-26
> **Zielsystem:** macOS 27.2, Darwin 27.2.0, arm64 (Apple Silicon)
> **Projekt:** `<repo>` = Wurzel dieses Git-Repos (Basis: upstream `57a4e71`)

## Zweck

CalculiX unter macOS kompilieren und als erzeugte Binary verfügbar machen. Es existiert **keine `install(`-Regel** in `CMakeLists.txt` — "installieren" endet daher bei der fertigen Binary `build/gnu-lp64/CalculiX` (plus `libccx.a`).

## Befunde (Stand 2026-11-26)

| Komponente | Status |
|---|---|
| cmake | 4.4.4 via `~/Library/Python/3.13/bin` (Mindestversion 3.28 erfüllt) |
| Apple `gcc` | `/usr/bin/gcc` ist Apple clang — **kein gfortran** vorhanden |
| MacPorts | `/opt/local` — gcc15 (`gcc-mp-15`, `gfortran-mp-15`), OpenBLAS 0.3.34 (+lapack, LP64), hwloc, metis, libgomp |
| ARPACK | 3.9.1 via MacPorts — `libarpack.dylib` in `/opt/local/lib` |
| PaStiX | 6.4.0 via MacPorts (`pastix @6.4.0_0+openblas`) — `PASTIXConfig.cmake` in `/opt/local/lib/cmake/pastix`, spm/scotch/metis/hwloc, **32-bit-Integer (LP64)** (`SPM_INT64` ist `undef` in `spm/config.h`) |
| SPOOLES | **nicht** in MacPorts — nicht verfügbar |
| OpenBLAS | 0.3.34 (+lapack eingebaut, **LP64**) — `/opt/local/lib/libopenblas.*` |

Wichtige Konsequenzen:
- Der `gnu-lp64`-Preset aus `CMakePresets.json` hardcodiert `gcc`/`gfortran` — das existiert hier nicht. Konfiguration also direkt mit `-D CMAKE_C_COMPILER=/opt/local/bin/gcc-mp-15 -D CMAKE_Fortran_COMPILER=/opt/local/bin/gfortran-mp-15`.
- PaStiX aus MacPorts ist ein **32-bit-Integer (LP64)**-Build → passt zum `gnu-lp64`-Pfad (32-bit Ints). Ein `gnu-ilp64`-Build (INTSIZE64) würde gegen die 32-bit libpastix und das LP64-OpenBLAS ABI-Mismatch erzeugen → **nicht** verwenden.
- PaStiX+LP64 ist in `INSTALL.md` als „untested combination" markiert (CMake-Warnung, nicht fatal), ist aber die konsistente Variante für diese Pakete.

## Globaler Ansatz

- **Solver:** PaStiX 6.4.0 (MacPorts) — Default-Solver SPOOLES ist nicht verfügbar. `SPOOLES=OFF`, `PASTIX=ON`.
- **Build-Type:** Release, LP64 (32-bit Ints)
- **Compilers:** MacPorts `gcc-mp-15`/`gfortran-mp-15`
- **FindARPACK** (MODULE, nicht CONFIG) findet `/opt/local/lib/libarpack.dylib` via `find_library`.
- **Kompatibilitäts-Schicht v6.4.0:** Ist bereits im `CMakeLists.txt` eingebunden — bei `PASTIX=ON` wird automatisch `src/pastix.c` durch `cmake/compat/pastix/v6.4.0/pastix.c` ersetzt. Kein manuelles Eingreifen in `src/`.

## Notwendige Anpassungen an `CMakeLists.txt` (Commit `c6cf771`, Branch `macos-build`)

Ohne diese beiden Änderungen scheitert der Build auf macOS.

**1. `ARCH`-Mapping Darwin → Linux**

```cmake
set(CCX_ARCH_CODE "${CMAKE_SYSTEM_NAME}")
if(CCX_ARCH_CODE STREQUAL "Darwin")
  set(CCX_ARCH_CODE "Linux")
endif()
target_compile_definitions(ccx PUBLIC
  ARCH=${CCX_ARCH_CODE}
  MATRIXSTORAGE
)
```

- Grund: `src/CalculiX.h` kennt für `ARCH` nur `Linux`, `IRIX`, `IRIX64`, `HP`. Mit `-DARCH=Darwin` greift keiner der `#if`-Zweige, die Makros `FORTRAN()`/`CEE()` bleiben undefiniert und der C-Teil kompiliert nicht.
- macOS/gfortran benutzt dieselbe Symbol-Konvention wie Linux (angehängter Unterstrich) → `Linux` ist korrekt.

**2. Sprachen auf C und Fortran beschränken**

```cmake
project(CalculiX LANGUAGES C Fortran)   # vorher: project(CalculiX) + enable_language(C Fortran)
```

- Grund: `project()` ohne `LANGUAGES` aktiviert implizit auch CXX. CMake findet dafür Apple clang (`/usr/bin/c++`), und `find_package(OpenMP REQUIRED)` verlangt OpenMP für alle aktiven Sprachen:
  `Could NOT find OpenMP_CXX (missing: OpenMP_CXX_FLAGS OpenMP_CXX_LIB_NAMES)`
- Das Projekt enthält keine C++-Quellen; CXX wird nicht gebraucht.

## Schritt 1: Konfigurieren

```bash
cd <repo>
cmake -B build/gnu-lp64 \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_C_COMPILER=/opt/local/bin/gcc-mp-15 \
  -DCMAKE_Fortran_COMPILER=/opt/local/bin/gfortran-mp-15 \
  -DCMAKE_PREFIX_PATH=/opt/local \
  -DSPOOLES=OFF \
  -DPASTIX=ON
```

- Erwartung: `finding PaStiX` über `/opt/local/lib/cmake/pastix/PASTIXConfig.cmake`; `ARPACK` über `cmake/FindARPACK.cmake`; Warnung „PaStiX with CALCULIX_INT64=OFF is an untested combination." ist **erwartet** (nicht fatal).
- Ergebnis: OpenMP 4.5 (C + Fortran), BLAS/LAPACK `/opt/local/lib/libopenblas.dylib`, ARPACK `/opt/local/lib/libarpack.dylib` (Find-Modul), `PASTIX_DIR=/opt/local/lib/cmake/pastix`. Compile-Defines: `-DARCH=Linux -DARPACK -DPASTIX -DMATRIXSTORAGE`, kein `INTSIZE64`.
- Zur Warnung: Integer-Breiten sind konsistent — `ITG` = `int` (LP64), `pastix_int_t`/`spm_int_t` = `int` (`PASTIX_INT64`/`SPM_INT64` undefiniert in MacPorts-Headern). Die Binding übergibt die Arrays per Pointer-Cast (`spm->colptr = (spm_int_t*) icolpastix`), daher ist Gleichheit zwingend. Die Warnung bedeutet „ungetestet", nicht „bekannt fehlerhaft"; ein ILP64-Build gegen diese PaStiX wäre dagegen falsch (und warnt nicht).
- **Achtung:** `cmake --build` konfiguriert nicht. Nach jedem Löschen des Build-Verzeichnisses (`rm -rf build/gnu-lp64/*`) muss dieser Schritt erneut laufen, sonst: `Error: not a CMake build directory (missing CMakeCache.txt)`. Für einen sauberen Neuanfang ohne Löschen: `cmake --fresh -B build/gnu-lp64 …`.

## Schritt 2: Bauen

```bash
cmake --build build/gnu-lp64 -j
```

Ergebnis: `build/gnu-lp64/CalculiX` (+ `libccx.a`)

Ist-Stand (2026-11-26):
- Build ohne Fehler; Binary `Mach-O 64-bit executable arm64`, 6,2 MB.
- 119 Compiler-Warnungen, ausschließlich Legacy-Code (Fortran deleted features wie DO-Termination/Arithmetic IF/ASSIGN, gfortran-Bounds-Hinweise in Schleifen, einige `const`-Qualifier-Warnungen in C) sowie `ld: warning: ignoring duplicate libraries: 'libccx.a'` (gewollt, `ccx` steht absichtlich zweimal auf der Link-Zeile).
- PaStiX/spm sind **statisch** gelinkt (MacPorts liefert nur `libpastix.a`/`libspm.a`); dynamisch: libgomp, ARPACK, OpenBLAS, scotch, metis, hwloc, libgfortran, libquadmath, libz — alle aus `/opt/local`.

## Schritt 3: Verification (Smoke-Test)

1. In `test/` ein kleines Beispiel (z. B. `achtel2.inp`) ausführen:
   ```bash
   cd <repo>/test
   <repo>/build/gnu-lp64/CalculiX -i achtel2
   ```
2. Prüfen, dass eine `.dat`-Datei entsteht und keine numerischen Abbrüche auftreten; gegebenenfalls Ergebnis mit der beigelegten Referenz `achtel2.dat.ref` vergleichen.
3. Optional: `test/compare` (seriell) oder `test/compare_par` ausführen — braucht ggf. `cgx` (externe Kommando) und ggf. manches mehr; als optionaler Volltest zu behandeln.
4. Falls Smoke-Test abbricht: Fehlermeldung protokollieren; Konsolenausgabe sowie die `.sta`-/`.cvg`-Datei des Jobs analysieren.

Ist-Stand (2026-11-26), `achtel2` mit `OMP_NUM_THREADS=1`, in einer Kopie außerhalb von `test/` ausgeführt:
- „Job finished", Exit-Code 0, Laufzeit ~1 s.
- Solver ist tatsächlich PaStiX (Banner, Refinement-Residuen 1,0e-08 / 8,1e-13).
- `achtel2.dat` **byte-identisch** mit `achtel2.dat.ref`; `datcheck.pl` ohne Befund, kein NaN.
- Hinweis: Die Ausgabe „CPU: AMD Opteron 6180 - Intel MKL" ist ein in `libpastix.a` fest eingebauter String, keine Erkennung der Hardware.
- Offen: breiterer Testlauf über mehrere Analysearten (Frequenz, Kontakt, nichtlinear). `test/compare` ruft fest `~/CalculiX/src/CalculiX` auf und muss dafür angepasst oder umgangen werden.

## Schritt 4: Installation

- Es gibt **keine** `install(`-Regeln in `CMakeLists.txt` — kein `cmake --install`.
- Ergebnis ist `build/gnu-lp64/CalculiX` (+ `libccx.a`). Falls ein globaler Zugriff gewünscht ist: Binary nach einem gewählten Ziel kopieren/verlinken (z. B. `/opt/local/bin/CalculiX`) — dies erst nach Freigabe durch den Nutzer. **Vorher keine Systemeinträge** vornehmen.

## Risiken & Fallbacks

| Risiko | Fallback |
|---|---|
| `FindARPACK` findet `libarpack.dylib` nicht | `-DARPACK_LIBRARY_DIR=/opt/local/lib` hinzufügen |
| `OpenMP` (libgomp) wird nicht erkannt | **Eingetreten** (für CXX/Apple clang) — behoben durch `project(CalculiX LANGUAGES C Fortran)`, siehe Anpassung 2. Für C/Fortran wird `-fopenmp` von gcc-mp-15 problemlos gefunden. |
| `FindLAPACK`/`find_package(BLAS/LAPACK)` findet MacPorts-OpenBLAS (`+lapack` eingebaut) nicht sicher | `-DCMAKE_PREFIX_PATH=/opt/local` (bereits in Schritt 1); ggf. `-DLAPACK_ROOT=/opt/local` |
| PaStiX-Link scheitert (z. B. wegen spm-/scotch-ABI) | **Fallback:** SPOOLES 2.2 von netlib als Default-Solver (`wget https://www.netlib.org/linalg/spooles/spooles.2.2.tgz`) mit SPOOLES=ON — SPOOLES erfordert LP64 (32-bit), passt daher zur selben LP64-Konfiguration |
| PaStiX+LP64 verhält sich nicht wie erwartet zur Laufzeit | Referenzvergleiche in `test/` ernst nehmen; ggf. SPOOLES-Fallback |
| CMake 4.4.4 vs. custom Find-Module | `FindARPACK.cmake` / `FindSPOOLES.cmake` sind einfach strukturiert; das SPOOLES-Modul wird ohne SPOOLES=ON gar nicht geladen |

## Nicht enthalten (bewusste Entscheidungen)

- Kein `gnu-ilp64`-Build: Die MacPorts-Abhängigkeiten sind nur LP64 (32-bit), nicht ILP64-fähig.
- Kein `make`-basierter Build (`src/Makefile`): Intel/Linux-zertifiziert, veraltet, SPOOLES/LAPACK-/ARPACK-Pfade im Makefile auf Linux festgelegt.
- Kein `cmake --install`: keine install-Regeln vorhanden.

## Offene Punkte für die Zukunft

- Falls der Nutzer einen global kompilierten CalculiX mit PaStiX-ILP64 will, wäre das ein separates Projekt mit frischen ILP64-Abhängigkeiten (BLAS/LAPACK/ARPACK/Solver) — außerhalb dieses Plans.
- Paketierung/`install`-Regeln im `CMakeLists.txt` ergänzen — nicht Teil dieses Plans.

---

<a id="english"></a>

# CalculiX on macOS (Apple Silicon) — Build and Install Plan

[Deutsch](#deutsch) · **English**

> **Status:** Steps 1–3 completed (2026-11-26): build successful, smoke test `achtel2` identical to the reference. Two changes to `CMakeLists.txt` were required (see below).
> **Date:** 2026-11-26
> **Target system:** macOS 27.2, Darwin 27.2.0, arm64 (Apple Silicon)
> **Project:** `<repo>` = root of this Git repository (base: upstream `57a4e71`)

## Purpose

Compile CalculiX on macOS and make the resulting binary available. There is **no `install(` rule** in `CMakeLists.txt`, so "installing" ends with the finished binary `build/gnu-lp64/CalculiX` (plus `libccx.a`).

## Findings (as of 2026-11-26)

| Component | Status |
|---|---|
| cmake | 4.4.4 via `~/Library/Python/3.13/bin` (minimum version 3.28 satisfied) |
| Apple `gcc` | `/usr/bin/gcc` is Apple clang — **no gfortran** available |
| MacPorts | `/opt/local` — gcc15 (`gcc-mp-15`, `gfortran-mp-15`), OpenBLAS 0.3.34 (+lapack, LP64), hwloc, metis, libgomp |
| ARPACK | 3.9.1 via MacPorts — `libarpack.dylib` in `/opt/local/lib` |
| PaStiX | 6.4.0 via MacPorts (`pastix @6.4.0_0+openblas`) — `PASTIXConfig.cmake` in `/opt/local/lib/cmake/pastix`, spm/scotch/metis/hwloc, **32-bit integers (LP64)** (`SPM_INT64` is `undef` in `spm/config.h`) |
| SPOOLES | **not** in MacPorts — not available |
| OpenBLAS | 0.3.34 (+lapack built in, **LP64**) — `/opt/local/lib/libopenblas.*` |

Key consequences:
- The `gnu-lp64` preset in `CMakePresets.json` hardcodes `gcc`/`gfortran`, which do not exist here. Configure directly with `-D CMAKE_C_COMPILER=/opt/local/bin/gcc-mp-15 -D CMAKE_Fortran_COMPILER=/opt/local/bin/gfortran-mp-15` instead.
- PaStiX from MacPorts is a **32-bit integer (LP64)** build → matches the `gnu-lp64` path (32-bit ints). A `gnu-ilp64` build (INTSIZE64) would cause an ABI mismatch with the 32-bit libpastix and the LP64 OpenBLAS → do **not** use it.
- PaStiX+LP64 is flagged as an "untested combination" (CMake warning, not fatal; according to `INSTALL.md`, PaStiX is only tested with 64-bit integers), but it is the consistent choice for these packages.

## Overall approach

- **Solver:** PaStiX 6.4.0 (MacPorts) — the default solver SPOOLES is not available. `SPOOLES=OFF`, `PASTIX=ON`.
- **Build type:** Release, LP64 (32-bit ints)
- **Compilers:** MacPorts `gcc-mp-15`/`gfortran-mp-15`
- **FindARPACK** (MODULE, not CONFIG) finds `/opt/local/lib/libarpack.dylib` via `find_library`.
- **Compatibility layer v6.4.0:** Already wired into `CMakeLists.txt` — with `PASTIX=ON`, `src/pastix.c` is automatically replaced by `cmake/compat/pastix/v6.4.0/pastix.c`. No manual changes in `src/`.

## Required changes to `CMakeLists.txt` (commit `c6cf771`, branch `macos-build`)

Without these two changes the build fails on macOS.

**1. `ARCH` mapping Darwin → Linux**

```cmake
set(CCX_ARCH_CODE "${CMAKE_SYSTEM_NAME}")
if(CCX_ARCH_CODE STREQUAL "Darwin")
  set(CCX_ARCH_CODE "Linux")
endif()
target_compile_definitions(ccx PUBLIC
  ARCH=${CCX_ARCH_CODE}
  MATRIXSTORAGE
)
```

- Reason: `src/CalculiX.h` only knows `Linux`, `IRIX`, `IRIX64` and `HP` for `ARCH`. With `-DARCH=Darwin` none of the `#if` branches match, the `FORTRAN()`/`CEE()` macros stay undefined and the C part does not compile.
- gfortran on macOS uses the same symbol convention as on Linux (trailing underscore) → `Linux` is correct.

**2. Restrict languages to C and Fortran**

```cmake
project(CalculiX LANGUAGES C Fortran)   # previously: project(CalculiX) + enable_language(C Fortran)
```

- Reason: `project()` without `LANGUAGES` implicitly enables CXX as well. CMake picks Apple clang (`/usr/bin/c++`) for it, and `find_package(OpenMP REQUIRED)` requires OpenMP for all enabled languages:
  `Could NOT find OpenMP_CXX (missing: OpenMP_CXX_FLAGS OpenMP_CXX_LIB_NAMES)`
- The project has no C++ sources; CXX is not needed.

## Step 1: Configure

```bash
cd <repo>
cmake -B build/gnu-lp64 \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_C_COMPILER=/opt/local/bin/gcc-mp-15 \
  -DCMAKE_Fortran_COMPILER=/opt/local/bin/gfortran-mp-15 \
  -DCMAKE_PREFIX_PATH=/opt/local \
  -DSPOOLES=OFF \
  -DPASTIX=ON
```

- Expected: PaStiX is found via `/opt/local/lib/cmake/pastix/PASTIXConfig.cmake`, `ARPACK` via `cmake/FindARPACK.cmake`; the warning "PaStiX with CALCULIX_INT64=OFF is an untested combination." is **expected** (not fatal).
- Result: OpenMP 4.5 (C + Fortran), BLAS/LAPACK `/opt/local/lib/libopenblas.dylib`, ARPACK `/opt/local/lib/libarpack.dylib` (find module), `PASTIX_DIR=/opt/local/lib/cmake/pastix`. Compile definitions: `-DARCH=Linux -DARPACK -DPASTIX -DMATRIXSTORAGE`, no `INTSIZE64`.
- About the warning: the integer widths are consistent — `ITG` = `int` (LP64), `pastix_int_t`/`spm_int_t` = `int` (`PASTIX_INT64`/`SPM_INT64` are undefined in the MacPorts headers). The binding hands the arrays over via a pointer cast (`spm->colptr = (spm_int_t*) icolpastix`), so the widths must match. The warning means "untested", not "known to be broken"; an ILP64 build against this PaStiX, on the other hand, would be wrong (and does not warn).
- **Note:** `cmake --build` does not configure. After every deletion of the build directory (`rm -rf build/gnu-lp64/*`) this step has to be run again, otherwise: `Error: not a CMake build directory (missing CMakeCache.txt)`. For a clean start without deleting: `cmake --fresh -B build/gnu-lp64 …`.

## Step 2: Build

```bash
cmake --build build/gnu-lp64 -j
```

Result: `build/gnu-lp64/CalculiX` (+ `libccx.a`)

Status (2026-11-26):
- Build without errors; binary `Mach-O 64-bit executable arm64`, 6.2 MB.
- 119 compiler warnings, all from legacy code (Fortran deleted features such as DO termination/arithmetic IF/ASSIGN, gfortran bounds hints in loops, a few `const` qualifier warnings in C), plus `ld: warning: ignoring duplicate libraries: 'libccx.a'` (intended: `ccx` appears twice on the link line on purpose).
- PaStiX/spm are linked **statically** (MacPorts only ships `libpastix.a`/`libspm.a`); linked dynamically: libgomp, ARPACK, OpenBLAS, scotch, metis, hwloc, libgfortran, libquadmath, libz — all from `/opt/local`.

## Step 3: Verification (smoke test)

1. Run a small example in `test/` (e.g. `achtel2.inp`):
   ```bash
   cd <repo>/test
   <repo>/build/gnu-lp64/CalculiX -i achtel2
   ```
2. Check that a `.dat` file is produced and that there are no numerical aborts; if applicable, compare the result with the bundled reference `achtel2.dat.ref`.
3. Optional: run `test/compare` (serial) or `test/compare_par` — may need `cgx` (external command) and possibly more; treat it as an optional full test.
4. If the smoke test fails: record the error message; inspect the console output and the job's `.sta`/`.cvg` files.

Status (2026-11-26), `achtel2` with `OMP_NUM_THREADS=1`, run in a copy outside `test/`:
- "Job finished", exit code 0, runtime ~1 s.
- The solver is indeed PaStiX (banner, refinement residuals 1.0e-08 / 8.1e-13).
- `achtel2.dat` is **byte-identical** to `achtel2.dat.ref`; `datcheck.pl` reports nothing, no NaN.
- Note: the output "CPU: AMD Opteron 6180 - Intel MKL" is a string hardcoded in `libpastix.a`, not a detection of the actual hardware.
- Open: a broader test run across several analysis types (frequency, contact, nonlinear). `test/compare` hardcodes `~/CalculiX/src/CalculiX` and has to be adapted or worked around for this.

## Step 4: Installation

- There are **no** `install(` rules in `CMakeLists.txt` — no `cmake --install`.
- The result is `build/gnu-lp64/CalculiX` (+ `libccx.a`). If global access is wanted: copy/link the binary to a chosen location (e.g. `/opt/local/bin/CalculiX`) — only after the user approves. **No system-wide changes** before that.

## Risks & fallbacks

| Risk | Fallback |
|---|---|
| `FindARPACK` does not find `libarpack.dylib` | add `-DARPACK_LIBRARY_DIR=/opt/local/lib` |
| `OpenMP` (libgomp) is not detected | **Occurred** (for CXX/Apple clang) — fixed by `project(CalculiX LANGUAGES C Fortran)`, see change 2. For C/Fortran, gcc-mp-15's `-fopenmp` is found without problems. |
| `FindLAPACK`/`find_package(BLAS/LAPACK)` does not reliably find MacPorts OpenBLAS (`+lapack` built in) | `-DCMAKE_PREFIX_PATH=/opt/local` (already in step 1); if needed `-DLAPACK_ROOT=/opt/local` |
| PaStiX link fails (e.g. due to spm/scotch ABI) | **Fallback:** SPOOLES 2.2 from netlib as the default solver (`wget https://www.netlib.org/linalg/spooles/spooles.2.2.tgz`) with SPOOLES=ON — SPOOLES requires LP64 (32-bit), so it fits the same LP64 configuration |
| PaStiX+LP64 does not behave as expected at runtime | take the reference comparisons in `test/` seriously; fall back to SPOOLES if necessary |
| CMake 4.4.4 vs. custom find modules | `FindARPACK.cmake` / `FindSPOOLES.cmake` are simply structured; the SPOOLES module is not loaded at all without SPOOLES=ON |

## Not included (deliberate decisions)

- No `gnu-ilp64` build: the MacPorts dependencies are LP64 only (32-bit), not ILP64-capable.
- No `make`-based build (`src/Makefile`): Intel/Linux-certified, outdated, SPOOLES/LAPACK/ARPACK paths in the Makefile are hardwired for Linux.
- No `cmake --install`: there are no install rules.

## Open items for the future

- If the user wants a globally compiled CalculiX with PaStiX in ILP64, that would be a separate project with fresh ILP64 dependencies (BLAS/LAPACK/ARPACK/solver) — outside the scope of this plan.
- Adding packaging/`install` rules to `CMakeLists.txt` — not part of this plan.
