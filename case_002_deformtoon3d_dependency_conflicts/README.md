# Case 002 — DeformToon3D Dependency Conflict Resolution

## Original Issue

GitHub Issue:
https://github.com/junzhezhang/DeformToon3D/issues/2

The project environment failed to build successfully due to multiple dependency and compatibility errors during Conda environment creation.

Errors included:

```text
dataclasses==0.8
detectron2==0.6
human-det==0.0.2
torchsearchsorted==1.1
unknown==0.0.0
```

along with multiple PyTorch dependency conflicts and native package build failures.

---

# Problem Summary

The repository contained an outdated and partially broken dependency stack inside its `environment.yml` configuration.

Several dependencies were:

- deprecated
- unavailable
- incompatible with modern Python versions
- unnecessary for the actual execution path
- or internally conflicting

The environment could not be recreated successfully without manual dependency investigation and cleanup.

---

# Root Cause Analysis

The main issue was not the core project itself, but rather long-term dependency drift and environment rot inside the project's Conda configuration.

The `environment.yml` attempted to recreate an older research environment using pinned package versions that no longer aligned with:

- modern Python versions
- modern CUDA versions
- current PyTorch ecosystems
- available package repositories

Specific failures included:

## Deprecated Python Package

```text
dataclasses==0.8
```

`dataclasses` was intended for older Python versions and became unnecessary because Python 3.7+ already includes built-in dataclass support.

---

## Missing or Removed Packages

```text
human-det==0.0.2
torchsearchsorted==1.1
unknown==0.0.0
```

These packages were unavailable or invalid.

---

## Dependency Conflicts

```text
torch==1.7.0
fairscale 0.4.6 depends on torch>=1.8.0
```

The environment attempted to install mutually incompatible package versions.

---

## Native Build Failures

`pycocotools` initially failed to build due to missing source/build compatibility issues.

---

# Reproduction Steps

## STEP 1 — Attempt Environment Creation

```bash
conda env create -f environment.yml
```

This triggered multiple package resolution failures.

---

## STEP 2 — Identify Broken Dependencies

Search the repository for problematic dependencies:

```bash
grep -R "detectron2\|human_det\|human-det\|torchsearchsorted" -n .
```

---

## STEP 3 — Remove or Bypass Invalid Dependencies

The environment file was progressively cleaned by:

- removing deprecated packages
- removing unavailable packages
- resolving incompatible version pins
- avoiding unnecessary dependencies

---

## STEP 4 — Rebuild the Environment

A simplified and compatible environment was rebuilt manually using:

- Python 3.7
- PyTorch 1.11
- torchvision 0.12
- pytorch3d
- pycocotools

Verification steps:

```bash
python --version

python -c "import torch; print(torch.__version__); print(torch.cuda.is_available())"

python -c "import torchvision; print(torchvision.__version__)"

python -c "import pytorch3d; print('pytorch3d ok')"

python -c "import pycocotools; print('pycocotools ok')"
```

---

# Proposed Fixes

## 1. Remove Deprecated Dependencies

Avoid legacy packages that modern Python versions already include.

Example:

```text
dataclasses==0.8
```

was unnecessary under Python 3.7+.

---

## 2. Remove Invalid or Missing Packages

Some packages listed in the environment configuration no longer existed or were not required for the actual execution path.

---

## 3. Resolve Version Compatibility

PyTorch ecosystem packages must remain version-aligned:

- torch
- torchvision
- fairscale
- detectron2
- pytorch3d

---

## 4. Rebuild Minimal Working Environments

Instead of blindly installing every dependency, isolate:

- what the project truly requires
- what is optional
- what is legacy
- what is no longer maintained

---

# Technical Findings

Additional observations during investigation:

- Research repositories frequently suffer from dependency drift over time
- Old Conda environments often become partially unreproducible
- Package pinning can become dangerous when ecosystems evolve
- Native build tools are especially sensitive to compiler/Python/CUDA version mismatches

Important realization during debugging:

The actual project logic was not necessarily broken.

The surrounding environment and dependency ecosystem had decayed over time.

---

# Lessons Learned

## Technical Lessons

- Environment configuration files are not permanent guarantees of reproducibility
- Old ML/CV repositories often contain obsolete dependencies
- Modern debugging frequently involves identifying unnecessary packages
- Dependency conflicts can completely block otherwise functional projects

---

## New Concepts Learned

- Conda environments
- `environment.yml`
- Dependency drift
- Environment rot
- Package version pinning
- PyTorch ecosystem compatibility
- Native package compilation
- CUDA/PyTorch dependency alignment
- Research repository reproducibility

---

# Key Takeaway

Many machine learning failures are not caused by broken models or algorithms, but by aging software ecosystems surrounding the project.

Modern AI engineering often requires identifying:

- what is truly required
- what is outdated
- what is optional
- and what no longer belongs in the environment at all

Successful debugging frequently comes from simplifying systems rather than blindly preserving every dependency.