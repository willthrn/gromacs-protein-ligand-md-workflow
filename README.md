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
```
Bash
cd /mnt/c/Users/YourUsername/Desktop/MD
source /usr/local/gromacs/bin/GMXRC
```
### 2. Structure Preparation & Topology Generation
A. Protein Preparation
Separate ligand from protein, strip non-standard HETATM records, and construct CHARMM36 topology with SPC water model:
```
grep XXX protein.pdb > ligand.pdb
grep -v HETATM protein.pdb > protein_processed.pdb
gmx pdb2gmx -f protein_processed.pdb -o protein_processed.gro -water spc
```
B. Ligand Parametrization (CGenFF Workflow)
Convert ligand to .mol2, sort bond orders, and generate GROMACS-compatible CHARMM topology files (.str / .itp):
```
obabel ligand.pdb -O ligand.mol2 -h
perl sort_mol2_bonds.pl ligand.mol2 ligand_fix.mol2
python3 cgenff_charmm2gmx_py3_nx2.py XXX ligand_fix.mol2 ligand_fix.str charmm36-jul2022.ff
```
### 3. Complex Assembly, Solvation & Neutralization
A. Solvation BoxBuild the ligand coordinate file, construct a rhombic dodecahedron box with a minimum $1.0\text{ nm}$ solute-box distance, and solvate using SPC water:
```
gmx editconf -f xxx_ini.pdb -o ligand.gro
gmx editconf -f complex.gro -o complex_box.gro -bt dodecahedron -d 1.0
gmx solvate -cp complex_box.gro -cs spc216.gro -p topol.top -o complex_solv.gro
```
B. Ion Addition & System Neutralization
Assemble TPR topology for ion addition and neutralize system charge using genion:
```
gmx grompp -f ions.mdp -c complex_solv.gro -p topol.top -o ions.tpr
gmx genion -s ions.tpr -o complex_solv_ions.gro -p topol.top -pname NA -nname CLA -neutral
```
### 4. Energy Minimization (EM) & System Relaxation
Perform energy minimization using steep-descent relaxation on 16 threads:
```
Bash
gmx grompp -f em.mdp -c complex_solv_ions.gro -p topol.top -o em.tpr
gmx mdrun -v -deffnm em -nt 16
```
### 5. Two-Stage System Equilibration
A. NVT Equilibration (Constant Temperature)
Generate index groups, restrict ligand position (posre_ligand.itp), and perform thermal equilibration:
```
gmx make_ndx -f ligand.gro -o index_ligand.ndx
gmx genrestr -f ligand.gro -n index_ligand.ndx -o posre_ligand.itp
gmx make_ndx -f em.gro -o index.ndx 
gmx grompp -f nvt.mdp -c em.gro -r em.gro -p topol.top -n index.ndx -o nvt.tpr
gmx mdrun -v -deffnm nvt -nt 16
```
B. NPT Equilibration (Constant Pressure)
Equilibrate system pressure while maintaining temperature coupling:
```
gmx grompp -f npt.mdp -c nvt.gro -t nvt.cpt -r nvt.gro -p topol.top -n index.ndx -o npt.tpr
gmx mdrun -v -deffnm npt -nt 16
```
### 6. Production MD Simulation
Run production MD simulation with checkpointing enabled for job pausing/resumption:

# Initial Production Run
```
gmx grompp -f md.mdp -c npt.gro -t npt.cpt -r npt.gro -p topol.top -n index.ndx -o md.tpr
gmx mdrun -v -deffnm md -nt 16
```
# Resume Simulation from Checkpoint (if interrupted)
```
gmx mdrun -v -deffnm md -nt 16 -cpi md.cpt
```
### 7. Trajectory Post-Processing & Structural Analysis
A. Periodic Boundary Condition (PBC) Correction
Center complex and remove periodic jumping across boundaries:
```
gmx trjconv -s md.tpr -f md.xtc -o md_corrected.xtc -pbc mol -ur compact -center
```
B. Quantitative Structural Metrics
Extract trajectory metrics into Grace (.xvg) format for publication plotting:

Root-Mean-Square Deviation (RMSD): 
```
gmx rms -s md.tpr -f md_corrected.xtc -o xvg/rmsd.xvg
```
Root-Mean-Square Fluctuation (RMSF): 
```
gmx rmsf -s md.tpr -f md_corrected.xtc -o xvg/rmsf.xvg -res
```
Radius of Gyration ($R_g$): 
```
gmx gyrate -s npt.gro -f md_corrected.xtc -o xvg/gyration.xvg
```
Hydrogen Bond Dynamics: 
```
gmx hbond -s md.tpr -f md_corrected.xtc -num xvg/hbond.xvg
```
Downsampled Trajectory (100 Frames): 
```
gmx trjconv -f md.xtc -s md_corrected.tpr -o reduced.xtc -dt 1000
```

📊 Summary of Extracted Structural Analysis
Energy, temperature, pressure, and structural trajectory graphs can be visualized using XMGRACE:
```
xmgrace xvg/rmsd.xvg
xmgrace xvg/rmsf.xvg
```
