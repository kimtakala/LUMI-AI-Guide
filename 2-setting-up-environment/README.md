# 2. Setting up your own environment

Machine learning frameworks on LUMI serve as isolated environments in the form of container images with a set of Python packages. LUMI uses the [Singularity](https://docs.sylabs.io/guides/main/user-guide/) (SingularityCE) container runtime. Containers can be seen as encapsulated images of a specific environment including all required libraries, tools and Python packages. Container images can be based on virtually any Linux distribution targeting the host architecture, but they still rely on the host kernel and kernel drivers. This plays a significant role in the case of LUMI.

## Containers on LUMI

The motivation for using containers on LUMI is twofold: 

- compatibility with ROCm (GPU runtime) and Slingshot network (inter-node communication), 
- filesystem friendliness (encapsulation helps reduce the overhead on the filesystem from accessing numerous small files).

Note that the first point implies that LUMI's containers are **not portable to other machines**, which is usually expected from containers, as these images are unlikely to run on other systems.

There are two different ways that containers provided by the LUMI User Support Team can be accessed:

- [Through modules and wrapper scripts generated via EasyBuild](https://lumi-supercomputer.github.io/LUMI-EasyBuild-docs/p/PyTorch/#module-and-wrapper-scripts)
- Directly, with you taking care of all bindings and all necessary environment variables.

In this example we will use the second option as this approach is used in the [LUMI AI workshop material](https://github.com/Lumi-supercomputer/Getting_Started_with_AI_workshop).

The latest versions of the provided containers can be found at `/appl/local/containers/sif-images`. This folder includes base containers, following the naming convention `lumi-rocm-<rocm version number>.sif` and containers that already include an ML framework and some commonly used packages. The names of the `.sif` files indicate which ML framework is installed and which versions are used for `ROCm`, `Python` and the framework. 

If you choose one of the containers in `/appl/local/containers/sif-images/` for your own project, we recommend copying the container to your working directory, as the containers are constantly updated. Newer containers might not be compatible with your setup.

## Interacting with a containerized environment

The Python environment from an image can be accessed either interactively by spawning a shell instance within a container (`singularity shell` command) or by executing commands within a container (`singularity exec` command). Do not expect there to be premade runscripts (`singularity run` command) within the container; you need to execute your own script inside the container.

These commonly used assumptions are good to keep in mind:

- most base images on LUMI use conda (Miniconda) environments that need to be activated with the `$WITH_CONDA` command,
- there is a basic compiler toolchain included; note specific compiler commands (`gcc-XX` for specific versions installed).

To inspect which specific packages are included in the images you can use this simple command:

```
export SIF=/appl/local/containers/sif-images/lumi-pytorch-rocm-6.2.1-python-3.12-pytorch-20240918-vllm-4075b35.sif
singularity exec $SIF bash -c '$WITH_CONDA && pip list'
``` 

## Singularity and Slurm

To run a program inside a container on a GPU node, you need to prepend the singularity command with the `srun` launcher. Please note that multiple srun tasks will spawn independent instances of the same container image. 

We can check whether the selected PyTorch image detects the allocated GPUs with the following: 

The command

```
salloc -p small-g --nodes=1 --gpus-per-node=2 --ntasks-per-node=1 --cpus-per-task=14 --time=3 \
    --account=project_xxxxxxxxx #specify your project id here
```

allocates 2 GPUs and 14 CPUs from a single compute node for a 3-minute job. We can then print out the number of detected GPUs via the following command:

```
export SIF=/appl/local/containers/sif-images/lumi-pytorch-rocm-6.0.3-python-3.12-pytorch-v2.3.1.sif
srun singularity exec $SIF \
    bash -c '$WITH_CONDA ; \
             python -c "import torch; print(torch.cuda.device_count())"'
```

For more information on SLURM on LUMI, please visit the [SLURM quickstart page in our documentation](https://docs.lumi-supercomputer.eu/runjobs/scheduled-jobs/slurm-quickstart/).

## `singularity-AI-bindings` module

To give LUMI containers access to the Slingshot network for good RCCL and MPI performance and access to the file system of the working directory, some additional bindings are required. As it can be quite cumbersome to set these bindings manually, we provide a module that does this for you. You can load the module with the following commands:

```
module use /appl/local/containers/ai-modules
module load singularity-AI-bindings
```

If you prefer to set the bindings manually, we recommend taking a look at the [Running containers on LUMI](https://lumi-supercomputer.github.io/LUMI-training-materials/ai-20240529/extra_05_RunningContainers/) lecture from the [LUMI AI workshop material](https://github.com/Lumi-supercomputer/Getting_Started_with_AI_workshop).

## Installing additional Python packages in a container 

You might find yourself in a situation where none of the provided containers contain all Python packages you need. One possible way of adding custom packages not included in the image is to use a virtual environment on top of the conda environment. For this example, we need to add the HDF5 Python package `h5py` to the environment:

```
module use /appl/local/containers/ai-modules
module load singularity-AI-bindings
export SIF=/appl/local/containers/sif-images/lumi-pytorch-rocm-6.2.1-python-3.12-pytorch-20240918-vllm-4075b35.sif
singularity shell $SIF
Singularity> $WITH_CONDA
(pytorch) Singularity> python -m venv h5-env --system-site-packages
(pytorch) Singularity> source h5-env/bin/activate
(h5-env) (pytorch) Singularity> pip install h5py
```

This will create an `h5-env` environment in the working directory. The `--system-site-packages` flag gives the virtual environment access to the packages from the container. Now one can execute a script with and import the `h5py` package. To execute a script called `my-script.py` within the container using the virtual environment, use the additional activation command:

```
singularity exec $SIF bash -c '$WITH_CONDA && source h5-env/bin/activate && python my-script.py'
```

This approach allows extending the environment without rebuilding the container from scratch every time a new package is added. The drawback is that the virtual environment is disjoint from the container, which makes it difficult to move as the path to the virtual environment needs to be updated accordingly. Moreover, installing Python packages typically creates thousands of small files. This puts a lot of strain on the Lustre file system and might exceed your file quota. This problem can be solved by creating a new container using the [cotainr tool](https://lumi-supercomputer.github.io/LUMI-training-materials/ai-20241126/extra_06_BuildingContainers/) or turning the virtual environment directory into a [SquashFS file](https://github.com/Lumi-supercomputer/Getting_Started_with_AI_workshop/blob/main/07_Extending_containers_with_virtual_environments_for_faster_testing/examples/extending_containers_with_venv.md). The examples included in this repository use the [SquashFS](https://github.com/Lumi-supercomputer/Getting_Started_with_AI_workshop/blob/main/07_Extending_containers_with_virtual_environments_for_faster_testing/examples/extending_containers_with_venv.md) option.

## Custom images

In theory, you can also bring your own container images or convert images from other registries (DockerHub for instance) to the singularity format. In this case it remains your responsibility to keep the container compatible with LUMI's hardware and system environment. We strongly recommend building your containers on top of the LUMI base images provided. 

### Table of contents

- [Home](..#readme)
- [1. QuickStart](https://github.com/Lumi-supercomputer/LUMI-AI-Guide/tree/main/1-quickstart#readme)
- [2. Setting up your own environment](https://github.com/Lumi-supercomputer/LUMI-AI-Guide/tree/main/2-setting-up-environment#readme)
- [3. File formats for training data](https://github.com/Lumi-supercomputer/LUMI-AI-Guide/tree/main/3-file-formats#readme)
- [4. Data Storage Options](https://github.com/Lumi-supercomputer/LUMI-AI-Guide/tree/main/4-data-storage#readme)
- [5. Multi-GPU and Multi-Node Training](https://github.com/Lumi-supercomputer/LUMI-AI-Guide/tree/main/5-multi-gpu-and-node#readme)
- [6. Monitoring and Profiling jobs](https://github.com/Lumi-supercomputer/LUMI-AI-Guide/tree/main/6-monitoring-and-profiling#readme)
- [7. TensorBoard visualization](https://github.com/Lumi-supercomputer/LUMI-AI-Guide/tree/main/7-TensorBoard-visualization#readme)
- [8. MLflow visualization](https://github.com/Lumi-supercomputer/LUMI-AI-Guide/tree/main/8-MLflow-visualization#readme)
- [9. Wandb visualization](https://github.com/Lumi-supercomputer/LUMI-AI-Guide/tree/main/9-Wandb-visualization#readme)
