# REPIC
<ins>RE</ins>liable <ins>PI</ins>cking by <ins>C</ins>onsensus (REPIC)

## Overview
REPIC is an ensemble learning approach to cryogenic-electron miscroscopy (cryo-EM) particle picking. It identifies particles common to multiple picked particle sets (i.e., consensus particles) using graph theory and integer linear programming (ILP). Picked particle sets may be found by a human specialist (manual), template matching, mathematical function (e.g., RELION's Laplacian-of-Gaussian auto-picking), or machine-learning method. A schematic representation of REPIC applied to the output of three CNN-based particle pickers is below:

<p align="center">
<img width="60%" src="imgs/repic_overview.png">
</p>

REPIC expects particle sets to be in BOX file format (*.box) where each particle has coordinates, a detection box size (in pixels), and (optional) a score [0-1].

## Software requirements
Required:
1. Python v3.8 interpreter ([Miniconda](https://docs.conda.io/en/latest/miniconda.html) installation recommended)
2. [Gurobi ILP optimizer](https://www.gurobi.com/products/gurobi-optimizer/) (v9.5.2 used) - requires free [academic license](https://www.gurobi.com/downloads/)
3. Python package dependencies described in [setup.py](setup.py)
4. _Windows users_ - [Ubuntu terminal environment with Windows Subsystem for Linux (WSL)](https://ubuntu.com/tutorials/install-ubuntu-on-wsl2-on-windows-10#1-overview) (v22.04.2 LTS tested)

*Optional:* 
1. [REgularised LIkelihood OptimisatioN (RELION)](https://relion.readthedocs.io/en/release-3.1/) - particle and density analyses (v3.13 used)
2. [UCSF Chimera](https://www.cgl.ucsf.edu/chimera/) - map alignment and density visualization (v1.16 used)

## Installation guide
REPIC installation is expected to only take a few minutes:

1. Either download the package by clicking the "Clone or download" button, unziping file in desired location, and renaming the directory "REPIC" OR using the following command line:\
``` git clone https://github.com/ccameron/REPIC ```
2. [Install Miniconda](https://docs.conda.io/en/latest/miniconda.html#latest-miniconda-installer-links) if the ``` conda ``` command is unavailable
3. ``` cd <install_path>/REPIC ```
4. Create a separate Conda environment and install Gurobi for REPIC:\
``` conda create -n repic -c gurobi python=3.8 gurobi```
5. Activate REPIC Conda environmnet:\
``` conda activate repic ```
6. Install REPIC using [pip](https://en.wikipedia.org/wiki/Pip_(package_manager)):\
``` pip install . ```
7. [Obtain Gurobi license](https://www.gurobi.com/academia/academic-program-and-licenses/) and set Gurobi key ``` grbgetkey <gurobi_key> ``` 
8. Remove unused or temporary Conda files:\
``` conda clean --all ```

To check if REPIC was correctly installed, run the following command:\
``` repic -h ```\
A help menu should appear in the terminal.

## Example data
Example [SPHIRE-crYOLO](https://cryolo.readthedocs.io/en/stable/), [DeepPicker](https://github.com/jianlin-cheng/DeepCryoEM), and [Topaz](https://github.com/tbepler/topaz) picked particle coordinate files for $\beta$-galactosidase ([EMPIAR-10017](https://www.ebi.ac.uk/empiar/EMPIAR-10017/)) micrographs are found in [ examples/10017/](examples/10017/).  These files were generated by applying the pre-trained pickers to $\beta$-galactosidase micrographs, filtering false positive per author suggested thresholds, and then converting files to BOX format using [coord_converter.py](repic/utils/coord_converter.py).

Example motion corrected T20S proteasome ([EMPIAR-10057](https://www.ebi.ac.uk/empiar/EMPIAR-10057/)) micrographs and normative particles for iterative ensemble particle picking are freely available via Amazon Web Services (AWS). To download this data, please run [get_examples.sh](repic/iterative_particle_picking/get_examples.sh) (see Quick start below for how to run this Bash script).

Installation instructions for SPHIRE-crYOLO, DeepPicker, and Topaz are found in [docs/](docs/).

Example commands for fitting and running SPHIRE-crYOLO, DeepPicker, and Topaz models are found in [repic/iterative_particle_picking/](repic/iterative_particle_picking).

Parameters used for particle picking algorithms and RELION are found in [supplemental_data_file_2.ods](supp_data_files/supplemental_data_file_2.ods).

## Quick start
### Creating consensus particle sets
1. Calculate the particle overlap ([Jaccard index](https://en.wikipedia.org/wiki/Jaccard_index) \[JI\]) and enumerate cliques using [get_cliques.py](repic/commands/get_cliques.py) (expected run time: 1-3 mins):

``` repic get_cliques examples/10017/ examples/10017/clique_files/ 180 ```

Note - REPIC will use the folder names found in the provided input directory (e.g., [``` examples/10017/ ```](examples/10017/)) to assign method labels (e.g., "crYOLO", "deepPicker", "topaz")

Correctly executing the above command will produce the following files for each micrograph in the output folder ``` examples/10017/clique_files/ ```:
  - *_clique_coords.pick: [pickled](https://docs.python.org/3/library/pickle.html) clique (*x*,*y*) coordinates (in BOX format)
  - *_constraint_matrix.pickle: pickled Gurobi constraint matrix file
  - *_weight_vector.pickle: pickled Gurobi clique weight vector file
  - *_runtime.tsv: runtime tracking TSV file

2. Find optimal cliques using the ILP solver (Gurobi) and create consensus particle BOX files using [run_ilp.py](repic/commands/run_ilp.py) (expected run time: <1 min):

``` repic run_ilp examples/10017/clique_files/ 180 ```

Correctly executing the above command will produce a particle coordinate file (in BOX format) for each micrograph  in the output directory ``` examples/10017/clique_files/ ```. The final column in these BOX files represents the clique weight for a consensus particle.

### Particle picking by iterative ensemble learning
1. Download example data from AWS S3 bucket using [get_examples.sh](repic/iterative_particle_picking/get_examples.sh) (expected run time: 1-5 mins):

``` bash $(pip show repic | grep -in "Location" | cut -f2 -d ' ')/repic/iterative_particle_picking/get_examples.sh examples/10057/data/ &> aws_download.log ```

2. Create a configuration file for iterative ensemble particle picking using [iter_config.py](repic/commands/iter_config.py) (expected run time: <1 min):

``` repic iter_config examples/10057/ 176 224 <file_path>/gmodel_phosnet_202005_N63_c17.h5 <file_path>/DeepPicker-python 4 22 ```

``` <file_path> ``` must be replaced with the full file paths to the SPHIRE-crYOLO pre-trained model and DeepPicker directory, respectively. See picker installation instructions in [docs/](docs/) for more information.

A configuration file ``` iter_config.json ``` will be created in the current working directory.

3. Pick particles by iterative ensemble learning using a Python script wrapper [iter_pick.py](repic/commands/iter_pick.py) of [run.sh](repic/iterative_particle_picking/run.sh) (expected run time: 20-30 min/iteration):

``` repic iter_pick ./iter_config.json 4 100 ```

The final set of consensus particles for the testing set should be found in:
``` examples/10057/iterative_particle_picking/round_4/train_100/clique_files/test/*.box ```

## Command line details
### Identifying consensus particle sets with REPIC
1. Calculating particle overlap (JI) and enumerate cliques using [get_cliques.py](repic/commands/get_cliques.py):

``` 
usage: repic get_cliques [-h] [--multi_out] [--get_cc] in_dir out_dir box_size

positional arguments:
  in_dir       path to input directory containing subdirectories of particle coordinate files
  out_dir      path to output directory (WARNING - script will delete directory if it exists)
  box_size     particle detection box size (in int[pixels])

options:
  -h, --help   show this help message and exit
  --multi_out  set output of cliques to be members sorted by picker name
  --get_cc     filters cliques for those in the largest Connected Component (CC)
  ```

2. Finding optimal cliques using ILP solver (Gurobi) and creating consensus particle BOX files using [run_ilp.py](repic/commands/run_ilp.py):

``` 
usage: repic run_ilp [-h] [--num_particles NUM_PARTICLES] in_dir box_size

positional arguments:
  in_dir                path to input directory containing get_cliques.py output
  box_size              particle detection box size (in int[pixels])

options:
  -h, --help            show this help message and exit
  --num_particles NUM_PARTICLES
                        filter for the number of expected particles (int)
  ```

### Particle picking by iterative ensemble learning

1. Create a configuration file for iterative ensemble particle picking using [iter_config.py](repic/commands/iter_config.py):
```
usage: repic iter_config [-h] [--cryolo_env CRYOLO_ENV] [--deep_env DEEP_ENV] [--topaz_env TOPAZ_ENV] [--out_file_path OUT_FILE_PATH]
                         data_dir box_size exp_particles cryolo_model deep_dir topaz_scale topaz_rad

positional arguments:
  data_dir              path to directory containing training data
  box_size              particle detection box size (in int[pixels])
  exp_particles         number of expected particles (int)
  cryolo_model          path to LOWPASS SPHIRE-crYOLO model
  deep_dir              path to DeepPicker scripts
  topaz_scale           Topaz scale value (int)
  topaz_rad             Topaz particle radius size (in int[pixels])

optional arguments:
  -h, --help            show this help message and exit
  --cryolo_env CRYOLO_ENV
                        Conda environment name for SPHIRE-crYOLO installation (default:cryolo)
  --deep_env DEEP_ENV   Conda environment name for DeepPicker installation (default:deep)
  --topaz_env TOPAZ_ENV
                        Conda environment name for Topaz installation (default:topaz)
  --out_file_path OUT_FILE_PATH
                        path for created config file (default:./iter_config.json)
```

`data_dir/` is expected to contain a three-column TSV file of CTFFIND4 defocus values: (1) micrograph filename, (2) defocus x, and (3) defocus y. If this file is not found, then all micrographs will be assigned the same defocus value. A defocus file can be built from the output of a RELION CTF refinement job using the following Bash script:

```
EMPIAR_ID=<complete>  # only integers - i.e., EMPIAR-10017 would be 10017
out=<install_path>/REPIC/examples/${EMPIAR_ID}/data/defocus_${EMPIAR_ID}.txt
rm -rf ${out}
for file in <relion_path>/relion/CtfFind/job00[0-9]/<mrc_pattern>; do
  grep '' /dev/null ${file} | tail -n 1 | awk -F ":| " '{print $1,$3,$4}' >> ${out}
done
```

` <mrc_pattern> ` is dependent on the naming convention used for micrographs and will need to be set to your specific substring. For EMPIAR-10017 and -10057, the substrings are '\*0.txt' and '\*[0-9].txt', respectively.

` <relion_path>/relion/CtfFind/job00[0-9]/*<mrc_suffix> ` should list all CTFFIND4 output files in RELION's ` CtfFind/ `.

2. Iteratively pick particles using a Python script wrapper [iter_pick.py](repic/commands/iter_pick.py) of [run.sh](repic/iterative_particle_picking/run.sh):
```
usage: repic iter_pick [-h] [--semi_auto] [--score] config_file num_iter train_size

positional arguments:
  config_file  path to REPIC config file
  num_iter     number of iterations (int)
  train_size   training subset size (int)

optional arguments:
  -h, --help   show this help message and exit
  --semi_auto  initialize training labels with known particles (semi-automatic)
  --score      evaluate picked particle sets
```
``` train_size ``` references the output of [build_subsets.py](repic/commands/build_subsets.py), which builds training subsets of sizes 1%, 25%, 50%, and 100% (i.e., 100% will use the entire training set). For more information on dataset handling please see "iterative ensemble particle picking with REPIC" in the Methods section of the REPIC manuscript.

## Testing
The REPIC software has been tested on two computer systems:
1. Ubuntu 16.04.6 LTS (Xenial Xerus) running CUDA v10.1 with four Nvidia GP102 TITAN Xp
2. Ubuntu 16.04.7 LTS (Xenial Xerus) running CUDA v11.3 with four Nvidia GeForce GTX 1080

## Citing REPIC
If REPIC was used in your analysis / study, please cite:

Christopher JF Cameron, Sebastian JH Seager, Frederick J Sigworth, Hemant D Tagare, and Mark B Gerstein. REPIC - an ensemble methodology for cryo-EM particle picking. In review. DOI: [TBD](https://www.youtube.com/watch?v=u6M-Ccp60zU)