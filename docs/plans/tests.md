# Tests

Which tests to run, prepare and integrate before the [IDE port](ide-port.md) adds its engine and implementation code. Each test proves a principle or requirement, named by its ID. Proposed 2026-09-30; the project lead ticks what gets built (A00). The ticked rows are the ones proposed to land before the IDE port.

The IDE port adds about 2,200 lines to an engine of about 3,300, and about 2,650 lines of recipes. Most of that is code that runs every frame, plus a new thread (the terminal reader) and the first file writes (save). A test that exists before that code lands turns each step into a measured change (C03). It also keeps A02, A03, C01, CPP10 and V10 from eroding unnoticed while the code grows.

## Today

- **ctest** runs `vulpen` with no arguments, `wave` for 120 frames, and `triangle` for 120 frames, which needs a display. Each checks only the exit code.
- **Gates**: the include map, and nothing else.
- **Validation**: a validation message does not fail a run. No debug messenger is installed, so the layer prints in its own format.

## Baseline

Measured 2026-09-30 at `5994c2c`, in a copy of the repo, on an RTX 5070 Laptop and on lavapipe. Rows marked **fix** are findings to settle before building on them.

| Check | Result | IDs |
| --- | --- | --- |
| Sanitizers (ASan, UBSan, LSan) | Headless `wave` on lavapipe is clean. Every other leak report is inside a library: the Wayland decoration plugin's GTK, D-Bus, the Vulkan loader and the NVIDIA driver. No UB. | A03, CPP09 |
| Library bounds assertions | `wave` and `triangle` run clean with `-D_GLIBCXX_ASSERTIONS`. | A02 |
| Thread races (TSan) | Clean over 6 live swaps on lavapipe. It needs `setarch -R` on this kernel, and it crashes inside the NVIDIA driver. | A01, C13 |
| Soak | `wave`, 2 minutes unpaced: memory, threads and file descriptors flat. `triangle`, 1 minute: +128 KB once, then flat; GPU memory flat at 50 MiB. | A03 |
| Swap soak | 60 swaps: memory +312 KB, then flat, then −204 KB. The loaded module count stays constant and no validation message appears. | A03 |
| Determinism | Bit-identical run to run, and between debug and release, on NVIDIA. NVIDIA and lavapipe differ in 31 of 32 values, by up to 261 ULP, because each driver's `sin` has its own precision. | C01, GLSL02 |
| Synchronization validation | Removing the engine's compute barrier gave no report: the layer does not track accesses through buffer device addresses. | V10 |
| GPU-assisted validation | Caught an injected out-of-bounds read through a buffer address (30 reports). Clean on `wave` and `triangle`. | GLSL02 |
| Heap on the hot path | Our code allocates nothing per frame. The process allocates about 4 times a frame headless and about 23 in a window, all in libraries. | CPP10 |
| Mutable globals | The built objects hold none from our code. One comes from vendored VMA. | C13 |
| Fail loud | Six broken manifests and a truncated or empty `.spv` exit 1, each naming its cause. | A02 |
| Manifest fuzz | 300 mutations in 15 s under sanitizers: 272 refused with a named error, 28 ran, no problem. | A02, A03 |
| Structure | No `shared_ptr` or naked `new`/`delete`, no OS header outside the platform files. `main.cpp` has 19 lines, the longest file 730 (`Schedule.cpp`), and every `.spv` passes `spirv-val`. | RC03, RA01, A04, CPP12 |
| Working directory | Running from `/tmp` works. | RP02 |
| **fix**: duplicates | Accepted, with one of each pair ignored: a param given twice (even with different values), a connection endpoint twice, two connections of one name, `[manifest]` twice, `version` twice. | A02 |
| **fix**: a limit without a name | Past 1,024 nodes with a pass block, the load fails as `VkResult -1000069000` (`max_passes` in `Pipelines.cpp`). | A02 |
| **fix**: time in a float | `Wave.cpp` adds its speed to a `float` every frame. At 60 fps, the wave is 2.3% slow after 2 hours and 56% fast after 67, and it stops after 4.4 days. | A03 |
| **fix**: fused multiply-add | GCC fuses `a*b+c` in C++ at `-O2` on any CPU with FMA. x86-64 builds are safe today, since the build sets no `-march`. On ARM64, release would differ from debug and from x86. `-ffp-contract=off` prevents it. | C01, RC06, C10 |
| **fix**: signals | SIGINT and SIGTERM end vulpen at once; no destructor runs. | A03, RA04 |
| Scale | 1,000 nodes run a frame in 0.7 ms. A chain loads in 0.68 s at 1,000 nodes and 2.16 s at 2,000. Ordering grows faster than the graph, and every edit will rerun it. | C03 |

