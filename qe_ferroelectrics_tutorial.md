# Getting Started: First-Principles DFT Calculations for Ferroelectrics with Quantum ESPRESSO on CHPC

*A practical onboarding tutorial for undergraduate researchers*

---

## How to use this tutorial

Work through this document linearly the first time — each section assumes the previous ones. Don't just read; **type the commands yourself**. After you finish, keep this as a reference you come back to. Expect the first pass to take 1–2 weeks of part-time work if you're brand new to Linux.

When something breaks (and it will), that's normal. Save the error, try the troubleshooting section, then ask. Learning to read error messages is half of computational research.

---

## Part 0 — The Big Picture (read this first)

### What we are actually doing

We are computing the properties of real materials by solving the equations of quantum mechanics for their electrons. Specifically, we use **Density Functional Theory (DFT)** — a framework in which the hard many-electron Schrödinger equation is reformulated in terms of the electron density, which is tractable on a computer. The software we use, **Quantum ESPRESSO (QE)**, is an open-source implementation of plane-wave DFT with pseudopotentials.

### Why ferroelectrics?

Ferroelectrics are materials with a spontaneous electric polarization that can be switched by an external field. Classic examples are **BaTiO₃**, **PbTiO₃**, and **LiNbO₃**. The physics is structural: at high temperature the material is in a high-symmetry *paraelectric* phase (cubic for the perovskites), and as it cools, the atoms shift slightly off their symmetric positions, breaking inversion symmetry and producing a net dipole. DFT is excellent for capturing this — we can compute:

- The ground-state atomic structure of each phase
- The energy difference between paraelectric and ferroelectric phases (the "double-well" in energy vs. distortion)
- Born effective charges (how much polarization is produced per unit displacement)
- The spontaneous polarization itself (via the modern theory of polarization / Berry phases)
- Phonons and soft modes
- Response to strain, electric field, etc.

You will start with the simplest calculation — the **SCF (self-consistent field) calculation** — which gives the ground-state electron density and total energy for a fixed set of atomic positions. Almost every other calculation is built on top of SCF.

### The workflow you are learning

```
 Local laptop  --(ssh)-->  CHPC login node  --(sbatch)-->  compute node
     |                           |                              |
     |                     (edit files,                    (QE runs here,
     |                      submit jobs)                   writes output)
     |                           |                              |
     <-------- (scp/rsync, pull results back) ------------------+
```

You write input files and SLURM scripts on CHPC, submit them to the queue, SLURM schedules them onto a compute node, QE runs, and you analyze the output. This tutorial takes you through every piece of that pipeline.

### Tools you'll learn

1. **Unix/Linux** — the operating system on every supercomputer
2. **Bash** — the shell language you type commands in, and the language of automation scripts
3. **A text editor** (nano to start; graduate to vim or VS Code Remote-SSH later)
4. **Environment modules** (`module load`) — how HPC systems expose software
5. **SLURM** — the job scheduler that decides when and where your job runs
6. **Quantum ESPRESSO** — specifically `pw.x`, the plane-wave SCF engine
7. **Basic data wrangling** — grep, awk, simple Python/gnuplot for plotting

---

## Part 1 — Unix/Linux Essentials

### Getting a shell

If you have a Mac, open **Terminal**. On Windows, install **WSL2** (Windows Subsystem for Linux) with Ubuntu, or use **MobaXterm** / **PuTTY**. Practice these commands locally before touching CHPC.

### The filesystem is a tree

Everything is a file, and files live in a tree rooted at `/`. Your personal space is your **home directory**, abbreviated `~`.

### The 20 commands you must know

| Command | What it does | Example |
|---|---|---|
| `pwd` | print working directory | `pwd` |
| `ls` | list files | `ls -la` (long, all files) |
| `cd` | change directory | `cd ~/projects` |
| `mkdir` | make directory | `mkdir batio3_scf` |
| `rm` | remove file (**no trash can**) | `rm file.txt` |
| `rm -r` | remove directory recursively | `rm -r old_runs` |
| `cp` | copy | `cp a.in b.in` |
| `mv` | move/rename | `mv old new` |
| `cat` | print file contents | `cat scf.out` |
| `less` | scroll through file (`q` to quit) | `less scf.out` |
| `head` / `tail` | first/last lines | `tail -n 50 scf.out` |
| `grep` | search for pattern | `grep "total energy" scf.out` |
| `wc` | count lines/words | `wc -l file.txt` |
| `du -sh` | size of a directory | `du -sh ~/scratch` |
| `df -h` | disk usage on filesystems | `df -h` |
| `chmod +x` | make executable | `chmod +x run.sh` |
| `which` | where is this command? | `which pw.x` |
| `man` | manual page (`q` to quit) | `man grep` |
| `history` | recent commands | `history \| tail` |
| `Ctrl+C` | kill the current command | — |

