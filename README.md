# Tips for running JupyterLab through the UEA's ADA

## Connect to ADA
Log in to ADA following the [HPC teams instructions](https://my.uea.ac.uk/divisions/it-and-computing-services/service-catalogue/research-it-services/hpc/ada-cluster/connecting-to-ada)

*Note:* Ensure you launch an interactive session for any conda environment configuration, etc., as these processes are too computationally expensive for the login node. To view the help documentation on interactive session within ADA use the command:

## Initial setup 

### Load conda to be used interactively. 

```console
interactive -h
```
Once in an interactive node you can view the available versions of python using:

```console
module spider python
```
Select an available version of python anaconda (e.g. 3.8) and then load it using:

```console
module add python/anaconda/2020.11/3.8
```

### Create a conda environment for JupyterLab to use. 

Before you can submit a job to initialise a JupyterLab session you must make sure the conda environment you load exists. For this example I have used the environment provided in AIRESenv.yml - which should be suitable for running the python training courses on [UEApy](https://github.com/ueapy).

Note: there is a current (Feb 2023) conflict within cartopy and matplotlib 3.6 when running JupyterLab on ADA. To avoid this ensure you have added cartopy constrained to version 0.21

If you want to use the attached .yml file for creating a conda env then upload the .yml file to your user space and use:

```console
conda env create --name AIRESenv --file=AIRESenv.yml
```

Note: If conda cannot find a way to install this environment it may be easier to manually create an environment using

```console
conda create -n AIREenv python=3.8.5 seaborn matplotlib numpy jupyter jupyterlab netcdf4 xlwt xlrd owslib xarray ipython spyder dask pandas
conda activate AIRESenv
conda install -c conda-forge cartopy=0.21
```

You may experience python version conflicts between the node environment you are working in and the base conda environment on ADA. If this happens then you need to stop nested environment activation. To do this use:

```console
conda config –-set auto_activate_base false
```

This creates a .condarc file with the nested environments turned off. 

To double check that there is no conflict of your python versions use the following commands:

```console
which python
python -c 'import sys; print(sys.prefix)' 
```
The output should looks something like this: 

![python_version_check](https://user-images.githubusercontent.com/111057180/217242277-83ae56cc-515b-4bfa-8a70-b3189a0e23b5.png)

where the paths that should match are highlighted in a red box.

### Create a batch submission script

To submit the JupyterLab instructions to ADA you need to create a batch job submission script. I have uploaded an example script - AIRESconda_sub.sh

In this file areas **you will need to modify** are:

- *Line 26*: Update `abc12def@ada.uea.ac.uk` where 'abc12def' should be your 8 digit UEA username.

In this file areas you may want to modify are:

- *Line 3*: The ADA partition the job is being submitted to.
- *Line 4*: The memory you are asking for (how much memory you think your JupyterLab session will use).
- *Line 5*: How long you want the JupyterLab session to last.
- *Line 6-8*: The job-name and output files (these are important).
- *Line 11*: Which python version you are using, this should match your conda env python version. 

## Submit your JupyterLab script

Note: Now we are past the initial setup. For future sessions you can start at this step and submit the job from the login node. 

There are three suitable partitions for running JupyterLab on ADA:

| Partition | Max Time | Default Memory Per CPU
| ----------- | ----------- | ----------- |
| compute-24-128 | 7 days | 5144 |  *due to be decommisioned shortly 
| compute-24-96 | 7 days | 3772 |  *default partition
| compute-64-512 | 7 days | 7975 |

You can check how busy the partitions are using `snoderes`. For example:

```console
snoderes -p compute-24-96
```

Once your batch file is uploaded and configured you can then submit the job using:

```console
sbatch AIRESconda_sub.sh
```

The job progress can be monitored using:

```console
squeue -u <Your 8 digit username>
```

Here can view all of your submitted jobs and their associated `JOBID`.

The job can be killed using:

```console
scancel <JOBID>
```

*Note*: It is good form to kill any active JupyterLab sessions when you no longer need them. This frees up the resources you have requested.

## Form the tunnel to your JupyterLab session

At this point you should (hopefully) have a JupyterLab session running on one of ADA's partitions which can access the modules loaded in your conda environment. From AIRESconda_sub.sh two files will be created - an output (.out) file and an error (.err) file. You may need to give it a minute for the .err file to populate fully. 

1) **.out file** (e.g. AIRESconda.out). This provides the `ssh` instruction for the local tunnel set up.  Copy the `ssh` command from the file (I usually open it directly using nano and copy the command). Use your systems local command prompt or terminal to submit the `ssh` command. If it is successful it will look as though it has frozen.
2) **.err file** (e.g. AIREDconda.err) This provides the url to access the Jupyter Lab session via your browser of choice. 
I recommend that you use the url that starts with `‘http://127.0.0.1:8888/lab?token=…’` and Google Chrome. It may take a few seconds to load.

## Potential 8888 port issues

### ADA 8888 port issues

In rare cases the 8888 port is already in use on the node selected by ADA. When this happends the `ssh` instruction in the .out file will not match the target url provided by the .err file. This is because ADA is smart and has tried the next available port to stop your submitted job from crashing. The correct port will always be displayed correctly in the .err file (e.g. it may use port 8881 if 8888 is busy) update your `ssh` command to match this.

For example if the url provided is: `http://127.0.0.1:8881/lab?token=…` then the correct port to use is **'8881'** and your `ssh` command would be `ssh -N -L 8881:c0012:8881 ...`

### Local 8888 port issues

If the 8888 port is already in use on the local machine then an error will occur. You will need to clear the port for the tunnel to configure itself properly. 

Windows:
```console
netstat -ano | findstr 8888    ## Locate the task ID that is using port 8888
taskkill /F /pid <TASKID>      ## Kill the task 
```

Linux:
```console
netstat -lnp | grep 8888          ## Locate the task ID that is using port 8888
kill -9 <TASKID>                  ## Kill the task
```
where `<TASKID>` represents the Task ID returned in the first line of code. 

## Housekeeping

At this stage you should have a working JupyterLab session on a remote node within ADA that you can access using a local browser. 

As the JupyterLab session is created via a batch job script you can cancel any interactive sessions and close login in nodes and your JupyterLab session will still be running. If you accidently close the browser just find the url again from the .err file and reloaded it. 

The JupyterLab session will finish when you manually cancel the job (`scancel <JOBID>`) or it runs out of allocated time. 

## UEA HPC - Further Information and Links

The UEA HPC team have provided information on ADA's software (including conda and python) within their [HPC itranet pages](https://my.uea.ac.uk/divisions/it-and-computing-services/service-catalogue/research-it-services/hpc/ada-cluster/ada-software). 

The HPC team also have some virtual training on ADA, Slurm etc. which is worth a look and is located on [Planet eStream](https://utv.uea.ac.uk/Default.aspx?search=HPC&saml=1).

If you have any issues I would recommend contacting the HPC team as they are the experts. The information provided here is based on what I encountered when I first set up JupyterLabs myself and is not endorsed by the UEA or HPC team in any way.



