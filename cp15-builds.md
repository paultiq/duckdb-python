# cp15 (Python 3.15) build support

Status of adding cp15 wheel builds to duckdb-python. Working notes for the fork `paultiq/duckdb-python` on `main`.

Status as of 2026-08-25 (session with user paultiq on `main`):

## Done in this branch session

- `pyproject.toml`
  - Added `Programming Language :: Python :: 3.15` classifier.
  - Added a new `[[tool.uv.index]]` named `scientific-python-nightly`, url `https://pypi.anaconda.org/scientific-python-nightly-wheels/simple`, `explicit = true`.
  - Extended `[tool.uv.sources]` so `pandas` and `pyarrow` resolve from that index when `python_version >= '3.15'`; PyPI is the fallback on earlier cpus.
  - Split pandas/pyarrow entries in `test` and `bench` dependency groups: `<3.15` pins stay on PyPI, `>=3.15` pins use `.dev0` lower bounds so uv accepts nightly `+g<sha>` tags.
- `.github/actions/build-wheel/action.yml`
  - Bumped `pypa/cibuildwheel@v3.2` to `@v4.2.0` (Aug 5 2026 release; ships cp15 by default, enables `delvewheel` on Windows and `abi3audit` by default).
- `.github/workflows/packaging_wheels.yml`
  - `seed_wheels` python unchanged at `[cp314]` (per user direction on second pass).
  - `build_wheels` python changed from `[cp311, cp312, cp313]` to `[cp311, cp312, cp313, cp315]`. cp15 ships from this fan-out, builds against cp14's ccache.

## Explicitly NOT changed (user direction this session)

- `.github/workflows/targeted_test.yml` — kept at 3.10-3.14. User said cp15 is pre-release, do not advertise as a manual test option yet.
- `seed_wheels` matrix — kept at cp14. User did not want seed_wheels modified.

## win32 ARM64 + pyarrow resolution (resolved 2026-08-26)

- **Root cause was NOT a cp15-only gap.** pyarrow has never shipped `win_arm64` wheels on PyPI (25.0.1 has only `win_amd64`; arm64 wheels are macOS-only) OR on the nightly index. Pre-cp15 `uv lock` still succeeded because PyPI (default index) tolerates "no wheel for this env" in universal mode — pyarrow locks globally and just isn't installed on win32 ARM64.
- The cp15 change broke it because `[tool.uv.sources]` routes `pyarrow{>=3.15}` to `scientific-python-nightly` with `explicit = true`. An explicit-index source must provide a wheel for every env it applies to; nightly has no win_arm64 pyarrow, so uv hard-fails for the cp15 + win32 ARM64 split. Breakage is from the explicit source-routing mechanism, not pyarrow's win_arm64 absence per se.
- **Fix applied (pyproject.toml:50):** added `; sys_platform != 'win32' or platform_machine != 'ARM64'` marker to the `pyarrow` entry in the `[all]` extra, matching the pattern already used in `test`/`bench`/`stubdeps`. `uv lock` then resolves 170 packages (pyarrow 26.0.0.dev164 nightly for cp15, 25.0.1 PyPI for <3.15). No cibuildwheel `skip` needed; no env bounding needed; cp15-win_arm64 stays in the lockfile for duckdb core + non-pyarrow deps.
- The earlier deferred plan (bound win32 ARM64 env to `<3.15` + `skip = "cp315-win_arm64"`) was the wrong shape — would have dropped cp15-win_arm64 entirely. Not applied.
- Comment on the line: `TODO: Remove after 3.15.0 release` (remove the win32 ARM64 marker once pyarrow ships win_arm64 wheels).
- pandas in `[all]` stays unconditional — it resolves fine on win32 ARM64 (nightly has win_arm64 pandas cp315).
- `uv.lock` — user said they will handle regen themselves.

## cp15 native test deps without wheels (resolved 2026-08-26)

