# Xilinx Dockers for GitHub Actions
The goal of this repository is to document the process of creating Vivado and/or Vitis Docker images for GitHub Action Runners. It also contains some example `Dockerfile` for you to use.

## What, Why, and How
### What is CI/CD?
In short, Continuous Integration and Continuous Delivery is an automation process that helps developers to automatically test, build, and release their code. 

For FPGA developers, we spend a lot of time waiting for the bitstream generation. For me, sometimes I would like to perform an extra "clean run" before I release the bit file for other colleagues to use. This process may take a while, and it will keep myself occupied. With CI/CD workflow, we can automate not only the bitstream generation process, but also testbench runs, firmware compile, and more. If we can utilize CI/CD for such tasks, we can devote more of our time into actual HDL developments.

Please refer to [this](https://resources.github.com/ci-cd/) from GitHub article to better understand CI/CD.

### Why GitHub Actions?
While existing explorations on FPGA CI/CD workflow exists, they usually requires a self-managed machine to achieve this. For example, [this article by VHDLwhiz](https://vhdlwhiz.com/jenkins-for-fpga/) use Jenkins on a Linux VPS to achieve a similar goal. However, such solutions needs one to devote their own resource for such automation.

With [GitHub Actions](https://github.com/features/actions), one can use GitHub-hosted Action Runners to perform such task. In addition, GitHub Action Runners are free for at least 2,000 minutes per month, and completely free for public repositories. This provides Open Source FPGA developers and hobbyists another free option to develop, test, and deliver their code.

### Challenges
Xilinx tools have long been criticized for their enoumous size, and they are not installed on GitHub Action Runners by default. Hence, using Xilinx tools in GitHub Actions can be a headache. We can not install Xilinx tools directly to the GitHub Action Runners due to its large file size and long installation time, not to mention the legal risks involved regards to licensing issues.

A common approach is to use [Docker](https://docs.docker.com/get-started/overview/). We can install Vivado and/or Vitis to Docker image(s) ahead of time, and let GitHub Action Runners to load Docker image(s) and perform the task for us.

While creating Docker images for Vivado and Vitis is quite common, using such Docker images with GitHub Actions has its own challenges. GitHub Action Runners usally gives you around 20GB of usable space, which is tiny compare to the size of Xilinx tools. While Actions like [`maximize-build-space`](https://github.com/easimon/maximize-build-space) exists, we still need to make efforts to minimize the Vivado and Vitis image size so that we can use them within GitHub Action Runners.

This tutorial helps you setup Docker on your own machine, create Vivado/Vitis Docker images with your own credentials, and help you use these images with GitHub Action Runners.

## Prerequisites
Since we want to run Vivado and Vitis in GitHub Actions, the Linux version of them is a more suitable choice. Hence, I build these images under Linux host or VM.

### Linux
- an architecture that supports Xilinx tools. `amd64` or `x86_64` for now.
- at least 200GB of free space.
- a recent distro. Personally I use Xubuntu 22.04.
  
### Windows & macOS
I know it is also possible to create Linux Docker image under Windows and macOS, but I do not have any experience building Docker this way, and I recommend you installing a Linux VM before continue. Personally I use VMWare Workstation Pro 16 on Windows to create VMs, if you are UIUC student you can find free license key in the webstore. For Apple Silicon Macs, I do not have a good solution as of now.

## Install Docker Engine
If you already have Docker installed on your Linux machine or VM, you can skip to the next section, [Prepare Xilinx Installer](#prepare-xilinx-installer).

Please follow the installation guide on Docker website. For Ubuntu, you can find it [here](https://docs.docker.com/engine/install/ubuntu/). Select other distros on the left. A summary of commands for Ubuntu is shown below for your convenience.

1. Update the apt package index and install packages to allow apt to use a repository over HTTPS:
    ```shell
    $ sudo apt-get update
    $ sudo apt-get install ca-certificates curl gnupg lsb-release
    ```

2. Add Docker’s official GPG key:
    ```shell
    $ sudo mkdir -p /etc/apt/keyrings
    $ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    ```

3. Set up the repository:
    ```shell
    $ echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    ```

4. Update the apt package index, and install the latest version of Docker Engine, containerd, and Docker Compose:
    ```shell
    $ sudo apt-get update
    $ sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
    ```

5. Verify that Docker Engine is installed correctly by running the `hello-world` image.
    ```shell
    $ sudo docker run hello-world
    ```

Below we follow the [Post-installation steps for Linux](https://docs.docker.com/engine/install/linux-postinstall/) guide so that we do not need `sudo docker` everytime we use it.

6. Create the `docker` group.
    ```shell
    $ sudo groupadd docker
    ```

7. Add your user to the `docker` group.
    ```shell
    $ sudo usermod -aG docker $USER
    ```

8. **Log out and log back in** so that your group membership is re-evaluated. If testing on a virtual machine, it may be necessary to **restart** the virtual machine for changes to take effect.
   
9.  Verify that you can run `docker` commands without `sudo`.
    ```shell
    $ docker run hello-world
    ```

## Prepare Xilinx Installer
We will be using Xilinx's batch installation mode to install Vivado and Vitis. At the time of writing this, the latest Vitis/Vivado version is 2022.1, so the rest of the tutorial will be referencing this version.

1. First, download the `Xilinx Unified Installer 2022.1: Linux Self Extracting Web Installer` from the [Xilinx Download Center](https://www.xilinx.com/support/download.html). This unified installer, as its name suggests, can install both Vivado and Vitis. You will need Xilinx account to download this. 
   
2. Put the [`Dockerfile`](2022.1/Dockerfile) and the installer you just downloaded under the same directory.
   
3. Change permission for the installer.
   ```shell
   $ sudo chmod +x ./Xilinx_Unified_2022.1_0420_0327_Lin64.bin
   ```

4. Generate the authentication token for installation. You will be asked to enter your Xilinx account email address and password.
   ```shell
   $ ./Xilinx_Unified_2022.1_0420_0327_Lin64.bin -- --batch AuthTokenGen
   ```

5. Copy the generated authentication token to the same folder as the `Dockerfile` and the Xilinx installer. Note that you can reuse this authentication token. You can check its content to see its expiration date.
   ```shell
   $ cp ~/.Xilinx/wi_authentication_key .
   ```

6. Generate the configuration file for installation. You will be asked to select what product you want to install. If you use Vitis and Vivado, select `Vitis`. If you only use Vivado, select `Vivado`, then select `Vivado ML Standard`. 
   ```shell
   $ ./Xilinx_Unified_2022.1_0420_0327_Lin64.bin -- --batch ConfigGen
   ```
   Alternatively, you can find a configuration file I created [here](2022.1/install_config.txt). I install both Vitis and Vivado, and only selected Artix-7 devices.

7. Copy the configuration file to the same folder. If you just generated the configuration with `ConfigGen`, do the following:
    ```shell
    $ cp ~/.Xilinx/install_config.txt .
    ```

8. Modify the configuration file as necessary.

If you have any quesitons, please check the [Vivado Design Suite User Guide (UG973)](https://docs.xilinx.com/r/en-US/ug973-vivado-release-notes-install-license/Batch-Mode-Installation-Flow).

## Create Docker Image(s)
Now we can create Docker image(s). The strategy here is to first install Vivado and/or Vitis in one image, then copy the installed files into new image(s). This is known as [multi-stage builds](https://docs.docker.com/develop/develop-images/multistage-build/), and it can help us save more space on our final image(s).
If you use both Vivado and Vitis, you will need to create two images. When you use these images with GitHub Actions, you do the Vivado work with Vivado image first, store the output files (i.e. `.xsa` files), remove the Vivado image, then load the Vitis image and do the rest of the work.

1. Be sure you have the following files under the same directory:
   ```
   Dockerfile
   Xilinx_Unified_2022.1_0420_0327_Lin64.bin
   install_config.txt
   wi_authentication_key
   ```

2. We first build the `installation` image with tag `2022.1`. 
   
   > Note this can take a while as the Xilinx installer will download files during this stage. The CPU utilization will also be quite high while the installer uncompress downloaded files. If you are running this under a Linux VM, please be aware and plan your workload accordingly.

   > There will be a long wait after the installation is complete. This is normal, do not abort installation.
   
   ```shell
   $ docker build --target installation -t installation:2022.1 .
   ```

3. After build finish, you can check the image created with:
   ```shell
   $ docker image ls
   ```

   Or if you want to get a more detailed look on the size etc., you can use:
   ```shell
   $ docker system df -v
   ```

   With either method, you should be able to see `installation` under `REPOSITORY`, and `2022.1` under `TAG`.

4. Now you can build Vivado and/or Vitis image(s).
   ```shell
   $ docker build --target vivado -t vivado:2022.1 .
   $ docker build --target vitis -t vitis:2022.1 .
   ```

5. Now you can safely delete the `installation` image.
   ```shell
   $ docker image rm installation:2022.1
   ```


```shell
$ docker stop <ID>
```

```shell
$ docker rm -f <ID>
```

## Limitations and Future Works
### Limitations
1. Personally I have only been using Artix-7 with MicroBlaze, ZYNQ-7000 in Bare Metal mode, and limited HLS works. Hence, I do not have much experience with Petalinux, and did not test it with the method mentioned above. Hoever, in theory they should work just fine. If you had any success, please feel free to create PR or issue to let me know.
   
2. There could be legal risks regards to installing Xilinx tools into Docker images and use them on GitHub-hosted runners. This tutorials serves as information sharing only.

### Future Works
1. Create shell scripts to further automate the Docker creation process mentioned above

## References
1. [Dockerizing Xilinx tools on r/FPGA](https://www.reddit.com/r/FPGA/comments/bk8b3n/dockerizing_xilinx_tools/?utm_source=share&utm_medium=web2x&context=3)
2. [wunderabt/arty-risc-v](https://github.com/wunderabt/arty-risc-v/blob/38d3efa4cfcf090bb3d6e8f455cfcea760642bc4/docker/xilinx/Dockerfile)
3. [misodengaku/vivado-docker](https://github.com/misodengaku/vivado-docker/blob/master/ubuntu/18.04/vivado-petalinux/2020.1/Dockerfile)
4. [phwl/docker-vivado](https://github.com/phwl/docker-vivado/tree/master/2020.1)
5. [yeasy/docker_practice](https://github.com/yeasy/docker_practice)