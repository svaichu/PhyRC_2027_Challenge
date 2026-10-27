# Repo memory for future agents — PhyRC 2027 Challenge (Phase 1)

Upstream: `rirolab/PhyRC_2027_Challenge` (this checkout: `svaichu/PhyRC_2027_Challenge`).
Challenge site: emprise.cs.cornell.edu/phyrc/challenge-2027 · Eval platform: Codabench 18366.
License: Apache-2.0 for project code; assets/DexGarmentLab-derived files are NOT covered (see `THIRD_PARTY.md`, `NOTICE`).

## What this is
Participant-facing package for a robot-learning competition: **two Stretch4 mobile manipulators dress a seated manikin in a T-shirt** in **NVIDIA Isaac Sim 6.0.1** (Docker). Participants teleop with the keyboard to collect demos, train a policy on RGB-D + proprioception, and submit a `policy.py` adapter (+ checkpoint) that is evaluated on hidden seeds. Most of the repo is simulator env code, a dataset/eval pipeline, and docs. There is **no test suite and no CI**; the code only runs inside the Isaac container (needs NVIDIA GPU) — in a cloud sandbox you can only read/edit, and lightly lint pure-Python bits.

## Layout
| Path | Role |
|---|---|
| `run.sh` | Single host entrypoint. Subcommands: `doctor`, `build`, `prepare`, `gui [flags]`, `replay <rec>`, `smoke`, `python <script>`, `cpu <script>`. Builds the `docker run` args, stages `src/` + assets into `.runtime/`, mounts `scripts/`→`/scripts:ro`, repo→`/project:ro`, `output/`→`/output`. Only forwards env vars named `STRETCH4_*|HUMAN_*|BOX_*|ROBOT*_*|GARMENT_X_OFFSETS`. Holds a `flock` on `cache/runtime.lock` (one run at a time). |
| `Dockerfile`, `entrypoint.sh`, `requirements-runtime.txt`, `sitecustomize/` | Image = pinned Isaac Sim 6.0.1 + termcolor/scipy/matplotlib/gymnasium/h5py. Entrypoint `cd /workspace/PhyRC_Sim` then `/isaac-sim/python.sh "$@"`. `sitecustomize.py` lazily enables the `omni.physx.demos` Kit extension so a dead `import omni.physxdemos` resolves. |
| `config/policy_interface.json` | **The contract** (schema 0.3.0): dims (2 robots, 13 joints, 5 cams, 256×256), timing, camera definitions/extrinsics, observation keys/shapes, action spec/scales. Loaded by `Policy/contract.py`. |
| `config/geometry.json`, `config/assets.lock.json` | Geometry params for scoring; sha256-pinned asset manifest (custom assets local; others downloaded from DexGarmentLab on HF/GitHub). |
| `assets/custom/` | Robot (`stretch_4.usd`), manikin (`female2_c4-c5.usd` + texture), T-shirt USD. Large binaries tracked in git. `assets/downloads/` is gitignored (fetched by `prepare`). |
| `src/PhyRC_Sim/Env_StandAlone/` | `BaseEnv.py`, and the 5.6k-line monolith `Teleop_TShirt_Stretch4_Env.py` (scene build, keyboard teleop, recording/eval hooks; configured by many `STRETCH4_*` env vars). |
| `src/PhyRC_Sim/Env_Config/` | `Garment/` (particle cloth, grasp, friction, contact guard, random spawn, mesh refine), `Human/` (manikin, hand colliders, round head, spawn), `Utils_Project/`, `Randomization.py`. |
| `src/PhyRC_Sim/Policy/` | Policy-facing stack: `environment.py` (`DressingEnv`, gymnasium; one per process; profiles `actor_rgbd`/`measured_state`), `contract.py` (validate obs, decode actions, gripper intent), `sensors.py` (RGB-D cams), `state.py` (state snapshot), `dataset_reader.py` (`PolicyDataset`, the allowed-only HDF5 loader), `training_*.py` (teleop→HDF5 writers; deferred vs live render), `teleop_recording.py`/`teleop_playback.py` (raw state archive, replay), `evaluation*.py` (scorer `Phase1Scorer`, contact/geometry/clock/live/replay/quality), `stages.py` (scene slots). |
| `scripts/` | Container-side entry scripts: `evaluate_policy.py` (run an adapter over seeds → `scores.json`), `evaluate_replay.py`, `evaluate_phase1.py`, `export_policy_dataset.py` (deferred RGB-D export after ESC), `inspect_policy_dataset.py`, `check_deferred_dataset.py`, `replay_teleop.py`, `smoke.py`, `fetch_assets.py`/`prepare_assets.py`/`remote_zip.py` (asset setup), shirt authoring tools (`make_wearable_shirt.py`, `resize_shirt.py`, `straighten_shirt_sleeves.py`), `export_replay_contact/quality.py`. |
| `examples/zero_policy.py` | Minimal adapter (zeros) for interface checks. |
| `docs/competition/` | `SETUP, DATA, POLICY, EVALUATION, RULES, GLOSSARY, VERIFICATION` (authoritative human docs). `docs/examples/teleop/` = sample 1,142-step recording as CSV/JSON + viewer; `docs/videos/` = demo media (mp4 un-ignored only here). |

