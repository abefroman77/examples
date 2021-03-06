# Introduction

Here are some brief instructions on how to build an NVIDIA GPU Accelerated Supercomputer using Jetson boards like the Jetson Nano and the Jetson Xavier.
These instructions accompany the video TBD.

# Hardware
You will need at least two Jetson boards. The cheapest is the Jetson Nano 2GB at $59. You will need Ethernet (preferably Gigabit), and a hub/switch.
For the initial setup, you will also need a monitor, keyboard, and mouse. But these aren't needed once the cluster is configured and running.

https://developer.nvidia.com/embedded/jetson-nano-2gb-developer-kit

# Software
You need to be running the same version of the NVIDIA JetPack SDK on each board.
JetPack includes the OS (Linux, based on Ubuntu) along with CUDA-X accelerated libraries, samples, documentation, and developer tools.

https://developer.nvidia.com/embedded/jetpack


# Configuration
## User configuration
Make sure you use the same user account name across all the boards.

## Network
Each board should be in the same physical network, within the same subnet. The easiest option is to use fixed IP addresses like:

```
192.168.1.51
192.168.1.52
192.168.1.53
192.168.1.54
```
## Prerequisite software
Install the prerequisite software on **every** node
```
sudo apt-get update; sudo apt-get -y install openssh-server git htop python3-pip python-pip nano mpich mpi-default-dev
```

## Install jtop (optional)
```
sudo pip3 install jetson_stats
```

## Generate your ssh keys
Generate your ssh keys on the controller node. Run the command and follow the prompts.

```
ssh-keygen
```
Then, for every node in your cluster (Replace <IP-ADDRESS> with IP of each node):
```
ssh-copy-id <IP-ADDRESS>
```
e.g. `ssh-copy-id 192.168.1.51`

## Cluster file
Create a cluster file on the controller node. One line per cluster.
```
192.168.1.51
192.168.1.52
etc
```
You can do this in an editor like vim or nano, or use a command like this:
```
cat > clusterfile << _EOF_
192.168.1.51
192.168.1.52
192.168.1.53
192.168.1.54
_EOF_
```

You might want to copy this file into the simpleMPI directory later. See below.

## /etc/hosts
On each node make sure all other nodes are listed in /etc/hosts with their right hostnames, e.g.
```
192.168.1.51   node1
192.168.1.52   node2
192.168.1.53   node3
192.168.1.54   node4
```

# The MPI & CUDA program
The video demonstrates running a CUDA aware workload on each node in the cluster. Here is a good overview of how that works: https://developer.nvidia.com/blog/introduction-cuda-aware-mpi/

The program is based on the simpleMPI example from NVIDIA:

https://docs.nvidia.com/cuda/cuda-samples/index.html#simplempi

You can find a copy in `/usr/local/cuda/samples/0_Simple/simpleMPI/` on your Jetson board.

Copy the `simpleMPI` directory and its contents to somewhere in your home directory. Edit the make file (`Makefile`) and remove these two lines to stop the binary from being copied to a common area in the samples directory. You don't need this now that you have copied the code to a local directory.

```
$(EXEC) mkdir -p ../../bin/$(TARGET_ARCH)/$(TARGET_OS)/$(BUILD_TYPE)
$(EXEC) cp $@ ../../bin/$(TARGET_ARCH)/$(TARGET_OS)/$(BUILD_TYPE)
```

To make the workload heavier on the GPU, change `simpleMPIKernel()` in `simpleMPI.cu` to:

```
__global__ void simpleMPIKernel(float *input, float *output)
{
    int i;
    int tid = blockIdx.x * blockDim.x + threadIdx.x;
    output[tid] = input[tid];
    for(i=0;i<50000;i++) {
        output[tid] = sqrt(output[tid]);
        output[tid] = output[tid] * output[tid];
    }
}
```

The new code basically adds a loop of 50000 where the square root is taken and then the result doubled each time.

Edit `simpleMPI.cpp` and alter `int blockSize = 256;` to:

```
int blockSize = 384 / commSize;
```

This change means that each node will get sent less data, depending on the number of nodes (which is MPI terms in held in the variable `commSize`).

Run `make` to build the binary. A copy of the binary `simpleMPI` must be present on each node with the same path and filename.

## To run it on the cluster
Use this command on the controller node.
```
mpiexec --hostfile clusterfile ./simpleMPI
```
