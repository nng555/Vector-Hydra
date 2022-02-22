# Slurm Launcher Plugin

Plugin files for launching jobs on a slurm cluster with Hydra. Currently works only with Hydra 1.0 to accomodate the use of fairseq as well, but fixing it to work with the latest dev version isn't too hard.

To install, run the following commands:

```
git clone -b 1.0_branch https://github.com/facebookresearch/hydra.git
git clone https://github.com/nng555/slurmlauncher.git
cp slurmlauncher/slurm_utils.py hydra/hydra/slurm_utils.py
cp slurmlauncher/slurm_launcher.py hydra/hydra/_internal/core_plugins/slurm_launcher.py
cd hydra
pip install -e .
```

This will install the 1.0 version of hydra with the slurm launcher plugin installed. To customize the launcher for your specific environment, you will need to change the global variables set in `slurm_utils.py`.

## Using the Launcher

This slurm launcher acts as a layer of abstraction on top of a system for autogenerating `.sh` and `.slrm` files and launching slurm jobs. It presents a simple way to run many sets of repeatable experiments and organize the outputs with minimal overhead. The core of the plugin is the hydra configuration files.

### Setting up the Configuration
In order to use the slurm launcher, you should create a configuration folder that will contain the `.yaml` configuration files for your project as well as some base config files. An example base configuration is provided in `base_conf`. The basic structure of the folder is shown below.

```
.
├── config.yaml
└── slurm
    ├── cpu.yaml
    └── default.yaml
```

`config.yaml` stores the top level defaults for each group as well as configs for the launcher itself. Additional keys and defaults can be added here as necessary. Details on what each key is used for are provided below:
- `venv/conda`: name of virtual environment to run job with. If both are provided, `conda` takes priority
- `max_running`: max \# of running jobs. If limit is reached, launcher will wait until jobs have completed then continue launching
- `max_pending`: max \# of pendng jobs. If limit is reached, launcher will wait until jobs have launched then continue launching
- `max_total`: max \# of total jobs. If limit is reached, launcher will wait until jobs have launched or completed then continue launching

`slurm/` contains the configuration for the slurm job. These values can be easily tweaked or overriden to fit whatever slurm cluster you are running on. To add additional slurm options, simply add in the key value pair into this file (or another alternate configuration) and it will automatically be added to the `.slrm` file.

### Dynamic Evaluation
This plugin adds in dynamic evaluation into configuration processing. Dynamic evluation is invoked using the keyword `eval:` at the start of a value. The launcher will then evaluate the expression after this keyword using python and interpolate the result as the value for that particular key.

As an example, in the `slurm/default.yaml` configuration file, we have the key/value pairs
```
cpus_per_task: eval:4*int("${slurm.gres}"[-1])
gres: gpu:1
```
Hydra will first interpolate `${slurm.gres}`, then evaluate the expression `4*int("gpu:1"[-1])` which returns `4`. This is then the value of `cpus_per_task` during runtime. 
Dynamic evaluation is useful for setting arguments based on the value from other arguments. In this case, we would like to request 4 more CPUs than there are GPUs.

### Decorating a Function
Modifying an existing script to run with the slurm launcher is simple. First add in the import statements
```
import hydra
from omegaconf import DictConfig
from hydra import slurm_utils
```
to the top of your script. The `__main__` function of the script should be as follows:
```
if __name__ == "__main__":
    my_method()
```
Finally, add the decorator and modify the method signature as follows:
```
@hydra.main(config_path='/path/to/base_conf', config_name='config')
def my_method(cfg: DictConfig):
    slurm_utils.symlink_hydra(cfg, os.getcwd())
    ...
```
All values in the configuration will then be available to your script from within the `cfg` object. As an example, the `run_bash.py` script launches a given bash command (stored within `cfg.command`) on the slurm cluster.
Adding in the final symlinking command is not necessary but is useful if you need to check certain log easily. Note that including this line means that jobs cannot run locally.

### Running a Job
To launch your script on the cluster, simply run
```
python3 my_script.py -m
```
and that's it! The plugin will automatically generate the required `.sh` and `.slrm` files then launch your job for you.


