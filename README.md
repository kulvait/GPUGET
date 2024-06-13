System wide GPU resource management for CUDA and OpenCL computations
====================================================================

Managing GPU resources on servers with multiple GPU cards can be challenging, especially when these cards are intended to run CUDA or OpenCL computational tasks. Typically, each process can utilize only a single GPU card at a time. Often, only the first card is utilized while others remain idle, leading to inefficient resource use. Additionally, negotiating GPU usage among multiple users and indicating the intention to use or wait for a GPU can be problematic.

The `GPUGET` system provides a solution for system-wide notification about GPU resource availability and reservation. Importantly, this system can manage resources beyond just GPUs, relying on user compliance for reserving and releasing resources as needed.

# Key Features

- **Resource Management**: Manages not only GPUs but can be extended to other resources.

- **User Compliance**: The system itself does not enforce resource management but depends on users to reserve and release resources appropriately.

- **Redis Integration**: Utilizes Redis server with `brpop` and `lpush` commands to manage resource lists. Creates system-wide database objects for resource management so that multiple users can share it.

# How It Works

The `GPUGET` uses a Redis server to maintain lists of available and managed GPUs. When a GPU is requested, the system pops the first available GPU ID from the list. When the GPU is released, its ID is pushed back into the list. This process relies on user programs being modified to use and release GPUs based on their IDs.


## User Responsibilities

The system relies on users' willingness to follow the following management protocol.

- **Modify Programs**: Ensure programs are adapted to use GPUs with specific IDs.
- **Request GPUs**: Programs must request GPUs before they can use them. This is achieved by potentially blocking call `GPUGET --get`.
- **Release GPUs**: Programs must release GPUs when they are no longer needed. This is done by calling `GPUGET --release $GPUID`.

# Prerequisites

To use `GPUGET`, you need the following.


## Python packages
```
pip install redis
pip install argparse
pip install pycuda
```

## Redis server

Install Redis server on Debian-based systems
```
sudo apt-get install redis-server
```

# Usage

## Installation

- Clone the repository and put the `GPU.py` into your path.
- Start the Redis server.

### Initialization

Initialize objects in Redis database by running
```
GPU.py --redis-manage-all
```

To reset objects when something is broken run
```
GPU.py --redis-manage-all --force
```

## Get and Release GPU ID

To get the ID of an IDLE GPU
```
GPU.py --get
```
In Bash script obtain the GPU ID by calling
```
read GPUID < <(GPU.py --get)
```

To release the GPU with given ID
```
GPU.py --release $GPUID
```

## Manage inconsistencies

To release GPU IDs which were allocated by the processes that no longer exist:
```
GPU.py ---redis-purge
```
**Warning**: This command is intended to be used when the system is in inconsistent state. It releases all GPU IDs that were allocated by processes that no longer exist. It is not recommended to use it in normal operation as it might release GPU IDs that are still in use in case that the process that allocated them is not running anymore.

To completly reinitialize all objects and clear the logs
```
GPU.py --redis-manage-all --force
```

To remove all the objects from the Redis database
```
GPU.py --delete
```

## Advanced Usage

To list all available options run
```
GPU.py --help
```
or consult the source code.

## Log and Status

To get the overview of GPU allocations
```
GPU.py --log
```

To list currently idle GPUs
```
GPU.py --idle
```

To list currently active GPUs
```
GPU.py --active
```

# Example Usage

Consider running multiple tasks, such as CT reconstructions using [KCT_cbct](https://github.com/kulvait/KCT_cbct), on a computer with multiple GPUs. To distribute tasks effectively, each GPU should run only one task at a time. `GPUGET` helps achieve this by managing GPU IDs through Redis. The program `kct-krylov` has `-p $PLATFORMID:$GPUID` option to specify the GPU ID. The following Bash script demonstrates how to use `GPUGET` to manage GPU IDs.

```bash
#!/bin/bash
local INPUTPROJ=$1
local GEOMETRY=$2
local OUTPUTVOL=$3
PLATFORMID=0
read DEV < <(GPU.py --get)
kct-pb2d-krylov --force --cvp --cgls --barrier --relaxed -p $PLATFORMID:$DEV $INPUTPROJ $GEOMETRY $OUTPUTVOL
GPU.py --release $DEV

```

You can then call multiple instances of this script to run multiple tasks on multiple GPUs. The `GPUGET` system ensures that each GPU is used only once at a time.

# License

When there is no other licensing and/or copyright information in the source files of this project, the following apply for the source files:

Copyright (C) 2024 Vojtěch Kulvait

    This program is free software: you can redistribute it and/or modify
    it under the terms of the GNU General Public License as published by
    the Free Software Foundation, version 3 of the License.

    This program is distributed in the hope that it will be useful,
    but WITHOUT ANY WARRANTY; without even the implied warranty of
    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
    GNU General Public License for more details.

    You should have received a copy of the GNU General Public License
    along with this program.  If not, see <https://www.gnu.org/licenses/>.

# Donations

If you find this software useful, you can support its development through a donation.

[![Thank you](https://img.shields.io/badge/donate-$15-blue.svg)](https://kulvait.github.io/donate/?amount=15&currency=USD)
