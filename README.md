GPU management through REDIS brpop and lpush
============================================

When running my programs in CUDA or OpenCL I often approach servers with multiple GPU cards. The problem is that the server has usually multiple users and I need to manage the GPU resources between users and also between my own programs. Imagine I run multiple tasks of e.g. CT reconstruction by means of [KCT_cbct](https://github.com/kulvait/KCT_cbct) on computer with multiple GPUs. I can have e.g. 150 tasks and want to distribute them over available GPUs in the way that at the same time at single GPU runs only one task.

# The solution

The solution is to use REDIS server to manage the GPU resources. The idea is to have a list of available idle GPUs and a list of GPUs which are managed. When a GPU is requested, the server pops the first available GPU and returns its ID. Then program can use the GPU with given ID in the same way, user can call `kct-krylov -p 0:$GPUID` to run the program on the GPU with given ID. When the GPU is released, the server pushes its ID back to the list of available GPUs.

This ways all the users must agree to use this solution to manage the GPU resources. The program must be modified to use the GPU with given ID. The program must also release the GPU when it is no longer needed. Program itself does not manage GPUs in the way of enforcing their actuall usage. It relies on the willinness of the users to use it.

When there is no GPU available, the program waits for the GPU to be released. This is done by means of `brpop` command in REDIS. The program waits for the GPU to be released and then pops the GPU from the list of available GPUs.

# Prerequisites

This implementation in python requires the following packages to be present in your system.
```
pip install redis
pip install argparse
pip install pycuda
```
Moreover it requires running instance of REDIS server. In Debian you can install it by means of
```
sudo apt-get install redis-server
```

# Usage

First start the REDIS server.

Put the `GPU.py` into your path.

To initialize all the objects in REDIS server run
```
GPU.py --redis-manage-all
```
When something is broken and you wish to reset objects run just
```
GPU.py --redis-manage-all --force
```

Now to get ID of the IDLE GPU run
```
GPU.py --get
```
as the program manages parent PID of the process, in Bash script the good way to get the GPU ID is to run
```
read GPUID < <(GPU.py --get)
```

To release the GPU with given ID run
```
GPU.py --release $GPUID
```

To release GPU IDs which were allocated by the processes with the PIDs, that no longer exist run
```
GPU.py ---redis-purge
```
This is potentially dangerous as e.g. when the `GPUID=$(GPU.py --get)` is called, the system stores PID of the subshell that no longer exist and `--redis-purge` might then return GPUID to the list of idle GPUs even if it is not idle. This is not a problem when the GPU is released by the same process that allocated it. Intended just as a soft solution of inconsistencies.

For mor advanced use try
```
GPU.py --help
```
or consult the source code.

To get overview of the allocations run
```
GPU.py --log
```

To list managed GPUs and IDLE GPUs run
```
GPU.py --managed
GPU.py --idle
GPU.py --info
```


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

If you find this software useful, you can support its development by means of small donation.

[![Thank you](https://img.shields.io/badge/donate-$15-blue.svg)](https://kulvait.github.io/donate/?amount=15&currency=USD)