## How to read the tables

- **Add**: `[x]` builds the test before the IDE port, `[ ]` later or with a step of the IDE port. The lead changes the ticks.
- **Proves**: the principles and requirements the test checks.
- **Cost**: estimated new lines (Python, CMake, C++ or test data), and how long the test runs.
- **When**:
  - **build**: a gate, before anything compiles;
  - **ctest**: with every build's tests;
  - **nightly**: a slower preset or run;
  - **release**: before a release;
  - **step**: with a step of the IDE port's [order](ide-port.md#order).

## 1. Gates

Static checks. They run with the include map in `src/tools/gates/` and fail the build (A02).

| Add | ID | Gate | Proves | How | Cost | When |
| --- | --- | --- | --- | --- | --- | --- |
| [x] | T1 | No mutable globals | C13 | lists writable data (`.data`, `.bss`, not `.data.rel.ro`) in every object built from `src/`, with vendored names allowlisted | 40 lines, <1 s | build |
| [x] | T2 | Bootstrap size | A04 | `main.cpp` has at most 30 lines | 10 | build |
| [x] | T3 | Ownership | RC03 | no `std::shared_ptr`, no naked `new` or `delete`, with comments and strings stripped | 30 | build |
| [x] | T4 | OS APIs in the platform files | RA01 | OS headers and `_WIN32`, `__linux__` appear only in `baseclasses/Platform.*` | 30 | build |
| [x] | T5 | Vendored libraries unchanged | RA02 | a checked-in SHA-256 list of `src/external-libraries/`; replacing a library updates it in the same commit | 30 + the list | build |
| [x] | T6 | Floating-point flags | RC06, C01, C10 | the compile commands carry no `-ffast-math`, `-Ofast` or `-funsafe-math-optimizations`, and do carry `-ffp-contract=off` | 20 | build |
| [x] | T7 | File length | CPP12 | a file over 800 lines says why at its top | 20 | build |
| [x] | T8 | Valid SPIR-V | RV02 | `recipe.cmake` runs `spirv-val` on each shader it compiles | 5 | build |
| [x] | T9 | Every ID has a test | A00, A02 | each principle and requirement is named by a test on this page, or marked review-only | 30 | build |

## 2. ctest

Fast tests, with every build: about 20 seconds more in all.