- **Precedent:** the 3.14 prerelease (commit d16cffa, Oct 2025) excluded deps lacking cp314 wheels via `python_version < '3.14'` markers — `pyarrow` and `torch` were marked `<3.14`; deps that had cp314 wheels (adbc, gcsfs, polars) stayed bare. Same per-package exclusion pattern is the project's established approach.
- **User stance for cp15:** "Exclude where it's not available. Since it's available for manylinux, keep that." cp15 is wheel-build-only in CI; do NOT source-build duckdb locally on cp15 (MAX_PATH). If a prebuilt duckdb cp15 wheel isn't installable locally, skip testing on cp15 — same stance as free-threading.
- **cp315 wheel coverage on PyPI (stable, Aug 2026):** numpy yes (win+linux); coverage yes (win+linux); pandas/pyarrow via nightly (win+linux). NO cp315 wheels anywhere: adbc-driver-manager, polars, grpcio (transitive via gcsfs). linux-only: torch (stable pytorch-cpu), pytest_codspeed (also macosx arm64, not win). psutil has no cp315 wheel but source-builds fine (small C ext, left bare).
- **Fixes applied (test group, pyproject.toml):**
  - torch: split the `>=3.14` entry — kept `>=3.14 and <3.15` for cp314; added `torch>=2.10.0; python_version >= '3.15' and sys_platform == 'linux'` (cp15 linux only).
  - adbc-driver-manager: `>=3.10` entry bounded to `and python_version < '3.15'`.
  - gcsfs: bounded to `python_version < '3.15'` (drops grpcio transitive on cp15).
  - polars: `polars>=1.33.0; python_version < '3.15'` (no cp315 wheel anywhere).
  - pytest_codspeed: `; python_version < '3.15' or sys_platform != 'win32'` (keep cp15 linux/macosx, drop win).
- **Not changed:** `[all]` extra's adbc-driver-manager (line 52) left bare — `uv sync` doesn't install `[all]`, and PyPI default index is lenient so `uv lock` passes. `duckdb[all]` install on cp315 would still fail on adbc; revisit if users hit it. `bench` group not yet mirrored (not installed by default `uv sync`); its polars/pytest_codspeed still need the same markers for `--group bench` on cp15.
- **Verified:** `uv lock` resolves 170 packages; `uv sync --no-install-project` succeeds on cp315 win_amd64 (all dev/test deps install, no duckdb source build). Full `uv sync` still triggers the editable duckdb build (MAX_PATH on Windows) — out of scope per user stance.

## Live validation in flight at hand-off

- Test run kicked off: `gh run view 32906126223` on fork `paultiq/duckdb-python`, branch `main`.
- Mode: `testsuite=none` (wheel build only, no pytest). Verifies cibuildwheel v4.2.0 produces wheel artifacts for every matrix entry including cp15.

## Open questions / risks for the next pass

- cibuildwheel 4.x: `delvewheel` default on Windows and `abi3audit` no-op likely fine, but the first Windows cp15 run should confirm no deltas in our wheel manifest or abi tags.
- pandas lower-bound choice (`>=3.0.0.dev0`) is the best guess for nightly acceptance; if `uv lock` complains, loosen it (e.g. drop dev segment).
- Once pyarrow cp15 win_arm64 windows shows up upstream, remove the `[all]` pyarrow win32 ARM64 marker (pyproject.toml:51). The deferred env-bound/skip plan was NOT applied.

## Windows Debug editable build requires python_d.lib (NOT cp15-specific, 2026-08-27)

- Symptom: local `uv sync` (editable, Debug) on Windows cp315 fails at LINK of `_duckdb.cp315-win_amd64.pyd`:
  `LNK2019: unresolved external symbol __imp__Py_NegativeRefcount / _Py_INCREF_IncRefTotal / _Py_DECREF_DecRefTotal` (in `numpy_array.obj`, via `Py_INCREF`/`Py_DECREF`).
