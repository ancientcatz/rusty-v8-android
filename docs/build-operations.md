# rusty-v8-android: build workflow operations manual

> This document is intended for future maintainers of this workflow (Claude Code sessions or humans).
> Purpose: codify how we build all rusty_v8 versions, troubleshoot, cache, and advance, to avoid retracing old ground.

---

## 1. Background

- This repository: a public repo that cross-compiles `librusty_v8.a` from [denoland/rusty_v8](https://github.com/denoland/rusty_v8) source and publishes it to GitHub Releases.
- Target platforms: Android (`aarch64` + `x86_64`). The official rusty_v8 releases do not provide these targets.
- Downstream consumers skip local V8 compilation via `RUSTY_V8_MIRROR=https://github.com/ancientcatz/rusty-v8-android/releases/download`.
- Goal: build all rusty_v8 versions (130.0.7 → 147.1.0 and subsequent upstream releases).

Core files:
- `.github/workflows/build.yml` — main workflow, triggered by `workflow_dispatch`
- `.github/workflows/sync.yml` — automatically tracks upstream (daily at 08:00 UTC, filters by `CUTOFF` timestamp then triggers build)

Downstream linking concerns (not handled here): libc++ symbol conflicts, `__clear_cache` static linking, etc. are handled by the consuming cdylib (`--allow-multiple-definition` + version script hiding libc++ symbols, build.rs statically linking compiler-rt builtins).

Build toolchain:
- Android NDK `r29` with `android-api-level = 24`.

---

## 2. Working method

### 2.1 Reason about consequences *before* editing the workflow

Do not "change it and see what happens." Before modifying, enumerate:
- Which targets × which V8 versions does this change affect?
- Is the currently-running job hitting the same problem as the driver for this change? If yes, it will certainly fail — cancel to save time. If not (or it has not reached that step yet), let it finish. "Because I changed it, it will fail" is wrong — my edits only affect runs triggered *after* the edit.
- Which case does the next cold start take? Which case does a warm cache take?
- Will it pollute other targets' caches?

**Past traps**: cancelling a still-running Build V8 job without reasoning; releasing before all jobs are green; retrying the same error multiple times with the same fix.

### 2.2 When a change may affect old versions, regression-test

The workflow logic is shared across all versions — one change may break old versions. Past incidents:
- After switching from crates.io dependency to git clone source build, old versions (v130-v139) used to pass via crates.io but post-clone some transitive deps (`temporal_rs`, etc.) were re-resolved by cargo to incompatible versions.
- src-cache key changes make the first checkout cold for every version.
- Toolchain upgrades (NDK / Clang / sysroot) may be compatible with new V8 but break old V8.

Rules:

1. **Think about the impact surface first**: does it affect only new versions? All versions? Only one target?
2. **If it could theoretically break old versions, proactively pick one or two old versions to regression-test.** Strategy: the earliest supported version + one middle version. No need to test all, but "theoretically may be affected" must not be skipped.
3. **Regression testing does not require the full pipeline** — reuse existing caches and retry; whether new errors appear shows in minutes.

Skipping regression testing and "ship the new version first" lets old versions silently bit-rot; by the time you next build them, you can no longer tell which change broke them.

### 2.3 Inspect three kinds of signals during patrol, not just red/green

1. **Pass/fail**: conclusion + failed step
2. **Cache status**: which caches hit / miss / how large, and **whether size grows across retries**
3. **Per-step duration**: compared to past successful runs, an inexplicably longer segment indicates a problem

**Example**: v8-deps cache stuck at 115MB across several retries when it should be 300-400MB — save was not running (our bug: `if: cache-hit != 'true'` permanently satisfies `cache-hit == true` after the first save, skipping subsequent saves). Partial cache is not bad; stagnant size is bad.

### 2.4 Monitoring: gh watch + ScheduleWakeup as dual safeguard

```bash
gh run watch <run_id> --interval 60 --exit-status   # 60s poll, notifies on completion
```

Add a timer (15-30 min) as backup in case watch silently fails. Whichever fires first triggers continuation; avoid idle waiting.

**Important: `--interval 60`**. The default 3s is too aggressive — each matrix poll takes ~2 API calls (run + jobs), 3s interval = ~2400/h; two concurrent watches plus manual queries can hit the 5000/h REST limit. 60s interval = 120/h, two watches only 240/h, far below the limit. V8 build steps routinely take tens of minutes; 60s granularity is more than enough.

At 60s intervals API usage stays low; no need to artificially cap the number of concurrent watches (different from the parallel build cap in section 2.6). But a few disciplines remain:
- **Before re-attaching watch, confirm the old background task has exited** to avoid zombie stacking.
- Use batch queries: `--json jobs` pulls all info in one call instead of step-by-step.
- In extreme cases (default 3s misuse or excessive concurrent queries) hitting rate limit requires ~1h wait; during that window all watches 403 out and manual queries are useless.

### 2.5 Drive forward autonomously, do not wait for prompts

After each patrol, check whether the task list is fully complete; if not, **schedule the next patrol** and **proceed directly to the next step**.

### 2.6 Parallel build cap: 2

GitHub Actions limits runners; two versions in parallel is safe. More will queue and actually slow down.

### 2.7 Release discipline

The `release` job only runs after `build` is fully green (the workflow's `needs: build` guarantees this).
If you find release was skipped but artifacts exist, do not manually upload — either fix the workflow or rerun.

---

## 3. Cache strategy

### 3.1 actions/cache pitfalls (common to v4/v5)

- `!path` exclusion pattern is **unreliable for directories**. Tested `!rusty_v8-src/third_party/rust-toolchain` did not exclude at all; the Darwin bindgen saved on macOS still ended up in cache and Exec-format-error'd on Linux. Correct approach: `rm -rf` after restore, then `rm -rf` again before save.

- `actions/cache/save@v4` **silently skips on duplicate key** (no-op, no error). To force overwrite: `gh api -X DELETE "repos/.../actions/caches?key=X"` first, then save. Requires `permissions: actions: write`.

### 3.2 Current key conventions

```
rusty-v8-src-v<version>                              # Source (per version, shared across targets)
android-ndk-r29                                      # NDK (shared across versions)
v8-deps-<target>-v<version>-api<api>                  # Android gn_out + clang
```

Fixed key, no run_id suffix. Force refresh via API DELETE before save. per-(target, version) isolation avoids races between matrix jobs.

### 3.3 fail-fast causes stratified cache completeness

**Status: as expected, no fix needed.** With matrix targets, one failure cancels the others. Each target's cache completeness depends on where it was when cancelled:

| Outcome | gn_out cache |
|---------|-------------|
| success | complete |
| failure after `[N/N] AR librusty_v8.a` | complete (build finished, only top-level bindgen failed) |
| failure mid-build | partial |
| cancelled mid-build | partial (save still runs, but state is incomplete) |

On next retry, targets with complete cache finish in seconds (~2min); partial cache requires a few to tens of minutes of ninja rebuild — still far faster than the 60-110min cold start. This is **expected behavior** — the leftovers of a failed run are worth keeping, iterate until fully green.

### 3.4 Cache signal diagnostics

Partial cache is a feature (resume next time), not a bug. What to watch for are concrete problems like **cache content not changing when it should** or **OS mismatch**.

- **Cache size stagnant across retries**: save logic bug (e.g. `if: cache-hit != 'true'` locked forever, or save step got cancelled before running), not "cache rot".
- **Cache content OS mismatch**: Darwin binary ended up in Linux cache → Exec format error. See 4.1 for the rust-toolchain handling.
- **Ninja rebuild step count `[XX/YYYY]`**: YYYY is stable (~4239 for v140+); a small XX starting point means ninja considers most things up-to-date, cache reuse is working.
- **`already downloaded` vs `Downloading`**: indicates whether clang / rust-toolchain is being reused.

---

## 4. rusty_v8 cross-compilation pitfalls

### 4.1 rust-toolchain is OS-specific

`third_party/rust-toolchain/bin/` contains `bindgen`, `rustc`, etc. downloaded by V8 itself, bound to the OS.
**Cannot** be shared across OSes. Must `rm -rf` before src cache save.

### 4.2 V8 internal bindgen vs rusty_v8 top-level bindgen

**Two completely independent** bindgen invocations with mutually exclusive configurations:

| | When triggered | libclang source | Args source |
|--|---------|-------------|---------|
| V8 internal | ninja invokes, mid V8 build | `third_party/rust-toolchain/bin/bindgen` (V8-bundled) | CLI arg `--libclang-path <V8-bundled clang>` |
| rusty_v8 top-level | End of `cargo build`, after V8 build completes | From env `LIBCLANG_PATH` | bindgen crate reads `BINDGEN_EXTRA_CLANG_ARGS[_<target>]` |

V8 internal does not read env `LIBCLANG_PATH` (uses CLI), but **does** read `BINDGEN_EXTRA_CLANG_ARGS_<target>` (both bindgen CLI and crate read it).
So adding `--target=...` to env pollutes V8 internal's invocation of host tools (`clang_x64_for_rust_host_build_tools`) — causing errors like `-msse3 unsupported`.

**Solution**: for v140+ Android env, only inject `--sysroot` + `-isystem ...`, **no `--target`**. The bindgen crate derives target from `TARGET`; V8 internal CLI keeps its own `--target=x86_64-linux-gnu` (host) or `<target><api>` (target) unoverridden.

### 4.3 Breaking changes between V8 versions

```
v130-v139:
  NDK libc++ and Chromium libc++ are still compatible → -isystem NDK c++/v1 works.
  env BINDGEN_EXTRA_CLANG_ARGS may include --target (no V8 internal bindgen to pollute).
  LIBCLANG_PATH=/usr/lib/x86_64-linux-gnu

v140+:
  V8 introduces internal ninja bindgen → env must not include --target.
  Chromium-bundled libc++ joins the build, requires Clang 19+.
  LIBCLANG_PATH must be Clang 19+ (apt.llvm.org install llvm-19).
  -isystem switches to V8's bundled Chromium libc++: $V8_SRC/third_party/libc++/src/include

v142+:
  Chromium libc++ bump uses __builtin_clzg / __builtin_ctzg (Clang 19+ only).
  → NDK r26c's Clang 17 libclang does not recognize them at all.
  → must use llvm-19 (the v140+ approach covers this).

v145+:
  host rust toolchain depends on Debian bullseye sysroot to pass gn gen's assert.
  → run build/linux/sysroot_scripts/install-sysroot.py --arch=amd64 in Prepare Android NDK.

v152+:
  Top-level bindgen requires Clang 21.1+.
  → install LLVM 21 from apt.llvm.org; LIBCLANG_PATH=/usr/lib/llvm-21/lib.
```

### 4.4 Other files V8 build.rs needs patched in

rusty_v8's checkout slims down some files V8 expects to find; the workflow creates them manually:

```
build/android/*.pydeps                           # pydeps stubs
build/android/pylib/results/presentation/...pydeps
build/android/test_wrapper/...pydeps
build/rust/known-target-triples.txt              # v140+
build/linux/debian_bullseye_amd64-sysroot/       # v145+, downloaded via install-sysroot.py
```

Plus the linker in `.cargo/config.toml` (NDK's clang wrapper, choose `aarch64-` or `x86_64-` prefix by target).

---

## 5. Troubleshooting playbook

### 5.1 Look at panic location, do not guess

```
thread 'main' panicked at rusty_v8-src/build.rs:233:6
```

`build.rs:233` is the **top-level** bindgen generation location. If the log shows `[4239/4239] AR obj/librusty_v8.a` beforehand, V8 compilation succeeded and only the top-level bindgen failed.

vs

```
ERROR at //build/config/sysroot.gni:60:7
```

This is a gn gen stage failure — never entered ninja; V8 internal never started.

### 5.2 Look at which file in `-isystem` paths errored

```
third_party/libc++/src/include/...              → Chromium's new libc++ (V8-bundled)
android_ndk/.../sysroot/usr/include/c++/v1/...  → NDK's older libc++
/usr/include/c++/11/...                         → Linux host libstdc++
```

Including both libc++ simultaneously → conflict. Decide which one you want and remove the other from the include path.

### 5.3 Compare successful vs failed runs

Same V8 version, same workflow, sometimes one run passes and another fails — compare cache hit status, step durations, env vars in the logs. Common causes: cache content OS mismatch, runner scheduling noise, network jitter.

---

## 6. Current status (snapshot)

**This section is a point-in-time snapshot, not guaranteed to match the repository.**
Authoritative status: `gh release list` + `gh run list`.

As of 2026-04-13:
- ✅ 130.0.7 / 134.5.0 / 135.1.1 / 136.0.0 / 137.3.0 / 139.0.0 / 140.2.0 / 142.2.0 / 145.0.0 / 146.9.0 / 147.1.0 all released
- ✅ `sync.yml` enabled (daily cron 08:00 UTC checks upstream for new versions and triggers build.yml)

---

## 7. Lessons learned (what was tried → what was learned)

| What we did wrong | Lesson learned |
|-------------------|----------------|
| Cancelled a still-running Build V8 job without reasoning | Guessing failure doesn't count; only reasoned proof of certain failure justifies cancel |
| `|| true` swallowing Build V8 error codes | CI failures must be visible; no fallback swallows them |
| src-cache key with `runner.os` dimension | Source is OS-agnostic; adding this dimension only fragments the cache |
| Believed `!path` exclusion without verifying | Verify first, do not trust advertising |
| `run_id` suffix key with forced overwrite | Causes LRU eviction of other caches; API DELETE is sufficient |
| Pushed 4 versions in parallel | macOS runners queue, total time is longer. Cap at 2 |
| Half-built cache locked out by `cache-hit` logic | Do not gate save with `if: cache-hit != 'true'`; force refresh |
| Tweaked env var config for the same bindgen error | First figure out which bindgen (V8 internal / top-level) failed, then target the fix |