### Shortcuts that make you 10× faster

- **Tab** — autocomplete filenames and commands. Use constantly.
- **Up/Down arrows** — scroll through command history.
- **Ctrl+R** — reverse-search history (start typing, it matches).
- **Ctrl+A / Ctrl+E** — jump to start / end of line.
- `*` — glob pattern, e.g., `ls *.out` lists every file ending in `.out`.
- `|` (pipe) — send output of one command into another: `cat scf.out | grep energy`.
- `>` and `>>` — redirect output to a file (overwrite / append): `ls > files.txt`.

### Text editing: nano (start here)

```bash
nano scf.in
```

Edit, then `Ctrl+O` to save, `Ctrl+X` to exit. That's it. Learn vim later when you're comfortable — the productivity gain is real but the learning curve is steep.

### Exercise 1

Create this directory structure and the files inside. This is exactly the kind of organization you'll use for real projects.

```
~/tutorial/
  part1_unix/
    hello.txt              <- contains your name
    notes.md               <- contains "DFT is density functional theory"
  part2_bash/
  part3_qe/
```

Verify with `ls -R ~/tutorial`. If you can do this without looking things up, you've got Part 1.

---

## Part 2 — Connecting to CHPC

### Accounts and access

The University of Utah's **Center for High Performance Computing (CHPC)** provides the clusters we use. Before anything else:

1. Your PI should have added you to their group/allocation. Confirm this with them.
2. Request a CHPC account at the CHPC website (chpc.utah.edu).
3. Read CHPC's own Getting Started page — it supersedes this tutorial for anything CHPC-specific, because their configuration (partition names, filesystem paths, module versions) changes over time.

### Logging in

From your laptop terminal:

```bash
ssh YourUNID@notchpeak.chpc.utah.edu
```

Replace `YourUNID` with your University of Utah ID. You'll be prompted for your campus password and likely a Duo 2FA push. After login you are on a **login node** — this is shared with every other user and is for editing files and submitting jobs, **not for running calculations**. Never run `pw.x` directly on a login node.

### Filesystems on CHPC (what goes where)

CHPC has several filesystems with different purposes. The exact paths evolve, so confirm against CHPC docs, but the categories are universal:

- **Home directory** — small quota, backed up. Scripts, input files, notes. Do *not* run jobs here; it's slow and small.
- **Group space** — medium quota, backed up. Shared PI resources — pseudopotentials, reference data.
- **Scratch** (e.g., `/scratch/general/vast/uXXXXXXX`) — large, fast, **not backed up, files may be purged after ~60 days**. This is where your QE runs happen. Copy what you want to keep back to home or group space.

Rule of thumb: **edit in home, run in scratch, archive to group space.**

### SSH keys (do this once, save yourself hours)

Set up an SSH key so you don't retype your password every time.

```bash
# On your laptop:
ssh-keygen -t ed25519          # press Enter to accept defaults; add a passphrase
ssh-copy-id YourUNID@notchpeak.chpc.utah.edu
```

### Transferring files

Small files — use `scp`:
```bash
scp local_file.txt YourUNID@notchpeak.chpc.utah.edu:~/tutorial/
scp YourUNID@notchpeak.chpc.utah.edu:~/tutorial/scf.out ./
```

Whole directories or big transfers — use `rsync` (resumes if interrupted):
```bash
rsync -avP local_dir/ YourUNID@notchpeak.chpc.utah.edu:~/remote_dir/
```

### Exercise 2

Log in, replicate the `~/tutorial/` structure from Exercise 1 on CHPC, and `scp` a text file from your laptop into `~/tutorial/part1_unix/` on CHPC. Confirm with `ls`.

---

## Part 3 — Bash Scripting

A bash script is just a text file containing commands, made executable. This is how you automate everything. Let's go from zero to a script that runs a parameter sweep.

### A minimal script

Create `hello.sh`:

```bash
#!/bin/bash
# The line above is the "shebang" — it tells the OS this is a bash script.
echo "Hello, $USER. Today is $(date)."
```