| Add | ID | Test | Proves | How | Cost | When |
| --- | --- | --- | --- | --- | --- | --- |
| [x] | T10 | Validation fails the test | RVK00 | every GPU test fails on a validation message (`FAIL_REGULAR_EXPRESSION`) | 5 | ctest |
| [x] | T11 | Title and quiet default | RC07, C09, V09 | the title line names the commit. At the default level, only the header, the footer and the lines of nodes that set their own level appear. | 15 | ctest |
| [x] | T12 | Same input, same output | C01 | two runs, and debug against release, read back identical bits | 20, 3 s | ctest |
| [x] | T13 | Golden output | C01, GLSL02 | a test recipe prints `wave`'s readback in exact bits. It matches a checked-in golden bit for bit on the reference device (T34), and within a tolerance per view on other drivers. | 60 + goldens, 2 s | ctest |
| [x] | T14 | Fail loud | A02, RV03 | broken manifests and shaders each expect exit 1 and a message naming the cause. The cases: the six measured, the six duplicates, a truncated and an empty `.spv`, a C++ name or type the shader does not declare, mismatched connection elements, invocations that do not fill workgroups, a draw that writes a buffer, and 1,025 passes. | 150, 10 s | ctest |
| [x] | T15 | Working directory | RP02 | the tests also run from another directory | 5 | ctest |
| [x] | T16 | Runs change no file | RA04 | the view folders hash the same before and after the tests | 20 | ctest |
| [x] | T17 | Barriers | V10 | the engine's hazard rule, on made-up pass lists: read after write, write after read and write after write each get a barrier, and read after read gets none. Synchronization validation cannot see buffer addresses, so nothing else checks V10 for buffers. | 80, <1 s | ctest |
| [x] | T18 | Module tests | A04, A02 | plain test programs, with no framework (C00, RA02), for the manifest, graph order, reflection on fixture SPIR-V, and the log | 250, <1 s | ctest |
| [x] | T19 | Display tests without a display | V07 | `triangle` is skipped, not failed, where no display exists | 5 | ctest |

## 3. Presets

Slower builds and runs: every night, and before engine work merges.

| Add | ID | Test | Proves | How | Cost | When |
| --- | --- | --- | --- | --- | --- | --- |
| [x] | T20 | Sanitizers | A03, CPP09, A02 | an `asan` preset (ASan, UBSan, LSan) runs every test on lavapipe. A windowed run takes a suppression file that names libraries, never our functions. | 30 | nightly |
| [x] | T21 | Thread races | A01, C13 | a `tsan` preset on lavapipe, under `setarch -R`, with live swaps | 20 | nightly |
| [x] | T22 | GPU-assisted validation | GLSL02, RVK00 | a layer settings file turns on `validate_gpu_based`, which catches out-of-bounds reads through buffer addresses | 10 | nightly |
| [x] | T23 | Library bounds assertions | A02 | `-D_GLIBCXX_ASSERTIONS` in the debug preset, so standard containers check their bounds | 1 | build |
| [x] | T24 | Heap on the hot path | CPP10 | an allocation counter loaded ahead of vulpen fails the test when our code (`vulpen`, recipe modules) allocates during steady-state frames. Library allocations are reported, not failed. | 100 | nightly |
| [x] | T25 | Manifest fuzz | A02, A03 | mutated manifests under sanitizers, with a fixed seed (C01): 300 in the `asan` preset, 10,000 nightly. A crash, a hang, a sanitizer report, or a refusal that names no cause fails it. | 80, 15 s | nightly |
| [ ] | T26 | Coverage | A02 | a gcov build reports line coverage per module in `docs/statistics/coverage.md` (RD01). It reports at first, then only goes up. | 60 | nightly |

## 4. Soak and stress

| Add | ID | Test | Proves | How | Cost | When |
| --- | --- | --- | --- | --- | --- | --- |
| [x] | T27 | Soak | A03 | headless `wave` and windowed `triangle`, sampling memory, GPU memory, file descriptors and threads every 10 s. It fails on growth after warm-up and writes `docs/statistics/soak.md` (RD01). | 120 | 10 min nightly, 8 h release |
| [x] | T28 | Swap soak | A03 | 500 swaps, of a shader, a module and the manifest in turn. It fails unless memory levels off, the loaded modules stay the same, and no validation message appears. | 60 | nightly |
| [x] | T29 | Scale and limits | C03, A02 | load and frame time from 10 to 2,000 nodes, unconnected and chained. A view past a limit must fail with a named error. | 60 | nightly |
| [x] | T30 | Long-run numbers | A03, C01 | fast-forward the frame count to days and years: time stays exact and nothing drifts. It covers `wave` now and the frame block (IDE port, A1) later. | 40 | ctest |
| [x] | T31 | Signals | A03, RA04 | SIGINT and SIGTERM end the run cleanly, with the footer and exit code the lead chooses | 45 | ctest |
| [ ] | T32 | More faults | RC04, A02 | a module that fails to load, running out of GPU memory, a lost device | 80 | nightly |

