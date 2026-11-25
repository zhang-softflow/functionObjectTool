# foamFO — search OpenFOAM function objects and generate entries

English | [日本語](README.ja.md)

OpenFOAM (ESI, openfoam.com) ships 130–140 function objects, and finding the mandatory keywords, the `libs` entry and a working example usually means digging through headers and tutorials.
**foamFO** parses the documentation comments in the OpenFOAM sources (`Description`, `Usage`, `\verbatim` examples, `\table` parameter tables), the tutorials and `etc/caseDicts/postProcessing`, and turns them into a searchable database, a CLI and an offline HTML catalog.

| File | Content |
|---|---|
| `foamFO` | CLI (Python ≥ 3.8, standard library only) |
| `catalog.html` | Searchable catalog — just open it in a browser (offline, single file) |
| `foDatabase.json` | Bundled database (generated from OpenFOAM-v2606) |
| `catalog_template.html` | Template of the catalog |
| `skill/openfoam-functionobjects/` | Claude Code skill |
| `install.sh` | Install helper (skill link, PATH) |

## Installation

```bash
git clone https://github.com/zhang-softflow/functionObjectTool.git
cd functionObjectTool
./install.sh            # link the skill into ~/.claude/skills and print the PATH line
./install.sh --bashrc   # append the PATH setting to ~/.bashrc (with backup)
./install.sh --uninstall
```

If you use a version other than v2606, build the database for it (takes a few seconds):

```bash
source /usr/lib/openfoam/openfoam2412/etc/bashrc
foamFO build            # -> ~/.local/share/foamFO/foDatabase-openfoam2412.json
```

The database matching the basename of `$WM_PROJECT_DIR` is picked up automatically.

- **Lookup order:** `$FOAMFO_DB` → `~/.local/share/foamFO/foDatabase-<version>.json` → bundled `foDatabase.json`.
- **Version mismatch:** a warning is printed when the bundled database does not match the environment (silence it with `FOAMFO_QUIET=1`).

## Usage

| Command | Description |
|---|---|
| `foamFO list [-g field]` | List function objects (F = `#includeFunc` template, T = used in tutorials) |
| `foamFO list --groups` | Count per group |
| `foamFO search wall shear` | Full-text search in names, descriptions, parameters and synonyms (`drag`, `flow rate`, `residual`, …) |
| `foamFO show forces [--inherited]` | Description, example, parameter table (mandatory in red), notes, `#includeFunc` templates, tutorials |
| `foamFO example forces -m 3` | Print real usage blocks from the tutorials (needs the OpenFOAM environment or `--src`) |
| `foamFO add yPlus -case .` | Write `system/yPlus1` with the mandatory entries |
| `foamFO add forces --full --stdout` | Also list optional entries as comments |
| `foamFO add fieldMinMax -n minMaxU --insert` | Write `system/minMaxU` and add `#include "minMaxU"` to `functions{}` in controlDict (original kept as `controlDict.bak`) |
| `foamFO build [--src DIR] [-o FILE]` | Build the database |
| `foamFO html` | Regenerate `catalog.html` |

- Placeholders such as `<word>` or `<wordList>` remain in the generated entry, and foamFO prints a warning listing them.
- Entries are derived from the header example (`// Mandatory entries`, `// Optional entries`, `// Conditional ...`) and the `Reqd` column of the parameter table.

## catalog.html

Open it in any browser. No OpenFOAM installation is needed.

- **Left pane:** incremental search (press `/`) and group filters.
- **Right pane:** copyable example, parameter table, inherited entries, and the tutorials that use it.
- **Deep link:** add the name to the URL, e.g. `catalog.html#forces`.

## Claude Code skill

`install.sh` links it as `~/.claude/skills/openfoam-functionobjects`.
Ask for example "output y+ and the drag coefficient on the walls": the skill checks the documented syntax with foamFO before writing the `functions{}` entries. To invoke it by name, use `/openfoam-functionobjects`.

## Built-in OpenFOAM helpers worth knowing

### `#includeFunc`
```cpp
functions
{
    #includeFunc Q
    #includeFunc mag(U)                         // one argument -> 'field' / 'fields'
    #includeFunc patchAverage(name=outlet,p,U)  // named args override entries, others -> fields
    #includeFunc flowRateFaceZone(name=fZone1)
}
```
- **No spaces inside the arguments.** They are read as a single word, so `patchAverage(name=outlet, fields=(p U))` fails.
- **Search order:** `system/<name>` of the case → `~/.OpenFOAM/<ver>/caseDicts/postProcessing` → `$WM_PROJECT_DIR/etc/caseDicts/postProcessing`.
  Put your own templates in `~/.OpenFOAM/<ver>/caseDicts/postProcessing/` to use them in every case.

### `postProcess` / `<solver> -postProcess`
```bash
postProcess -list                          # available #includeFunc templates
postProcess -func 'mag(U)' -latestTime
postProcess -funcs '(Q vorticity)'
simpleFoam -postProcess -func wallShearStress -latestTime   # needs solver models
```

### `foamGetDict`, `foamSearch`, `foamDictionary`
```bash
foamGetDict forcesIncompressible                      # copy a template into system/
foamSearch $FOAM_TUTORIALS functions controlDict      # collect entries from tutorials
foamDictionary system/controlDict -expand             # resolves #include, NOT #includeFunc
```

## Limitations
- Targets the **ESI edition (openfoam.com, vYYMM)**. The Foundation edition (openfoam.org) is not tested.
- Function objects with incomplete header documentation have no example or table. `foamFO build -v` lists them; for those, read the header shown by `foamFO show`.
- Lagrangian cloud function objects (`cloudFunctions`) are hidden by default (use `-a` or the catalog checkbox).
- Tested:
  - Database build on v2606 (source tree) and v2412 (packaged).
  - Generated entries run with v2512 (simpleFoam/pitzDaily).

## License
GPL-3.0-or-later (see [LICENSE](LICENSE)).
The bundled `foDatabase.json` and `catalog.html` contain documentation text extracted from the OpenFOAM sources (GPL-3.0-or-later, © OpenCFD Ltd. and others).

OPENFOAM® is a registered trade mark of OpenCFD Limited, producer and distributor of the OpenFOAM software via www.openfoam.com. This offering is not approved or endorsed by OpenCFD Limited.
