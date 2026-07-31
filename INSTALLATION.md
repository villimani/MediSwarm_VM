# Installation of the cifar10 example on ARM64 architecture

## Initial Steps
### Docker
*`docker-desktop` does not run on this architecture.*
- Install Docker following the process [here](https://docs.docker.com/engine/install/ubuntu/).
- Add your user to the docker group so you don't have to `sudo` to run any docker commands:
    ```bash
    sudo usermod -aG docker $USER
    ```
- Restart your system.

### MediSwarm
- If you haven't cloned MediSwarm yet, clone it with recurse:
    ```bash
    git clone https://github.com/KatherLab/MediSwarm.git --recurse-submodules
    ```
- Otherwise run the following from within the MediSwarm dir:
    ```bash
    git submodule update --init --recursive
    ```
- This will add the NVFlare submodules in the NVFlare directory under `docker_config`.

### Dockerfile changes
- From the [NGC Catalog](https://catalog.ngc.nvidia.com/orgs/nvidia/-/containers/pytorch/-/tags) select the pytorch/cuda images and set it in [docker_config/Dockerfile_cifar10](docker_config/Dockerfile_cifar10). For PyTorch 2.2.0 as expected in this implementation, use the following:
    ```
    ARG PYTORCH_IMAGE=nvcr.io/nvidia/pytorch:23.11-py3-igpu
    ```
- Comment out line 18 `python3 -m pip install torchvision==0.14.0` as the autoinstalled version of torchvision is newer than 0.14.0.
- Add a new line to install tensorboard 2.16.2 `RUN python3 -m pip install tensorboard==2.16.2`.
- Comment out the lines (29, 33, 34) related to installing `NVFlare/dashboard` and `../controller`.

## Containers
- Build the image:
    ```bash
    cd ./Mediswarm
    docker build -t nvflare-pt-dev:cifar10 . -f docker_config/Dockerfile_cifar10
    ```

- Start the Docker container:
    ```bash
    docker run -it --rm \
        --shm-size=16g \
        --ipc=host \
        --ulimit memlock=-1 \
        --ulimit stack=67108864 \
        -v ./docker_config/NVFlare:/workspace/nvflare \
        --gpus=all \
        -v ./:/workspace \
        --name=testrun \ #assign a name for simple start/stop
        -p 6006:6006 \ #listen to this port for tensorboard
        nvflare-pt-dev:cifar10
    ```

- \[OPTIONAL\] If you've already downloaded the cifar10 tarball, you can copy it into the container using `docker cp /path/to/cifar-10-python.tar.gz [CONTAINER_NAME]:/path/to/dir/`
- Perform data prep:
    ```bash
    mkdir /tmp/cifar10 && mv /tmp/cifar-10-python.tar.gz /tmp/cifar10/
    ./application/jobs/cifar10/prepare_data.sh
    ```
- need to figure out how to do the data split without needing to run the simulation first (as shown in the [cifar10 readme](application/jobs/cifar10/README.md)

- To get tensorboard to show training progress (only in POC), run:
    ```bash
    tensorboard --logdir /tmp/nvflare/poc/example_project/prod_00/
    ```
- Stop all running POCs using `nvflare poc stop`.

- Clear all POCs in `/tmp` using `nvflare poc clean`. (THIS ALSO DELETES ALL THE LOGS)
