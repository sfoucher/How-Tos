# How-To Calcul Canada

## Scope

The goal is to document how to use Calcul Québec infrastructure for deep learning projects.

**ToDO**:

- [X] Account Creation
- [ ] Environement Setup
- [ ] How to interact with the infrastructure
- [ ] How to push jobs
- [ ] Best practices and receipes


## Account Creation
First, you need to create an account on ComputeCanada :
https://ccdb.computecanada.ca/account_application

Your sponsor should be an accredited person who already has access, your sponsor needs to give you a Compute Canada Role Identifier (CCRI) (ex: `pev-1234-01`). *Important*: the account must be created with an email address from the same institution as your sponsor.

Then, you will have to activate this new account through the CalculQuebec portal:
http://www.calculquebec.ca/en/academic-research-services/procedures/

Available Servers:
* [Beluga](https://www.calculquebec.ca/communiques/beluga-un-superordinateur-pour-la-science/) 
* [Cedar](https://docs.computecanada.ca/wiki/Cedar)
* [Helios](https://docs.computecanada.ca/wiki/H%C3%A9lios/en) 

Only Helios is a full GPU cluster, Beluga and Cedar have a few nodes with GPUs.

The server status is available here:
https://status.computecanada.ca/

### Login

#### Linux
Using SSH for Beluga:
```bash=
ssh -X <username>@helios.calculquebec.ca
```
For Cedar:
```bash=
ssh -X <username>@cedar.computecanada.ca
```
:::info 
* Use the same password as your Compute Canada account. 
:::
* Type `logout` to exit
#### Windows
you can use the Ubuntu SubSystem for Windows ([WSL](https://docs.microsoft.com/en-us/windows/wsl/install-win10)) to obtain a console with SSH/SCP capability, or use a native application such as [MobaXTerm](https://mobaxterm.mobatek.net/download.html). In the latter case, choose the SSH connection and fill in the parameters as illustrated below.

## Transferring your data
https://docs.computecanada.ca/wiki/Transferring_data
### scp

### Globus
https://docs.computecanada.ca/wiki/Globus

## Environment Setup
### Installation of the necessary modules and learning environment
CalculQuebec provides `packages` that are specifically prepared to be compatible with the cluster computing environment; these can be loaded using the module load `<MOD_NAME>` command.
In your terminal, first run : 
```bash=
module load python/3.6
module load scipy-stack
```
This should make available in your environment a standard version of the Python interpreter (3.6) in addition to some very useful packages for scientific computation (e.g. Numpy).
Next, you need to create a virtual environment with [virtualenv](http://pypi.python.org/pypi/virtualenv) that will use the now loaded Python interpreter to perform your learning tasks. To do this : 
```bash=
virtualenv pytorch-env
```
By default, the environment named `pytorch-env` will be created (if you have not changed your working location) in your "home" folder. This folder does not yet contain the packages that really interest us (e.g. PyTorch), so we have to install them ourselves. To do this, first activate the environment via : 
```bash=
source ~/pytorch-env/bin/activate
```
Note that the above command is the one that should be used to reactivate the already installed environment when you later reconnect to the cluster. 
To continue the installation, run the following command: 
```bash=
pip install numpy torch_gpu torchvision matplotlib opencv_python scikit_learn scikit_image scipy cython --no-index
```
Note that this is a single line that should be copied in its entirety. It will install by PIP all the dependencies necessary for the execution of a learning task in the virtual environment on one of the cluster nodes.

**Optional step**:
Once a task is submitted to the cluster, it will be executed on a compute node which, unlike the login node used so far, is probably not connected to the Internet. So if you need to start your training from a pre-trained model, you need to make sure that it is already downloaded to your computing environment. Here we provide the necessary command to download a pre-trained ResNet18 model using torchvision : 
```python=
python -c "import torch; import torchvision; import torch.utils.model_zoo as model_zoo; model_zoo.load_url(torchvision.models.resnet.model_urls['resnet18'])"
```
## Launching a learning task 

Compute Canada is using [Slurm](https://slurm.schedmd.com/overview.html) for job management.

![](https://i.imgur.com/QaGJXx8.png)

Once your environment is in place, you will need to write a script that activates it and executes your learning task. You can write this script on your own machine and then transfer it to the login node, or write it directly to its destination using e.g. [Vim](https://www.vim.org/).
Here is an example of a script that can be executed on the cluster: 
```bash=
#!/bin/bash
#PBS -N <JOB_NAME>
#PBS -A <PROJECT_ID>
#PBS -l walltime=<MAX_TIME>
#PBS -l nodes=1:gpus=1
#PBS -j oe
module load python/3.6
module load scipy-stack
source ~/pytorch-env/bin/activate
cd "${PBS_O_WORKDIR}"
echo "starting..."
python main.py
echo "all done"
```
In this script you have to specify the fields surrounded by < ... > (but without including these characters!). 
The `JOB_NAME` is used to identify your task on the cluster (a number will also be automatically associated to the task, but the name is useful to track its execution). 

The `PROJECT_ID`is the mandatory identifier to determine which resources are available to you on the cluster (it comes from your sponsor, and is in the form `abc-123-aa`). 

The `MAX_TIME` is the maximum number of seconds your script should run. Finally, the main.py script should contain your training code. 
For more information on script content, see :
https://docs.computecanada.ca/wiki/Compute_Canada_Documentation

Finally, to run your script on one of the compute nodes of the cluster, execute the following command: 
```bash=
qsub <SCRIPT_NAME>.sh
```
Note that the scripts provided to `qsub` must contain the `#PBS` parameters mentioned above, and that these cannot be Python scripts directly. If everything works, `qsub`should return a job number, and your training should start soon on one of the cluster nodes.
You can remove a task already started using `qdel`: 
```bash=
qdel <JOB_ID>
```
### Quick Test on Cedar
Once logged on Cedar, create this script:
```bash=
#!/bin/bash
#PBS -N test-cuda
#PBS -A def-fouchers
#PBS -l walltime=10
#PBS -l nodes=1:gpus=1
#PBS -j oe
module load python/3.6
module load scipy-stack
source ~/pytorch-env/bin/activate
cd "${PBS_O_WORKDIR}"
echo "starting..."
python -c "import torch; print('CUDA AVAILABLE: ' + str(torch.cuda.is_available()))"
echo "all done"
```
The script must be moved to and launched from `/scratch/<username>` or to `/project/def-<your_sponsor>/<username>` where `<username>` is your account name. On Cedar, the job is pushed using `sbatch`:
```bash=
sbatch test.sh 
Submitted batch job 61770526
```
and the job queue can be checked with `sq`:
```bash=
JOBID     USER      ACCOUNT           NAME  ST  TIME_LEFT NODES CPUS TRES_PER_N MIN_MEM NODELIST (REASON) 
61770526 fouchers def-fouchers      test-cuda  PD      10:00     1    1      gpu:1    256M  (Priority)
```
The output will be in a `slurm-` file in the same directory:
```bash=
-rw-r----- 1 fouchers fouchers  39 Feb 12 08:24 slurm-61770526.out
-rw-r----- 1 fouchers fouchers 325 Feb 12 08:17 test.sh
```
Then you can check the content with `cat slurm-61770526.out`
## Mapping Distant Folder using sshfs
`usage: sshfs [user@]host:[dir] mountpoint [options]`

We first create a local directory:
```bash=
export USER_AT_HOST=username@cedar.computecanada.ca
mkdir -p '$HOME/sshfs/$USER_AT_HOST'
```
The we mount to the distant directory:
```bash=
sshfs "$USER_AT_HOST:" "$HOME/sshfs/$USER_AT_HOST" -p 22 -o workaround=nonodelay -o transform_symlinks -o idmap=user  -C
```
so `$HOME/sshfs/$USER_AT_HOST` will map the `HOME` directory on Cedar.

## Varia

## Ressources
* https://docs.computecanada.ca/mediawiki/images/e/e7/Accessing_CC_Compute_and_Storage_Environment.pdf
* Compute Canada youtube channel: https://www.youtube.com/channel/UC2f3cwviToj-mazutBNhzFw
* AI and Machine Learning: https://docs.computecanada.ca/wiki/AI_and_Machine_Learning
