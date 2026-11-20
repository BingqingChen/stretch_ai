# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Stretch AI is a Python package (`stretch_ai`, importable as `stretch`) for grasping, mapping, navigation, LLM agents and speech on the Hello Robot Stretch (RE1/RE2/SE3; Stretch 4 lives in a separate repo, `stretch4_ai`). It runs as two halves:

- **Robot side**: a ROS 2 package (`src/stretch_ros2_bridge`), launched via `./scripts/run_stretch_ai_ros2_bridge_server.sh` (usually in Docker). It exposes robot state, cameras and commands over ZMQ.
- **GPU computer side**: the `stretch` package (`src/stretch`), which runs perception models, mapping, planning and LLM agents, and talks to the robot over ZMQ (ports 4401–4404). Set the robot IP once with `./scripts/set_robot_ip.sh <ip>`.

Python 3.10, Ubuntu 22.04, NVIDIA GPU. Heavy ML dependencies (torch, etc.) come from `install.sh` (conda/mamba env) and are not in `setup.py`.

## Commands

```bash
# Dev install (from src/)
cd src && pip install -e .[dev] && pre-commit install

# Lint/format everything (black, isort, flake8, mypy, codespell, license headers, nbstripout, beautysh)
pre-commit run --all-files

# Tests (run from src/)
cd src && pytest
pytest test/core/test_task.py                 # single file
pytest test/motion/test_ik.py -k <name>       # single test
```

Apps are run as modules, e.g. `python -m stretch.app.ai_pickup [--use_llm --use_voice]`, `python -m stretch.app.view_images`, `python -m stretch.app.mapping`. See `docs/apps.md`.

**Simulation (no robot needed)**: start `python -m stretch.simulation.mujoco_server` (add `--use-robocasa` for Robocasa), then run an app against it with `--robot_ip 127.0.0.1 --parameter_file=sim_planner.yaml`. Requires the `stretch_mujoco` install from `docs/simulation.md`.

## Style rules enforced by pre-commit

- black, line length 100; isort with black profile (`stretch` first-party).
- flake8 checks only a narrow set: syntax errors, undefined names, **unused imports (F401)** — unused imports fail outside `__init__.py`.
- mypy runs only on some packages; `src/stretch/app`, `src/stretch/agent`, `src/stretch/perception`, `src/stretch_ros2_bridge` and `third_party/` are excluded (see `.pre-commit-config.yaml`).
- Every source file needs the license header from `docs/license_header.txt`; the `insert-license` hook adds it automatically.
- CONTRIBUTING asks for type hints and docstrings. PRs are squash-merged.

## Architecture

**Robot client abstraction**: `stretch/core/robot.py` defines `AbstractRobotClient` (move_base_to, switch_to_navigation/manipulation mode, arm control, get_observation…). The main implementation is `HomeRobotZmqClient` in `stretch/agent/zmq_client.py`, which talks to either the ROS 2 bridge server (`stretch_ros2_bridge/remote/server.py`) or the MuJoCo sim server (`stretch/simulation/mujoco_server.py`); both speak the same ZMQ protocol. Shared data types (`Observations`, actions) are in `stretch/core/interfaces.py`. Messages are pickled via pyzmq `recv_pyobj`; the README flags this as a known remote-execution risk on port 4402.

**Parameters**: Apps load a YAML from `stretch/config/` with `get_parameters(...)` (`stretch/core/parameters.py`). `default_planner.yaml` is the real-robot default; `sim_planner.yaml` and `dynav_config.yaml` are alternatives. Config paths resolve against `stretch/config` (`stretch/utils/config.py`).

**Agent**: `RobotAgent` (`stretch/agent/robot_agent.py`) owns the robot client, a semantic sensor (perception wrapper), and the voxel map / instance memory. `robot_agent_dynamem.py` is the DynaMem variant for dynamic scenes.

**Task / Operation framework**: Behaviors are `Task`s made of `Operation`s (`stretch/core/task.py`). Concrete operations subclass `ManagedOperation` (`stretch/agent/base/managed_operation.py`) and live in `stretch/agent/operations/` (navigate, update, grasp_object, place_object, speak, …). Tasks under `stretch/agent/task/` (`pickup/`, `dynamem/`, `emote/`, `llm_plan/`) chain operations with `task.add_operation(...)` and failure edges via `connect_on_failure`.

**LLM path**: `stretch/llms/` holds client wrappers (OpenAI, Gemini, Gemma, Qwen, Llama) and prompts in `stretch/llms/prompts/`. The LLM emits a list of task tuples that an executor (e.g. `stretch/agent/task/pickup/pickup_executor.py`) dispatches to tasks. Adding an LLM-callable skill means: new task (+ new operations if needed), a standalone test app in `stretch/app/`, a branch in the executor, and an updated prompt. `docs/adding_a_new_task.md` walks through this with the handover task.

**Perception & mapping**: `stretch/perception/` contains detectors (OWL, SAM2, YOLOE), encoders (CLIP, SigLIP, MobileCLIP) and captioners, wrapped by `perception/wrapper.py`. `stretch/mapping/voxel/` builds the 3D voxel map and planners; `mapping/instance/` tracks object instances (cuboids + features) that operations query by name or embedding. `stretch/motion/` handles kinematics/IK (pinocchio) and motion planning algorithms. Visualization uses Rerun (`stretch/visualization/rerun.py`).

**Other**: `stretch/app/lfd` and `dex_teleop` cover learning from demonstration (uses a hello-robot fork of LeRobot). `stretch/utils/` includes vendored pieces of pytorch3d / torch_scatter / torch_cluster / torch_geometric. `third_party/` holds git submodules (segment-anything-2, ok-robot), so clone with `--recursive`.