Make it executable and run it:
```bash
chmod +x hello.sh
./hello.sh
```

### Variables

```bash
#!/bin/bash
material="BaTiO3"
alat=4.00
echo "Running $material with lattice constant $alat Å"
```

Two rules people get wrong:
- **No spaces around `=`** when assigning (`x=3`, not `x = 3`).
- **Use `$var` to read, plain `var` to assign.**

### Arithmetic

Bash doesn't do floats natively. Use `bc` or `awk`:

```bash
new_alat=$(echo "4.00 * 1.02" | bc -l)
echo "$new_alat"
```

### Loops

This is where bash earns its keep. Say you want to run an SCF calculation at 5 different lattice constants:

```bash
#!/bin/bash
for a in 3.90 3.95 4.00 4.05 4.10; do
    echo "Preparing input for a = $a"
    mkdir -p run_a${a}
    # (we'll fill this in later with real QE input generation)
done
```

Nested loops for 2D scans (e.g., `a` and `c` for tetragonal systems):

```bash
for a in 3.95 4.00 4.05; do
    for c_over_a in 1.00 1.02 1.04; do
        c=$(echo "$a * $c_over_a" | bc -l)
        echo "a=$a c=$c"
    done
done
```

### Conditionals

```bash
if [ -f scf.out ]; then
    echo "Output exists."
else
    echo "Not run yet."
fi
```

### A realistic helper script: grab total energies from many runs

```bash
#!/bin/bash
# collect_energies.sh — scan every subdirectory and pull the final total energy
for dir in run_a*/; do
    energy=$(grep "!    total energy" "$dir"/scf.out | tail -n 1 | awk '{print $5}')
    echo "$dir $energy"
done
```

Run `./collect_energies.sh > energies.dat` to dump a two-column file ready for plotting.

### Exercise 3

Write a script that creates 5 directories named `test_01` through `test_05` and writes a file inside each named `info.txt` containing the directory's index. Verify it works.

---

## Part 4 — Environment Modules

HPC systems have dozens of compilers and hundreds of software packages at multiple versions. The **module system** lets you load what you need for this session without polluting your environment.

### The 5 commands

```bash
module avail                       # list everything available
module avail quantum               # filter
module load quantumespresso        # or the exact name/version CHPC uses
module list                        # what's loaded right now
module purge                       # unload everything
```

On CHPC the exact module name for QE varies by version (and which compiler/MPI stack it was built against). Check with `module spider quantumespresso` or `module avail qe`. Once you find it, note the full name — you'll use it in your SLURM scripts.

### Exercise 4

Log in to CHPC, run `module spider quantumespresso`, and write down the exact module name(s) available. Load one, run `which pw.x`, and confirm it returns a path.

---

## Part 5 — SLURM: Submitting Jobs

SLURM is the scheduler. You hand it a script that describes (a) what resources you want and (b) the commands to run. It finds a node, runs your job, and gives you the output.

### Anatomy of a SLURM submit script

Save this as `run_scf.slurm`:

```bash
#!/bin/bash
#SBATCH --job-name=scf_test
#SBATCH --account=your_pi_allocation      # ASK YOUR PI for this
#SBATCH --partition=notchpeak              # or kingspeak, lonepeak, etc.
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=32               # MPI ranks; match node core count
#SBATCH --time=01:00:00                    # HH:MM:SS; under-request -> faster queue
#SBATCH --mem=0                            # 0 = all memory on the node
#SBATCH --output=slurm-%j.out              # %j = job ID
#SBATCH --error=slurm-%j.err

# --- environment ---
module purge
module load quantumespresso/<version>      # use the exact name from Part 4

# --- run ---
cd $SLURM_SUBMIT_DIR
mpirun pw.x -in scf.in > scf.out
```

Replace `your_pi_allocation` and the module version with the real values your PI gives you.

### Submitting and monitoring

```bash
sbatch run_scf.slurm         # submit; prints a job ID
squeue -u $USER              # your jobs in the queue
scontrol show job <jobid>    # detailed info
scancel <jobid>              # kill a job
sacct -j <jobid>             # info after it has finished (runtime, state, etc.)
```

States you'll see: `PD` = pending, `R` = running, `CG` = completing, `CD` = completed, `F` = failed.

### How to choose resources

Start small. For the SCF calculations in this tutorial:

- **1 node, 16–32 MPI ranks, 1 hour** is more than enough for a 5-atom perovskite unit cell.
- Requesting more does *not* make things faster past a point — QE scales to a limit, and larger requests wait longer in the queue.
- When you scale up to bigger systems (supercells, relaxations, phonons), you'll increase nodes and time. Benchmark — don't guess.

### CHPC-specific notes

- CHPC uses partitions like `notchpeak`, `kingspeak`, `lonepeak`, plus their `-shared` and `-guest` variants. Guest partitions are free but preemptible. Confirm with your PI which allocation and partition you should use.
- Your account name (the `--account=` line) is tied to your PI's allocation. If you omit it or use the wrong one, your job fails immediately.
- Always put intensive I/O in scratch, not home. Set your working directory in scratch before submitting.

### Exercise 5

Write a SLURM script that prints the hostname and sleeps 30 seconds. Submit it, watch it with `squeue`, read the output file. You've now submitted your first HPC job.

---

## Part 6 — Quantum ESPRESSO: the SCF Calculation

QE is a suite. The one you'll use first is `pw.x` — plane-wave self-consistent DFT.

### What SCF does

Given atomic positions and a choice of exchange-correlation functional, `pw.x` iteratively solves the Kohn–Sham equations until the electron density is self-consistent. Output: the ground-state total energy, charge density, eigenvalues, and forces on atoms.

### Pseudopotentials — get these right or nothing works

In plane-wave DFT we don't treat core electrons explicitly. A **pseudopotential** file (`.UPF`) encodes how each element's valence electrons feel the nucleus + core. Rules:

1. Use the **same functional** (e.g., all PBE, or all LDA) across *every* element in a calculation. Mixing functionals silently gives garbage.
2. Use a well-tested library. Good starting points: **SSSP** (Standard Solid-State Pseudopotentials), **PseudoDojo**, or **GBRV**.
3. Each pseudopotential has a recommended **ecutwfc** and **ecutrho**. Respect them as minimums, then converge upward.

Download UPFs, put them in one directory (e.g., `~/pseudo/`), and point QE to it.

### Anatomy of a QE input file

QE input files have Fortran-namelist sections (`&CONTROL`, `&SYSTEM`, `&ELECTRONS`, ...) followed by card sections (`ATOMIC_SPECIES`, `ATOMIC_POSITIONS`, `K_POINTS`). Here is a minimal SCF input for cubic BaTiO₃ (the paraelectric phase, 5 atoms per cell):

```
&CONTROL
    calculation = 'scf'
    prefix      = 'batio3_cubic'
    outdir      = './tmp/'
    pseudo_dir  = '/uufs/chpc.utah.edu/common/home/uXXXXXXX/pseudo/'
    verbosity   = 'high'
/
&SYSTEM
    ibrav       = 1              ! simple cubic
    celldm(1)   = 7.559           ! a in Bohr (~4.00 Å). USE YOUR VALUE.
    nat         = 5
    ntyp        = 3
    ecutwfc     = 60              ! Ry; CONVERGE THIS
    ecutrho     = 480             ! Ry; usually 8× ecutwfc for USPP/PAW, 4× for NC
    occupations = 'fixed'         ! insulator
/
&ELECTRONS
    conv_thr    = 1.0d-8
    mixing_beta = 0.7
/
ATOMIC_SPECIES
  Ba  137.327  Ba.pbe-spn-kjpaw_psl.1.0.0.UPF
  Ti   47.867  Ti.pbe-spn-kjpaw_psl.1.0.0.UPF
  O    15.999  O.pbe-n-kjpaw_psl.1.0.0.UPF

ATOMIC_POSITIONS crystal
  Ba 0.0 0.0 0.0
  Ti 0.5 0.5 0.5
  O  0.5 0.5 0.0
  O  0.5 0.0 0.5
  O  0.0 0.5 0.5

K_POINTS automatic
  8 8 8 0 0 0
```

Replace the pseudopotential filenames with whatever you actually downloaded, and `pseudo_dir` with your real path. The units: `celldm(1)` is in **Bohr** by default (1 Bohr ≈ 0.529 Å). If you'd rather work in Å, use `A = 4.00` instead of `celldm(1)`.

### The tetragonal (ferroelectric) phase of BaTiO₃

