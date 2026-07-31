# gromacs-protein-ligand-md-workflow

# Protein-Ligand Molecular Dynamics Workflow (GROMACS + CHARMM36)

This repository contains an end-to-end command-line workflow for setting up, running, and analyzing a **protein-ligand molecular dynamics (MD) simulation** using **GROMACS** under a Linux/WSL environment.

The pipeline utilizes the **CHARMM36 force field** for the protein and **CGenFF** for ligand topology generation, featuring a dodecahedron solvation box, two-stage equilibration (NVT & NPT), production MD, and comprehensive post-simulation trajectory analysis.

---

## 🛠️ Environment & Prerequisites

* **OS:** Linux / Windows Subsystem for Linux (WSL2 / Ubuntu)
* **MD Engine:** GROMACS
* **Force Field:** CHARMM36 (July 2022 release or equivalent)
* **Ligand Parametrization Tools:** Open Babel, CGenFF (`cgenff_charmm2gmx_py3_nx2.py`), Perl
* **Analysis & Visualization:** Grace (`xmgrace`), PyMOL / VMD

---

## 🔬 Simulation Pipeline

### 1. Environment Setup & File Isolation
Locate working directory and initialize GROMACS environment variables:
```bash
cd /mnt/c/Users/YourUsername/Desktop/MD
source /usr/local/gromacs/bin/GMXRC