## Key facts (verify against `config/policy_interface.json` / `docs/` before relying)
- Rates: policy **20 Hz**, controller 60 Hz, physics 240 Hz; each action held 0.05 s = 3 control ticks = 12 physics steps. All timing/scoring is **simulation time**.
- Obs per step: RGB `(5,256,256,3)` uint8, depth/valid `(5,256,256,1)`, joint pos/vel `(2,13)`, base pose `(2,7)` (xyz+wxyz), base vel `(2,6)`, gripper pose `(2,7)` in base frame, fingertips, controller target `(2,9)`, gripper close cmd, previous action `(2,9)`, sim time. Camera order: overview, r0 wrist, r1 wrist, r0 head, r1 head.
- Action `(2,9)` float32 in [-1,1]; per robot: fwd, left, base_yaw, lift, arm, wrist_yaw, pitch, roll, gripper (+1 close, −1 open, **0 = keep previous intent**). HDF5 stores flat `(18,)` = robot_0 then robot_1.
- Dataset: `output/policy_datasets/<run-id>/policy.hdf5` (robomimic-style `data/demo_N/{obs,next_obs,actions,rewards,dones}`; rewards are placeholder 0) plus `audit.hdf5` (**never a policy input**), `capture.json`, `evaluation/`. Raw log in `output/full_teleop/<run-id>/`. File marked `complete=false` if the run is killed.
- Scoring: 50 pts (pickup 5, first sleeve 5, opposite shoulder 5, second sleeve 5, overall: L/R upper arm 5+5, neck 10, V-neck front 10). Metric `points/s = (total − pickup pts) / (last award time − first shirt–manikin contact)`; 0 if no contact. Phase-1 rules also gate "success".
- Submission: zip with top-level `policy.py` defining `load_policy(checkpoint, observation_space, action_space)` → callable with `reset()` and `__call__(obs)->(2,9)`; optional `checkpoint.*`, optional `Dockerfile` (`FROM phyrc-2027:isaac-6.0.1`, only add packages, don't touch torch). 1 RTX 3090, ≤12 GB policy GPU, 900 s episode limit incl. inference time, 3 hidden seeds final; leaderboard checkpoints Nov 17, Dec 1/4/8/12.
- Rules (`docs/competition/RULES.md`): only public obs + (2,9) action; no cloth particles, exact shirt/manikin/chair poses, collision meshes, grasp/eval internals as inputs, labels, or teacher policies; no replaying state trajectories; no changing physics/limits/eval. Training-time resets/rewards/terminations may use internal state if no model sees it. Pretrained public models and outside data are OK.

## Typical commands (host with Docker + NVIDIA toolkit)
```
./run.sh doctor && ./run.sh build && ./run.sh prepare && ./run.sh smoke
./run.sh gui [--training-record 1] [--evaluate 1] [--training-render live] [--no-randomization] [--full-record 1]
./run.sh python /scripts/evaluate_policy.py --policy /project/examples/zero_policy.py --seeds 42 43 --seconds 1 --output /output/evaluation/adapter_check
```
Teleop keys are in README §2 (robot_0 on letters, robot_1 on arrows/numpad; `P` reset, `ESC` save+quit — wait for `[Dataset export] READY`).

## Conventions & gotchas for editing
- Path rule: env files resolve assets from CWD = `PhyRC_Sim` root, with the robot USD at sibling `../RCareWorld-2.0/assets/robots/...`; `run.sh prepare` copies `src/` into `.runtime/` — **edit `src/`, then re-run `prepare`** (or the container sees stale code).
- `scripts/` and `/project` are mounted read-only at run time; outputs go to `/output` (host `output/`, gitignored with `cache/`, `.runtime/`, `*.hdf5`, `*.npz`, `*.mp4` except `docs/videos/`).
- One Isaac app per process; `DressingEnv` raises if instantiated twice. Multi-seed eval needs randomization on.
- Keep policy-facing code free of privileged state (see RULES); `audit.hdf5`/evaluation JSON are for verification only.
- Don't commit big recordings. Doc text is the source of truth for rules/deadlines; if a doc and code disagree, flag it rather than guessing.
- Recent history is mostly rules/docs clarifications (training rewards/resets, pretrained models, submission format).
