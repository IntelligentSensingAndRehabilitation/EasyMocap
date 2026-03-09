# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

EasyMocap is an ISR fork of [zju3dv/EasyMocap](https://github.com/zju3dv/EasyMocap) — a toolbox for markerless human motion capture from RGB videos. It fits SMPL/SMPL-X body models to 2D/3D keypoints from multiple camera views.

## How ISR Uses EasyMocap

MultiCameraTracking is the primary consumer. EasyMocap is imported conditionally (inside functions) to avoid hard dependencies. The two core uses are:

### 1. Multi-view Multi-person Tracking (MVMP)

`multi_camera/analysis/easymocap.py` → `mvmp_association_and_tracking()` converts 2D detections from PosePipeline into associated 3D tracks across views and time.

**EasyMocap modules used:**
- `easymocap.config.mvmp1f.Config` — MVMP config loading
- `easymocap.assignment.group.Person`, `PeopleGroup` — person grouping
- `easymocap.assignment.associate.simple_associate` — cross-view association
- `easymocap.affinity.affinity.ComposedAffinity` — ray-based affinity
- `easymocap.assignment.track.Track3D` — temporal tracking

MultiCameraTracking ships 5 fallback YAML configs (`mvmp1f_default.yml` through `mvmp1f_fallback4.yml`) that tune association thresholds (min_views, max_repro_error, torso constraints). On SVD convergence failures, it retries with the next config.

### 2. SMPL Body Model Fitting

`multi_camera/analysis/easymocap.py` → `easymocap_fit_smpl_3d()` fits SMPL/SMPLx to 3D keypoints.

**EasyMocap modules used:**
- `easymocap.smplmodel.body_param.load_model` — load SMPL/SMPLx model
- `easymocap.pipeline.smpl_from_keypoints3d` — core fitting function
- `easymocap.pipeline.weight.load_weight_pose`, `load_weight_shape` — fitting weights
- `easymocap.smplmodel.body_model.SMPLlayer` — forward model for mesh vertices
- `easymocap.visualize.renderer.Renderer` — rendering fitted meshes
- `easymocap.dataset.CONFIG` — skeleton/keypoint definitions (body25, HALPE)

### DataJoint Integration

Three computed tables drive the pipeline:
- **`EasymocapTracking`** (depends on `CalibratedRecording`) → calls `mvmp_association_and_tracking()`
- **`EasymocapSmpl`** (depends on `EasymocapTracking`) → calls `fit_multiple_smpl()`
- **`SMPLReconstruction` / `SMPLXReconstruction`** → calls `easymocap_fit_smpl_3d()` on top-down keypoints

### Bridge: MCTDataset

`multi_camera/datajoint/easymocap.py` defines `MCTDataset(MVBase)` which adapts DataJoint data to EasyMocap's dataset interface (`easymocap.dataset.base.MVBase`). It fetches keypoints from `BottomUpPeople`, builds camera matrices, and handles unit conversion (divides `tvec` by 1000).

### Model Paths

- SMPL: `model_data/smpl_clean/` (override: `SMPL_CLEAN_PATH` env var)
- SMPLx: `model_data/smplx/` (override: `SMPLX_PATH` env var)

## Build & Install

```bash
# Installed as editable in the ISR Docker container
pip install --no-deps -e .

# OpenCV is optional to avoid conflicts with other ISR packages
# Install ONE variant if needed: pip install easymocap[opencv-headless]
```

Build uses hatchling (`pyproject.toml`). Legacy `setup.py` exists but `pyproject.toml` is authoritative.

## Architecture

### Config System (YACS-based)

Pipelines are driven by YAML configs in `config/`. Uses a custom YACS fork (`easymocap/config/yacs.py`) with `Config.load()` merging. Configs specify Python module paths dynamically loaded via `load_object()` (`easymocap/config/baseconfig.py`).

Two config types are always required:
- **Data config** (`config/data/`): dataset loader module and camera/video parameters
- **Experiment config** (`config/exp/`, `config/mv1p/`, `config/fit/`): processing pipeline steps (`at_step`, `at_final`)

### Key Modules

- **`easymocap/bodymodel/`** — SMPL/SMPL-X/MANO body model implementations with Linear Blend Skinning (`lbs.py`)
- **`easymocap/pyfitting/`** — Optimization routines for fitting body models to keypoints (L-BFGS, loss factories)
- **`easymocap/smplmodel/`** — SMPL parameter handling, model loading (`body_param.py`), forward model (`body_model.py`)
- **`easymocap/assignment/`** — Cross-view person association and temporal tracking
- **`easymocap/affinity/`** — Ray-based affinity computation for multi-view matching
- **`easymocap/pipeline/`** — High-level fitting pipelines (e.g., `smpl_from_keypoints3d`)
- **`easymocap/dataset/`** — Data loading for multi-view setups; `CONFIG` holds skeleton definitions
- **`easymocap/mytools/`** — Utility functions (`file_utils.get_bbox_from_pose`, etc.)
- **`easymocap/visualize/`** — Rendering utilities (`Renderer`)
- **`easymocap/socket/`** — Real-time visualization server/client (Open3D-based)
- **`myeasymocap/`** — Extended pipeline with backbone networks, dataset loaders, and composable stages

### CLI Entrypoint

`emc` (defined in `apps/mocap/run.py:main_entrypoint`) — not used by MultiCameraTracking (which calls the Python API directly), but useful for standalone testing:

```bash
emc --data config/data/mv1p.yml --exp config/mv1p/detect_triangulate_fitSMPL.yml --root /path/to/data
```

### Dependencies

Key runtime deps: `chumpy` (ISR fork), `mediapipe`, `pytorch-lightning==1.5.0`, `ultralytics` (YOLO). OpenCV is required but intentionally optional in `pyproject.toml` to avoid conflicts with other ISR packages.

### 3rdparty

Contains `pybind11` and `eigen-3.3.7` for C++ extensions (used in `src/easymocap/`).