The ferroelectric phase is tetragonal — the Ti and O sublattices shift relative to Ba along the c-axis, producing polarization. Experimental lattice constants are roughly a ≈ 3.99 Å, c ≈ 4.04 Å, with displacements of order 0.05–0.1 Å. You set this up with `ibrav = 6` (simple tetragonal) and displaced atomic positions:

```
&SYSTEM
    ibrav       = 6
    celldm(1)   = 7.539           ! a in Bohr
    celldm(3)   = 1.013           ! c/a ratio
    nat         = 5
    ntyp        = 3
    ecutwfc     = 60
    ecutrho     = 480
    occupations = 'fixed'
/
...
ATOMIC_POSITIONS crystal
  Ba 0.00 0.00  0.000
  Ti 0.50 0.50  0.515    ! shifted along z
  O  0.50 0.50 -0.025    ! apical O, shifted opposite
  O  0.50 0.00  0.490
  O  0.00 0.50  0.490
```

Running SCF on both structures and comparing `total energy` gives you the **energy gain from ferroelectric distortion** — your first real ferroelectric result.

### Running it

With your input file `scf.in` and your SLURM script `run_scf.slurm` in the same directory in scratch:

```bash
sbatch run_scf.slurm
```

When it finishes, the key lines in `scf.out`:

```
!    total energy              =    -XXX.XXXXXXXX Ry
     total force               =     0.0XXXXX     
     highest occupied, lowest unoccupied level =    X.XXX   X.XXX eV
     convergence has been achieved in  N iterations
```

The `!` before `total energy` marks the final converged value — that's what you want.

### Pulling energies out

```bash
grep '!    total energy' scf.out | tail -n 1 | awk '{print $5}'
```

Converts to scripts nicely. **1 Ry = 13.6057 eV**; multiply by that to compare with experimental data in eV.

---

## Part 7 — Convergence Testing (non-negotiable)

Any DFT number you report must be converged with respect to:

1. **Plane-wave cutoff `ecutwfc`** — controls basis set completeness.
2. **Density cutoff `ecutrho`** — for USPP/PAW, typically 8× `ecutwfc`; for norm-conserving, 4×.
3. **k-point mesh** — samples the Brillouin zone.

Convergence criterion for total energies: **< 1 meV/atom** is standard. For energy *differences* (like our paraelectric vs. ferroelectric), convergence is even more important because small errors subtract and what's left might be noise.

### k-point convergence script

```bash
#!/bin/bash
for k in 4 6 8 10 12; do
    dir="kpt_${k}"
    mkdir -p $dir
    sed "s/K_POINTS automatic/K_POINTS automatic\n  $k $k $k 0 0 0/" \
        scf_template.in | \
        grep -v "^  8 8 8 0 0 0$" > $dir/scf.in
    # (submit SLURM job in each directory — or run inline)
done
```

Plot total energy vs. k-mesh density. The converged value is where the curve flattens. Do the same for `ecutwfc` (hold k fixed). **Only once these are converged** do you trust the physics.

### Exercise 6

Run SCF for cubic BaTiO₃ at `ecutwfc` = 40, 50, 60, 70, 80 Ry. Plot energy vs. cutoff. Identify the smallest cutoff that's converged to within 1 meV/atom. Do the same for k-points with the converged cutoff. Write up your choice and show your PI.

---

## Part 8 — A Complete Example Project Layout

This is how a real calculation should be organized. Not optional — future you will thank present you.

```
~/scratch/uXXXXXXX/batio3_project/
├── pseudo/
│   ├── Ba.pbe-spn-kjpaw_psl.1.0.0.UPF
│   ├── Ti.pbe-spn-kjpaw_psl.1.0.0.UPF
│   └── O.pbe-n-kjpaw_psl.1.0.0.UPF
├── 01_convergence/
│   ├── ecutwfc/
│   │   ├── run_40/ scf.in scf.out ...
│   │   ├── run_50/
│   │   └── ...
│   └── kpoints/
├── 02_cubic_scf/
│   ├── scf.in
│   ├── run_scf.slurm
│   └── scf.out
├── 03_tetragonal_scf/
│   └── ...
├── 04_analysis/
│   ├── collect_energies.sh
│   ├── energies.dat
│   └── plot.py
├── NOTES.md                  <- what you did, what you found, what broke
└── README.md
```

Keep a running **NOTES.md** in the project root. Every time you start a new calculation, write down: the date, what you're testing, what you changed from the last run, and the result. This is your lab notebook. The version in your head isn't good enough — you will forget.

---