## 5. Devices and platforms

| Add | ID | Test | Proves | How | Cost | When |
| --- | --- | --- | --- | --- | --- | --- |
| [x] | T33 | CI without a GPU | V07, RP01 | every headless test runs on lavapipe, so a machine with no GPU runs them | 10 | ctest |
| [x] | T34 | Reference device | C01, C10 | a pinned lavapipe (the Mesa build in `../vulpen-lavapipe`, or a fixed package) holds the goldens, so a driver update never changes them | 10 | ctest |
| [ ] | T35 | The Vulkan floor | RVK01, C10 | Vulkan 1.2 with only the named features, through the profiles layer (not installed) or a floor device | – | release |
| [ ] | T36 | ARM64 | C10, C01 | an ARM64 build compares its goldens with x86 | – | release |
| [ ] | T37 | Windows | RP00 | the tests on Windows | – | release |

## 6. With the IDE port's steps

Prepared now, added when their step lands.

| Add | ID | Test | Proves | Step |
| --- | --- | --- | --- | --- |
| [ ] | T38 | Replay: record a session, replay it headless, compare the saved manifest and the goldens | V08, C01 | 3 (B3) |
| [ ] | T39 | Command port: a command without usage, help or completion does not register; command fuzzing on T25's harness | RV04, A02 | 1 (B1) |
| [ ] | T40 | Save: killing vulpen during a save leaves the old file or the new one, and comments survive a save | RA04 | 4 (B4) |
| [ ] | T41 | The terminal reader thread under T21 | A01, C13 | 8 (B9) |
| [ ] | T42 | GUI tests headless: input logs drive them, and offscreen goldens check them. The goldens need A8 (later). | V06, V07 | 15, 22 (B6, A7) |
| [ ] | T43 | Windows and hosted views opened and closed in a soak | A03 | 5, 10 (B14, C3) |
| [ ] | T44 | Synchronization validation over images and attachments | V10, RVK00 | 14, 22 (A5, A7) |

## Cost

The ticked rows are about 1,400 lines of tests and tooling:

| Section | Lines |
| --- | --: |
| Gates | 215 |
| ctest | 610 |
| Presets | 240 |
| Soak and stress | 325 |
| Devices | 20 |

The **fix** rows add about 60 lines to the engine and one line to `Wave.cpp`.

## Order

1. Fix the findings, each a small change: duplicates, the named limit, `-ffp-contract=off`, time in `Wave.cpp`, signals.
2. Add the gates (T1–T9), and T10, T15, T16 and T19. They are cheap and catch a regression at once.
3. Add the fail-loud corpus (T14), the goldens (T12, T13), the reference device (T34), and CI without a GPU (T33).
4. Add the presets (T20–T25) and the nightly soaks (T27–T29).
5. Add the barrier and module tests (T17, T18), which need small structure changes, and T30 and T31.
6. Start step 1 of the IDE port, with T39 beside it.

## Decisions (A00)

- **C01 and GLSL02 across drivers.** Either bit-exact per device and driver with a tolerance across drivers (what the baseline shows), or shaders without built-in transcendentals and with `precise`, so every GPU matches bit for bit. The second costs what a shader can use.
- **Signals.** Whether SIGINT and SIGTERM end the run cleanly. It needs a signal handler in the platform files (RA01).
- **Structure for T17 and T18.** The barrier rule must be reachable from a test, as a free function beside `Engine`: a new include edge. Tests also need a home: `src/tests/` for C++ and test data, and `src/tools/` for Python, both in the include map.
- **Time budget.** How long ctest may take per build; this page assumes under a minute.