- Root cause: `CMakeLists.txt:9` sets `CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>"` — i.e. `/MTd` (debug runtime) in Debug builds. `/MTd` defines `_DEBUG` -> `pyconfig.h` defines `Py_DEBUG` -> `Py_REF_DEBUG`. Python 3.15's `refcount.h` then makes inline `Py_INCREF`/`Py_DECREF` call `_Py_NegativeRefcount` / `_Py_INCREF_IncRefTotal` / `_Py_DECREF_DecRefTotal`. Those symbols live ONLY in `python315_d.lib` (debug import lib). `pyconfig.h` auto-links `python315_d.lib` only under `Py_BUILD_CORE` (extensions don't qualify), so CMake links the release `python315.lib` — mismatch -> unresolved.
- This is **not cp15-specific and not introduced by the cp15 work.** It fails any Windows *Debug editable* build whose Python lacks `python_d.lib`. uv-managed Python ships only `python31X.lib` (release) for EVERY version — no `python31X_d.lib` for 3.13/3.14/3.15. So cp313/cp314 Debug editable on Windows with uv Python would fail identically.
- It does **NOT** affect CI Windows wheel builds: cibuildwheel uses Release config -> `MultiThreaded` (`/MT`) + `NDEBUG` -> no `Py_REF_DEBUG` -> links `python31X.lib` cleanly. Shipped Windows wheels are fine.
- The project's Windows Debug dev path is designed around having a debug Python import lib. uv Python doesn't provide one.
- Fixes (none are pyproject changes; none were applied):
  - Use a Python 3.15 with debug binaries (python.org debug install / build from source / conda debug Python) so `python315_d.lib` exists.
  - Or do a non-debug local build: `uv sync --no-editable --no-build-isolation -v --reinstall -p 3.15` (Release, links release lib fine). Not editable; C++ changes need full rebuild.
  - Do NOT "fix" by forcing `MultiThreaded` in Debug — that's an intentional upstream setting (CMakeLists.txt:9), tied to ASAN/debug-allocator interplay (comment near line 91). Diverging is out of scope.
- Consistent with user stance: cp15 local source-build is not supported via uv Python; cp15 deliverable is the CI wheel. If no prebuilt cp15 wheel is installable locally, skip local cp15 testing (like free-threading).

## nanobind 3.0.0 API break (NOT cp15-specific, fixed 2026-08-27)

- nanobind 3.0.0 released 2026-08-22. It changed `arg_t::none()` to take NO arguments; the 2.x API was `none(bool)`. The duckdb-python bindings call `none(false)`/`none(true)` (6 sites in `src/duckdb_python.cpp`, plus `src/pyconnection.cpp`). Upstream `main` is also broken (same unbounded `nanobind>=2.0` pin, same 2.x calls) — not yet fixed upstream.
- nanobind ships a `py3-none-any` universal wheel (installs on any Python 3 incl. cp315); it does NOT source-build. The `nanobind-static.lib` compiled during the duckdb build is nanobind's bundled C++ runtime compiled into the extension, not nanobind installing from sdist.
- uv resolved 3.0.0 only because the pin was unbounded (`nanobind>=2.0`). Not cp15-related.
- Fix applied (pyproject.toml :67, build-system requires): pinned `nanobind>=2.0,<3.0`. Committed in `ce85b04`. NOTE: the dev-group pin at :342 is still `nanobind>=2.0` (UNBOUNDED) as of 2026-08-27 — only :67 was pinned. :67 is what cibuildwheel's `build[uv]` frontend uses (build-system requires), so CI wheel builds are safe. :342 affects local `uv sync` dev resolution; if a local dev sync pulls nanobind 3.0 it would break the bindings compile the same way. Consider bounding :342 too.
- Reversible once the bindings are ported to the 3.0.0 API (drop `.none(false)` calls — they're the default — and change `.none(true)` to `.none()`). Worth raising upstream since `main` is currently unbuildable from a fresh resolve.

## CI versioning failure on fork (root cause + fix, 2026-08-27)

- **Symptom**: fork's first cp15 wheel-build run (`gh run view 32906126223` on `paultiq/duckdb-python`, `testsuite=none`) failed before building anything. Every job (sdist + all wheel seeds) died at version detection: `ValueError: Invalid version format: 0.0.1.dev1 (expected X.Y.Z, X.Y.Z(a|b|rc)N or X.Y.Z.postN)`; `version.tag: 0.0.1.dev1`, distance = commit count (1182).
- **Root cause**: the fork's GitHub remote had ZERO version tags (forks don't copy tags). CI `actions/checkout@v4` (fetch-depth 0, no `fetch-tags`) checks out the fork -> `git describe --match v*.*.0` found nothing -> setuptools_scm used `fallback_version = "0.0.1.dev1"` (pyproject.toml:100) -> custom `version_scheme` (`duckdb_packaging/setuptools_scm_version.py:63` -> `_bump_dev_version` -> `parse_version` in `duckdb_packaging/_versioning.py:15`) rejects the `.devN` suffix (regex expects `X.Y.Z` forms only) -> RuntimeError. Local clone worked because upstream tags had been fetched locally.
- **This was the SOLE blocker.** The cp15 pyproject fixes were already committed in `ce85b04 exclude win32_arm64` (nanobind `>=2.0,<3.0` at :67; `[all]` pyarrow win32_arm64 marker at :51 comment "Remove after 3.15.0 release"; cp15 test-dep markers for adbc/gcsfs/polars/pytest_codspeed/torch). cibuildwheel builds Release wheels (`/MT` + `NDEBUG`), so no Windows Debug `Py_REF_DEBUG` link issue, and the nanobind `<3` pin means bindings compile against the 2.x `none(bool)` API.
- **Debugging procedure** (future fork CI runs that fail at "Build sdist" / version detection):
  1. `gh run view <id> -R paultiq/duckdb-python --log-failed | grep -E "version_scheme|Invalid version|0.0.1.dev1"` — confirms fallback was hit (version.tag = 0.0.1.dev1, distance = total commit count).
  2. `git ls-remote --tags origin | grep -E 'refs/tags/v[0-9]'` — if empty, the fork lacks version tags (the cause).
  3. `git describe --dirty --tags --long --abbrev=40 --match 'v*.*.0'` locally — should resolve (e.g. `v1.5.0-367-g...`); if it fails locally too, tags aren't fetched (`git fetch upstream --tags`).
  4. Confirm the cp15 pyproject fixes are on the branch being built (`git show HEAD:pyproject.toml` for `nanobind>=2.0,<3.0` at :67 and the `[all]` pyarrow marker at :51) — `gh workflow run` builds from the committed ref, NOT the working tree.
