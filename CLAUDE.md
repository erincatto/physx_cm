# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A monorepo bundling several independent NVIDIA physics SDKs, each self-contained
with its own build system, dependencies, and versioning:

| Directory   | What it is                                                        | Build system |
|-------------|-------------------------------------------------------------------|--------------|
| `ovphysx/`  | Standalone C API + Python bindings for USD physics simulation     | pip / ctypes |
| `physx/`    | PhysX SDK — C++ real-time physics engine (CPU + CUDA GPU)         | CMake via packman |
| `omni/`     | Omniverse PhysX extensions for Kit-based apps                     | premake / repo |
| `blast/`    | Blast SDK — destruction and fracture                              | premake / repo |
| `flow/`     | Flow SDK — fluid and fire                                         | premake / repo |

The subprojects do not share code or a top-level build. Work within one
subdirectory at a time; each has its own README.

**`ovphysx/` is the primary working area** (see `AGENTS.md`, and recent git
history). It is where day-to-day changes land. The rest of this file focuses
there; treat `physx/`, `omni/`, `blast/`, `flow/` as large upstream vendored
trees you touch only when a task explicitly names them.

## ovphysx architecture

ovphysx is a thin, self-contained physics library: load a USD scene, step it,
exchange state as zero-copy DLPack tensors, with no Omniverse Kit required. The
code in this repo is **the interface layer only** — the native library
(`libovphysx.so` / `ovphysx.dll`) is pre-built and bundled into the wheel or
downloaded from GitHub Releases. Nothing under `ovphysx/` compiles a native lib
from this checkout.

Three layers, top to bottom:

1. **C ABI** — `include/ovphysx/ovphysx.h` (~2100 lines). The stable contract.
   Everything crosses this boundary. Synchronous calls return
   `ovphysx_result_t`; async calls return `ovphysx_enqueue_result_t` with an
   `op_index`. Read the header's top comment block before changing bindings.
2. **ctypes bindings** — `python/ovphysx/_bindings.py`. Mechanical
   `ctypes` mirror of the C ABI: function prototypes, struct layouts, handle
   types. No native compilation — Python-side changes need no build step.
3. **High-level API** — `python/ovphysx/api.py` (~2600 lines). The `PhysX` and
   `ContactBinding` classes users actually call. Wraps handles, marshals
   tensors, enforces the execution model.

### Execution model (governs api.py and the C header)

- **Stream-ordered**: operations execute in submission order with sequential
  consistency. Writes from op N are visible to op N+1, so dependent calls need
  no explicit sync. `wait_op()` is only required before reading results outside
  the stream (CPU/GPU) or before exit. Preserve this contract when adding APIs.
- **Thread safety**: multiple `PhysX` instances are fully independent and
  thread-safe; a single instance is **not** thread-safe.
- **Tensors via DLPack**: state crosses as DLPack tensors (`dlpack.py`,
  `_dlpack_utils.py`), zero-copy interop with NumPy (CPU) and PyTorch (GPU).
  numpy/torch are deliberately **not** dependencies — users bring their own.

### Import-time behavior (subtle, in `__init__.py`)

- **Lazy native loading**: `import ovphysx` loads no native code. Only pure
  Python (enums, config, version) is available until first access of a native
  attribute (`PhysX`, `ContactBinding`, …), which triggers `_bootstrap_native()`
  via `__getattr__`. Bootstrap failures are cached and re-raised so callers see
  the real error. Call `bootstrap()` to force loading at a chosen moment.
- **Schema-path registration is a side effect of import**: `register_schema_paths()`
  runs on import and appends ovphysx's USD plugin path to
  `OV_PXR_PLUGINPATH_2511`. This must happen before the first USD stage open, or
  USD's `PlugRegistry` locks in without ovphysx's schemas and applied-schema
  lookups silently fail. It is pure-Python and idempotent. When a process mixes
  ovphysx with another OV USD-aware subsystem, schema paths must be registered
  before any USD stage/registry access.
- The `__all__` list in `__init__.py` is the single source of truth for the
  public API; a drift guard in `_bootstrap_native_impl()` asserts every entry is
  actually injected. Keep `__all__` and the injected globals in sync.

### ovphysx layout

- `include/ovphysx/` — public C headers (`.h`) plus experimental C++ wrappers
  (`experimental/*.hpp`).
- `python/ovphysx/` — the Python package. Files prefixed `_` (`_bindings.py`,
  `_dlpack_utils.py`, `_pxr_stubs.py`) are internal. Note: repo style forbids
  leading underscores on *symbol* names, but these established internal *module*
  filenames are pre-existing and kept as-is.
- `tests/python_samples/` — runnable Python samples (`hello_world.py`,
  `clone.py`, `tensor_bindings.py`, `contact_binding.py`, …). Doubles as the
  usage reference.
- `tests/c_samples/` — standalone C/C++ samples, each with its own
  `CMakeLists.txt` using `find_package(ovphysx)`.
- `tests/data/*.usda` — USD scenes the samples load.

## Common commands

### ovphysx — Python

```bash
# Editable install against a bundled/pre-built native lib
pip install -e ovphysx/python

# Run a sample (samples are the de-facto tests)
python ovphysx/tests/python_samples/hello_world.py

# Version lives in ovphysx/VERSION and is read at build/runtime
cat ovphysx/VERSION
```

The Python package is pure ctypes over the C ABI, so editing `python/ovphysx/`
requires no compile step — just re-run. The wheel (`python/setup.py`,
`pyproject.toml`) is a platform wheel that bundles the native `.so`/`.dll` and
its USD/carbonite plugins under `lib/` and `plugins/`; a `_version.py` is
generated at wheel-staging time.

### ovphysx — C/C++ samples

Build against an extracted/pre-built SDK (from GitHub Releases), not from a raw
source checkout:

```bash
cmake -B build -S ovphysx/tests/c_samples/hello_world_c -DCMAKE_PREFIX_PATH=/path/to/ovphysx-sdk
cmake --build build
```

### physx (C++ SDK)

```bash
# From physx/ — lists presets when run with no arg, then generates a solution
physx/generate_projects.bat        # Windows
physx/generate_projects.sh         # Linux
# Build from the generated solution/makefile under physx/compiler/<preset>/
```

Requires CMake ≥ 3.21, Python ≥ 3.5, and (for GPU) CUDA Toolkit 12.8. Packman
downloads dependencies and GPU binaries on demand from CloudFront. Platform
details: `physx/documentation/platformreadme/{windows,linux}/`.

### blast / flow / omni

Each uses NVIDIA's `repo`/premake tooling driven by a `build.bat` / `build.sh`
at its root (e.g. `blast/build.bat` → `_build/<platform>/<config>/`). See each
subdirectory's README.

## Conventions

- **Contributions require a DCO sign-off**: commit with `git commit -s` (see
  `CONTRIBUTING.md`). Cosmetic-only patches (formatting, whitespace) are
  rejected.
- Source files carry SPDX headers: `BSD-3-Clause` for source; pre-built binaries
  and wheels are under the NVIDIA Omniverse License.
- ovphysx is **pre-1.0** — the C ABI and Python API may still change. When
  touching the ABI, update `ovphysx.h`, `_bindings.py`, and `api.py` together
  and keep the execution-model and thread-safety contracts intact.
