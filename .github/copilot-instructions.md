## Simulator Architecture
- `src/main.rs` is the Rust entrypoint: it parses CLI args, loads or initializes a `Scene`, builds `Props` and `DataSet`, then drives the `backend::Backend` loop.
- `src/backend.rs` is the bridge to CUDA: it calls FFI hooks (`advance`, `initialize`, `update_bvh`, …) exposed by the shared libs under `src/cpp`, manages BVHs, auto-saves, and writes performance logs.
- `build.rs` always runs `make` in `src/cpp`, compiling CUDA/C++ kernels into `libsimbackend_cuda.so` and `libsimplelog.so`; any Rust build (even `cargo check`) needs CUDA 12.x, `nvcc`, `clang`, and Eigen headers available.
- Geometry/topology live in `src/data.rs`, `src/mesh.rs`, and `src/bvh.rs`; they expect column-major `nalgebra::Matrix3xX<f32>` buffers exported by the Python frontend, so keep shapes consistent when extending.
- FFI structs in `src/data.rs` and the matching C++ headers under `src/cpp/**` must stay binary-compatible—change both sides together and regenerate docstrings via `CppRustDocStringParser` if you touch the `/*== push "log" ==*/` blocks.

## Frontend Workflow
- The public API is the `frontend` package (`frontend/__init__.py`), where `App.create()` wires together managers for assets, scenes, sessions, and plots using method chaining.
- Scene authoring happens in `frontend/_scene_.py`: `FixedScene.export()` writes session bundles (`info.toml`, `bin/*.bin`, per-element parameter bins) that the Rust binary consumes via `Scene::new` in `src/scene.rs`.
- Simulation control lives in `frontend/_session_.py`: `Session.start()` enforces one solver process via `Utils.busy()`, validates GPU availability with `nvidia-smi`, and spawns the Rust binary using the generated `command.sh` shell wrapper.
- Export paths default to `~/.local/share/ppf-cts/git-<branch>/<app>/<session>`; respect that layout when adding new assets so the Rust loader’s relative lookups continue to work.
- Dynamic parameter ramps are serialized through `ParamManager.dyn().time().change()` and later replayed by `Scene::update_param`; prefer that path instead of custom ad-hoc files.

## Building & Running
- Use `cargo build --release` (or `cargo run --release -- --path <session> --output <dir>`) once CUDA toolchains are installed; debug builds are rarely used because the kernels assume `-O3`.
- The Makefile expects `nvcc` at `/usr/local/cuda/bin/nvcc` and `clang++`; override with env vars (`NVCC=...`, `CXX=...`) before running Cargo if needed.
- Python users typically call `App.create(...).scene.build(...).session.create(...).start()`; the first run compiles the Rust/CUDA stack and caches the session under `session/` within the app root.
- CI/headless runs rely on Docker images (`ghcr.io/st-tech/ppf-contact-solver-compiled:latest`); when scripting locally, mirror those driver versions (>=520) to avoid the guardrails in `Session.start()`.
- Only one solver process should run per workspace—`Utils.terminate()` targets binaries named `ppf-contact`, so keep that executable name in sync if you rename artifacts.

## Data & Logs
- Simulation outputs land under `--output <dir>`: geometry frames (`vert_<frame>.bin`), compressed states (`state_<frame>.bin.gz`), and per-frame metrics in `data/*.out`.
- `Scene::export_param_summary` writes `param_summary.txt` beside `param.toml`; read it to inspect aggregated masses/areas/volumes before diving into raw bins.
- Log naming (`time_per_frame`, `frame_to_time`, `clock`, etc.) is consumed by the docs generator—when adding a log, follow the `/*== push "name" ==*/` comment convention so `frontend/_parse_.py` can scrape metadata.
- Static collision meshes are bundled via `static_vert.bin`, `static_tri.bin`, and parameter bins; `Scene::make_constraint` recomputes displacement offsets every step, so supply consistent displacement maps when extending exporters.
- When troubleshooting, check `session/<name>/stdout.log` and `error.log`; initialization success is gated by `initialize_finish.txt`, and completion by `finished.txt`.

## Coding Patterns & Conventions
- Indices exported from Python are stored as `uint64` bins but cast to `usize`/`u32` in Rust, so keep meshes <2^32 elements or update both sides for larger scenes.
- Per-element properties (`density`, `contact-gap`, etc.) are stored in parallel arrays; extend them by updating `frontend/_scene_.py` exporters, the enums in `Scene::make_props`, and the CUDA structs that consume them.
- BVH rebuilds happen asynchronously via an mpsc worker in `backend::run`; if you add new geometry types, push their buffers through that channel and update `BvhSet`.
- Constraint transitions (`smooth` easing) and pin pulls are handled in `Scene::make_constraint`; reuse those helpers rather than duplicating interpolation logic elsewhere.
- The solver assumes column-major ordering and single-precision GPU buffers; convert doubles in Python exports (see `read_f64_mat_as_f32`) instead of mixing precisions downstream.

## Supporting Utilities
- `warmup.py` bootstraps system packages, CUDA, a Python venv, and docs tooling; call `python warmup.py setup` (or subcommands like `docs-build`) on fresh machines.
- Sphinx docs under `docs/` are generated from runtime metadata—after altering parameters or logs, run `python warmup.py docs-build` to refresh HTML outputs.
- Eigenvalue tools live in `eigsys/`; the Rust crate under `eigsys/eig-rust` mirrors the C++ headers in `eigsys/eig-hpp`, so keep those in sync when experimenting with stiffness analysis.
- Large demo notebooks under `examples/` expect the frontend API; keep method signatures stable or update the tutorials and the GitHub Actions that run them ten times per suite.

## Further Resources
- Project README collects highlight videos, cloud deployment notes, and example galleries: skim `README.md` when adjusting UX-facing code.
- Python API reference and parameter tables are published on GitHub Pages: https://st-tech.github.io/ppf-contact-solver/frontend.html (global/object params, logs).
- Stress-test GitHub Actions (e.g., `run-all-once.yml`, individual example workflows) demonstrate expected runtime stability requirements.
- Prebuilt Docker image `ghcr.io/st-tech/ppf-contact-solver-compiled:latest` matches CI environments; README outlines launch commands for local smoke tests.
