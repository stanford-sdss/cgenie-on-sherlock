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

## Step 3.2: Submitting the Model as a Batch Job

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

## Step 3.3 Submiting an Ensemble
### Step 3.3a Make Ensemble

In your local [muffindata directory](https://github.com/derpycode/muffindata.git) use 'fun_make_ensemble_2d.m' to generate an ensemble. Will look something like 'date.str_parms.dat.##'.

### Step 3.3b Rename Ensemble Files
Rename ensemble (if desired) using the following MATLAB script. 

```bash
folder = pwd;
date = 'date';  % as a character not datetime
prefix = 'prefered_prefix';

files = dir(fullfile(folder, ['*' date '*']));

for i = 1:length(files)
    oldname = files(i).name;
    newname = replace(oldname, date, prefix);

    movefile(fullfile(folder, oldname), fullfile(folder, newname));
end
```
### Step 3.3c Generate submission file

Use the below MATLAB script to make a list of files to submit.
You need to ensure the text file generated (ensembles.txt -- or whatever you choose to name it) has unix escape characters. 

```bash
%-------------Conditions to change
userconfig = 'prefered_prefix.dat'; 
restart = '';
year = '10000';
baseconfig = 'baseconfig_name.config'; 
subdir = 'subdir'; %subfolder within genie-userconfigs that you are saving userconfigs

%------------ Don't modify
userconfig_list = dir(fullfile(pwd, [userconfig '*']));
userconfig_list = {userconfig_list.name}';  % extract names as cell column vector

num_userconfigs = numel(userconfig_list);
baseconfig_list = repmat({baseconfig}, num_userconfigs, 1);
restart_list = repmat({restart}, num_userconfigs, 1);
year_list = repmat({year}, num_userconfigs, 1);
subdir_list = repmat({subdir}, num_userconfigs, 1);

T = table(baseconfig_list, subdir_list, userconfig_list, year_list, restart_list);
T(26, :) = []; %remove the .key file (no this is not fool proof-- i'm lazy)

writetable(T, 'ensembles.txt', 'Delimiter', ' ', 'WriteVariableNames', false); % or name it whatever suits you
```
### Step 3.3d Submit Ensemble as Batch Job

Transfer 'ensembles.txt' to your HOME directory (or wherever you are running cGENIE.muffin iteratively).
To submit a batch job of an ensemble, you want a file 'cgenie.sh' will the following contents.

```bash
#!/bin/bash
cd $HOME/cgenie.muffin/genie-main
export CGENIE_MUFFIN_LOCATION=$PWD
./runmuffin.sh $BASECONFIG $DIR $USERCONFIG $YEAR $RESTART
```

And a file 'submit_ensemble.sh'

```bash
#!/bin/bash
#SBATCH --job-name=genie-ensemble
#SBATCH --output="cgenie_output/%A_%a.out"
#SBATCH --array=0-24         # for a 5x5 grid. Change depending on your ensemble size
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=8
#SBATCH --partition=serc
#SBATCH --mem=50GB
#SBATCH --time=24:00:00       # allocate time. It takes about 8 hr for a 10k model your run. 
#SBATCH --mail-type=ALL
#SBATCH --mail-user=YOUR-EMAIL # optional

IFS=' ' read -r BASECONFIG DIR USERCONFIG YEAR RESTART < <(sed -n "$((SLURM_ARRAY_TASK_ID + 1))p" ensembles.txt)

echo "BASECONFIG: $BASECONFIG, DIR: $DIR, USERCONFIG: $USERCONFIG, YEAR: $YEAR, RESTART: $RESTART"

export BASECONFIG DIR USERCONFIG YEAR RESTART

apptainer exec \
  --env BASECONFIG=$BASECONFIG \
  --env DIR=$DIR \
  --env USERCONFIG=$USERCONFIG \
  --env YEAR=$YEAR \
  --env RESTART=$RESTART \
  cgenie_latest.sif /bin/bash cgenie.sh

```

Run in your console $sbatch submit_ensemble.sh