## Part 9 — Common Errors and How to Read Them

| Symptom | Likely cause | Fix |
|---|---|---|
| `from pseudo_dir : error # 1 : file not found` | wrong `pseudo_dir` or misspelled UPF filename | check path with `ls`, fix filename |
| `convergence NOT achieved` | bad mixing, bad starting positions, or metal treated as insulator | lower `mixing_beta` to 0.3, set `occupations='smearing'`, `smearing='gaussian'`, `degauss=0.01` for metals |
| `Error in routine ... wrong number of atoms` | `nat` inconsistent with `ATOMIC_POSITIONS` | count atoms again |
| `Not enough memory` | cutoffs too high for node, or too few MPI ranks | request more memory, more nodes, or reduce cutoff |
| Job dies immediately in SLURM with `account invalid` | wrong `--account` | ask PI for correct name |
| Job pending forever | requested too much time or wrong partition | reduce `--time`, check partition limits |
| Energy oscillates wildly | ionic positions bad or functional/pseudo mismatch | sanity-check the input; verify all UPFs are the same functional |

### The debugging mindset

1. Read the error. Actually read it, don't panic.
2. `tail -n 50 scf.out` — the real error is usually near the end.
3. Check `slurm-<jobid>.err` — SLURM or MPI errors land here.
4. Simplify: reduce the system to the smallest thing that reproduces the bug.
5. Then ask. Your PI and senior group members want to help, but they want to see that you tried.

---

## Part 10 — Where to Go Next

You now have the foundations. Real ferroelectrics research from this starting point:

1. **Geometry relaxation** (`calculation = 'vc-relax'`) — let QE optimize the lattice and atomic positions. Essential for finding the true ferroelectric ground state.
2. **Born effective charges** — use `ph.x` (the phonon code in QE). The Z* tensors tell you how polar each atom is and are directly linked to polarization.
3. **Spontaneous polarization** via the **Berry phase** (`calculation = 'nscf'` + a Berry-phase input, or `pw2wannier90` + Wannier functions). This is the real modern-theory-of-polarization calculation and is the headline number for any ferroelectric.
4. **Phonons and soft modes** — in the paraelectric phase, the ferroelectric instability shows up as an imaginary phonon frequency at the zone center (the "soft mode"). Computing this with `ph.x` is a beautiful check.
5. **Strain engineering** — apply biaxial or epitaxial strain and watch the polarization change. Directly relevant to thin-film ferroelectrics.
6. **More exotic functionals** — LDA and PBE both have known issues for ferroelectrics (LDA under-predicts volume, PBE over-predicts). Meta-GGAs like **SCAN** are often better for these systems.

### Resources worth bookmarking

- **QE website and documentation:** quantum-espresso.org (input variable reference is gold)
- **CHPC documentation:** chpc.utah.edu/documentation/
- **Materials Project:** materialsproject.org — known structures with lattice parameters you can use as starting points
- **SSSP pseudopotential library:** materialscloud.org/discover/sssp
- **QE forum / mailing list:** when you're truly stuck, search their archives first — someone almost always had your problem

### How to learn more deeply

- Work a textbook chapter on DFT alongside doing calculations. Richard Martin's *Electronic Structure* is the standard reference; Sholl & Steckel's *Density Functional Theory: A Practical Introduction* is more approachable.
- For ferroelectrics specifically, look up Resta's review of the modern theory of polarization, and the reviews by Rabe and Ghosez.
- Read the QE tutorials at the Materials Cloud hands-on school archives. They're free and excellent.

---

## Final checklist — can you do all of these unaided?

- [ ] Log into CHPC and navigate between home and scratch
- [ ] Write, save, and make executable a bash script
- [ ] Load the QE module and locate `pw.x`
- [ ] Submit a SLURM job and monitor it with `squeue` / `sacct`
- [ ] Write a QE SCF input for a simple crystal from scratch
- [ ] Run an SCF calculation and extract the total energy with grep/awk
- [ ] Run a k-point and `ecutwfc` convergence study and defend your choice of parameters
- [ ] Organize a project directory cleanly and keep a running NOTES.md
- [ ] Read an error message and find the actual cause before asking for help

When every box is checked, you're ready for real research tasks. Bring your converged cubic and tetragonal BaTiO₃ energies to your next group meeting — that's your ticket in.

---

*Last updated: start-of-semester. Edit freely as CHPC and group conventions evolve.*