- **Fix applied by user**: pushed upstream `v*` tags to the fork's GitHub remote (`git push origin --tags`). Verified: 17 `v*` tags now on origin (v1.3.0 ... v1.5.5); `git describe --match v*.*.0` -> `v1.5.0-367-gce85b04d4` -> package version `1.6.0.dev367` (main-branch versioning bumps minor+1).
- **Alternative one-off fix** (no git ops): `gh workflow run packaging.yml -f testsuite=none -f set-version=v1.5.0` — `set-version` flows to `OVERRIDE_GIT_DESCRIBE` (`packaging_sdist.yml:66`, `build-wheel/action.yml:71`) and bypasses tag-based detection (`forced_version_from_env` in `setuptools_scm_version.py:94`).
- **Re-run command** (now that tags are pushed): `gh workflow run packaging.yml -f testsuite=none -R paultiq/duckdb-python`, then `gh run list -R paultiq/duckdb-python --workflow=Packaging --limit 1` and `gh run view <new-id> -R paultiq/duckdb-python`.
- **Expected outcome**: sdist builds with version `1.6.0.dev367`; cp315 wheel jobs build Release wheels. cp315 win_arm64 wheel: the `build_wheels` matrix still fans out cp315 x windows-11-arm; cibuildwheel will attempt it. The `[all]` pyarrow marker only fixes `uv` lockfile resolution, NOT cibuildwheel — but `testsuite=none` doesn't install `[all]`/test deps, and the duckdb wheel itself doesn't need pyarrow at build time, so cp315-win_arm64 should build. Watch the first run; add `skip = "cp315-win_arm64"` under `[tool.cibuildwheel]` only if it fails.
- **Robustness note (NOT applied)**: `fallback_version = "0.0.1.dev1"` is unparseable by `version_scheme`, so any fallback path (shallow clone, missing tags) yields a confusing RuntimeError instead of using the fallback. Upstream could harden by setting `fallback_version = "0.0.0"` or having `version_scheme` handle the `.devN` form. Out of scope for the fork's cp15 task; flagged for awareness.
