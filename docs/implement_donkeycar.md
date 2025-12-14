# Implementing Donkeycar Support for the AgileFlight Racing Stack

This document compares the existing multi-agent drone racing environment in this repository with the Donkeycar simulator and outlines a concise plan for bringing Donkeycar into the workflow.

## Current Drone Racing Setup (this repo)
- **Simulator / physics:** NVIDIA Isaac Lab + Isaac Sim, 500 Hz physics, 50 Hz policy.
- **Agents:** Two quadrotors (ego + adversary) with colored USD assets and contact sensors.
- **Tracks:** Procedural gates (complex or lemniscate) with optional walls; multi-lap scoring, gate-normal alignment, crash/time-out handling.
- **Observations:** Body-frame velocities, attitude matrix, two upcoming gate vertices, relative opponent pose/velocity, optional global position (with walls).
- **Actions:** 4D continuous (normalized collective thrust + body rates), motor/drag model, PID inner loop.
- **Rewards:** Gate pass + lap bonuses, rate penalties, crash penalties, leader-aware shaping.

## Donkeycar Simulator Quick Reference
- **Projects:** [donkeycar](https://github.com/autorope/donkeycar) platform and [gym-donkeycar](https://github.com/tawnkramer/gym-donkeycar) OpenAI Gym bridge.
- **Vehicle model:** 4-wheel ground vehicle, Unity-based simulator, typically single-agent.
- **Observations:** RGB front camera (default 120x160), optional state channels (speed, steering), Gym spaces.
- **Actions:** 2D continuous — steering in `[-1, 1]`, throttle/brake in `[0, 1]` (or signed throttle). Optional manual reset via simulator menu or reset command.
- **Control rate:** ~20–30 Hz by default; tunable via frame skip.

## Key Differences
| Aspect | Drone Racing (Isaac) | Donkeycar |
| --- | --- | --- |
| Platform | Isaac Lab + PhysX | Unity + Gym bridge |
| DOF | 6-DOF quadrotor, 3D gates | Ground vehicle, 2D track plane |
| Agents | Multi-agent (ego + adversary) | Typically single agent |
| Obs focus | Gate geometry, relative opponent pose, body velocities | Camera image (RGB), optional speed/steering |
| Actions | Thrust + body rates (4D) | Steering + throttle (2D) |
| Rewarding | Gate/lap bonuses, crashes, leader shaping | Progress/speed/track-centering (user-defined) |

## Implementation Plan for Donkeycar Integration
1. **Install Donkeycar + Gym Bridge**
   - Install system dependencies (Python 3.10 to 3.11 recommended/tested), pygame, and Unity sim build (from `gym-donkeycar` releases).
   - `pip install donkeycar gym-donkeycar` (or editable clones from the repos above).
   - Verify Gym launch: `python -m donkeycar.envs.donkey_proc --sim path/to/sim_executable`.

2. **Create an Environment Adapter in This Repo**
   - Add a new task directory (e.g., `src/isaac_quad_sim2real/tasks/donkeycar/`).
   - Implement a `DirectRLEnv`-style (Isaac Lab RL base class) wrapper; start with `gym_donkeycar.make("donkey-minimon-v0")` as the default but keep the environment ID configurable.
   - Allow configuration to pick alternate Donkeycar tracks/environments.
   - Map observation space: camera frames (optionally downsample/grayscale + stack), append speed/steering if enabled. Convert to torch tensors matching Isaac Lab rollout buffers.
   - Map action space: 2D `[-1, 1]` steering and throttle to Donkeycar’s expected tuple; include optional brake channel if enabled.
   - Implement reset/close hooks that respect the Donkeycar subprocess lifecycle (start sim once, reuse across envs or vectorize via Gym wrappers).

3. **Define Rewards and Terminations**
   - Start simple: reward = forward progress along centerline + speed bonus - off-track penalty - collision penalty (use Donkeycar’s `done` flags).
   - Add lap-completion bonus analogous to gate bonuses; log episode stats with existing `extras` pattern.

4. **Training/Evaluation Wiring**
   - Provide a config entry so `scripts/skrl/ma_train_race.py` (or a single-agent variant) can select the Donkeycar task via `--task Donkeycar-Race-v0`.
   - If only single-agent is needed, mirror the MAPPO script but disable adversary branches; keep rollout/logging APIs consistent.
   - Expose camera normalization, frame stacking, and action noise through CLI flags for quick ablations.

5. **Runtime Tips**
   - Run the Unity simulator headless when possible (`--headless` flag in the Donkeycar launcher) to avoid rendering overhead on the GPU. Physics still uses GPU if available.
   - Align control frequency. Either set Donkeycar frame-skip to approximate the 50 Hz policy rate or adjust the RL loop to match the sim step time reported by Gym.
   - For reproducibility, pin simulator version and record the SHA-256 of the Unity build used.

6. **Validation Checklist**
   - Smoke test: reset + one random action rollout without crash.
   - Policy sanity check: train PPO for a few thousand steps and confirm non-degenerate steering.
   - Logging: verify episode rewards/timeouts appear under `extras["log"]` similarly to the drone task.

Following this plan keeps changes localized (new task module + config) while preserving the existing training pipeline and logging conventions from the drone racing stack.
