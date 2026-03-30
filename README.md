[![DOI](https://img.shields.io/badge/DOI-here-blue)](https://doi.org/10.25740/xg249tm1018)

# cgenie-on-sherlock

This repository contains the package and instructions necessary to run the "cgenie" model on Stanford's Sherlock cluster.

This was built off of the [cgenie.muffin](https://github.com/derpycode/cgenie.muffin) repository, on the master.python3 branch, given [these instructions](https://github.com/derpycode/muffindoc/blob/master/muffin3.install.Ubuntu.pdf)


## Step 1: Pull the Image on Sherlock

Login to Sherlock, and use the following apptainer command to pull the image:

```bash
apptainer pull oras://ghcr.io/stanford-sdss/cgenie-on-sherlock/cgenie:latest
```
You should now see a file named `cgenie_latest.sif`

# Step 2: Pull this repo

In the same directories in which you'd like your output stored, clone this reposity.

## Step 3: Running the Model
## Step 3.1: Running the model interactively

To run the model interactively, you'll need to first obtain an interactive session.  The easiest way to do that is via [`sh_dev`](https://www.sherlock.stanford.edu/docs/user-guide/running-jobs/#interactive-jobs), which provides 1 core and 4GB of memory for 1 hour.

If you need more resources, you can use the flags available in the `sh_dev` command.  Here's an example command that asks for 8 cores, 50GB of memory and 4 hours on the `serc` partition:

```bash
sh_dev -c 8 -m 50GB -p serc -t 4:00:00
```

Once you're allocated resources, you can run the model with the following commands:

```bash
apptainer shell cgenie_latest.sif
export CGENIE_MUFFIN_LOCATION=$PWD
cd cgenie.muffin/genie-main;
./runmuffin.sh cgenie.eb_go_gs_ac_bg.worbe2.BASE LABS LAB_0.EXAMPLE 10
```

You can replace the ./runmuffin.sh line with your specific configurations.  

Shout out to [this repository](https://github.com/b-reyes/singularity_cgenie_muffin) for showing how to use a custom output directory!

## Step 3.1: Submitting the Model as a Batch Job

To submit the model as a batch job, you'll need a submit file that runs the model via apptainer.  Here's an example of one:

```bash
#!/bin/bash
# ----------------SLURM Parameters----------------
#SBATCH --partition=serc
#SBATCH -c 8
#SBATCH --mem 50GB
#SBATCH --time=4:00:00
#SBATCH --job-name=cgenie-test
#SBATCH --output="out_files/%A_%a.out"
# ----------------Load Modules--------------------
# ----------------Commands------------------------

echo "cd cgenie.muffin/genie-main; export CGENIE_MUFFIN_LOCATION=$PWD; ./runmuffin.sh cgenie.eb_go_gs_ac_bg.worbe2.BASE LABS LAB_0.EXAMPLE 10" | apptainer exec cgenie_latest.sif /bin/bash
```

This basically submits the interactive commands above to the `apptainer exec` command.  Alternatively, you could save those commands in a file, and input it like this:

Contents of cgenie.sh:
```
cd cgenie.muffin/genie-main;
export CGENIE_MUFFIN_LOCATION=$PWD
./runmuffin.sh cgenie.eb_go_gs_ac_bg.worbe2.BASE LABS LAB_0.EXAMPLE 10
```

Contents of cgenie.submit:
```bash
#!/bin/bash
# ----------------SLURM Parameters----------------
#SBATCH --partition=serc
#SBATCH -c 8
#SBATCH --mem 50GB
#SBATCH --time=4:00:00
#SBATCH --job-name=cgenie-test
#SBATCH --output="out_files/%A_%a.out"
# ----------------Load Modules--------------------
# ----------------Commands------------------------

apptainer exec cgenie_latest.sif /bin/bash cgenie.sh
```


