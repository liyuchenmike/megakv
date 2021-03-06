Mega-KV is a high-throughput in-memory key-value store (cache) which adopts a
novel approach by offloading index data structure and corresponding operations
to GPU.

Mega-KV is currently implemented above NVIDIA CUDA APIs and Intel DPDK on Linux,
but it can be ported to other GPGPU programming frameworks, such as OpenCL, and
operating systems as well.


## GETTING STARTED

If you intend to run Mega-KV on AWS `p2.xlarge` instances using the AMIs 
listed on [Deep Learning AMI CUDA 9 Ubuntu Version][aws-deep-learning-cuda-9], 
the script in [`bin/setup.sh`](bin/setup.sh) may work for you, 
if you have a different environment to set up or 
wish to understand better what is going on here, 
please follow the USAGE instructions below.

### AWS P2 instance type results

We were able to rent an AWS `p2.xlarge` instance at the almost 
[too cheap to meter][too-cheap-to-meter] AWS [spot price][spot-price] 
of $0.1301 per hour, many many times cheaper than purchasing the equivalent
CPUs, GPUs, motherboard, RAM, PSU, case/rack and other system components.
This fulfils the standard dramatically lower capex outlay promise of the cloud.

#### MegaKV Tesla K80 GPU utilisation out of the box was between 1-5%

Using [`htop`][htop] and [`nvidia-smi`][nvidia-smi] combined with the 
Mega-KV `src` (see USAGE Steps 4-6) we found for the default workload, 
that both `insert` and `search` performance was still CPU-bound, with 
GPU utilisation at 1-2% and 4-5% for the  `insert` and `search` 
phases respectively.

Future work could look into why this is the case and investigate 
how to attain a higher utilisation of the available GPU resource, such as 
offloading more of the CPU-bound work onto the GPU itself, other parts of 
the system such as say the networking stack, or even other AWS instances.

#### Example improvement - Modifying `NUM_QUEUE_PER_PORT` and `MAX_WORKER_NUM`

We found that modifying the values `NUM_QUEUE_PER_PORT` and `MAX_WORKER_NUM`
in [macros.h][macros-h-opt] from 7 and 12 respectively to 1 and 1 improved
out of the box MegaKV insert phase throughput on our `p2.xlarge` instance 
from ~0.5 to ~2 Mops, and also improved search phase performance 
from ~8 to ~18 Mops, a ~4x and ~2x improvement respectively 
for our machine.

#### The benchmark `rte_eth_dev_count()` returns 0

This most likely means DPDK-enabled network interfaces are not available on P2 
instances, [only X1 instances, at the time of writing][aws-dpdk-ena-x1-only].

Future work could wait for a [P2, P3 or CG1 CUDA-enabled][aws-cuda-instances] 
instance to also have DPDK / ENA support, or as Kai Zhang, et al suggested
earlier in this README, consider ports to other GPGPU programming frameworks,
such as OpenCL, or support for other operating systems as well.


## HISTORY

1. Jun 1, 2015: megakv-0.1-alpha. Initial release; basic interfaces for an in
memory key-value store. This is a demo and is not ready for production use yet.
Bugs are expected.
2. Nov 1, 2017: For [MongoDB Skunkworks][mongodb-skunkworks] - 
Updates to run on AWS `p2.xlarge` instances, Intel DPDK v16.11, CUDA 9 
and Ubuntu gcc 5.4.0


## PROTOCOL

Mega-KV currently uses a simple self-defined protocol for efficient communication.

* A request packet has a 16-bit magic number in the beginning: 0x1234.
* A request packet has a 16-bit ending mark in the end: 0xFFFF.
* Each GET query in the packet has the format: 16-bit Job Type(0x2), 16-bit Key
Length, and the key.
* Each SET query in the packet has the format: 16-bit Job Type(0x3), 16-bit Key
Length, 32-bit Value Length, and the key and value.

Anyone can improve or modify this protocol according to the practical needs.


## HARDWARE

* NIC: Intel 10 Gigabit NIC that is supported by Intel DPDK SDK.
* CPU: Intel CPU that supports the SSE instruction set in Intel DPDK SDK.
* GPU: NVIDIA GPU newer than GTX680. We have conducted experiments on GTX780.


## USAGE

1. Setup network with Intel DPDK. We recommend installing Intel DPDK 1.7.1,
which is known to work with Mega-KV. Newer versions of DPDK may have some
compiling problems with Mega-KV. Then run `export RTE_SDK=$(PATH_TO_DPDK)`.
`PATH_TO_DPDK` is the path of the DPDK directory.


2. Go to `libgpuhash` directory, edit `Makefile` to setup correct CUDA installation
path. We recommend installing CUDA SDK 6.5, which is known to work with Mega-KV.

    Some important macros in `gpu_hash.h`:

    * MEM_P: 2^MEM_P bytes GPU device memory space for hash table.
    * HASH_CUCKOO/HASH_2CHOICE: cuckoo hash or two choice hash.


3. Run `make`. This should compile the CUDA hash table library, including cuckoo
hash or two choice hash. Macros can be set in `gpu_hash.h`. This will generate
`libgpuhash.a` in lib directory, which is used by Mega-KV as the GPU hash table
library.


4. Go to `src` directory, edit `Makefile` to setup correct CUDA installation path.
Setup other macros in `Makefile` and `macros.h` for test or production use. Edit the
config variables in `mega.c` for different GPUs or configurations.

    In the `Makefile`, a macro is disabled with the `_0` suffix. You can enable the
macro by removing the suffix.

    Some important macros in `Makefile`:

    * PREFETCH_BATCH: enable batch prefetching to improve performance.
    * PRELOAD: preload key/value items into Mega-KV before test.
    * LOCAL_TEST: run Mega-KV locally, just for testing.
    * SIGNATURE: enable a simple signature algorithm instead the one used for testing.
      You can implement a new signature algorithm under this macro.

    Some important macros in `macros.h`:

    * CPU_FREQUENCY_US: set the CPU frequency for the timers.
    * MEM_LIMIT: set the memory limit to avoid using virtual memory.
    * NUM_QUEUE_PER_PORT: number of queues per NIC port. Each queue will have one
      receiver and one sender.


5. Edit the CPU core mappings in `mega.c`. Three functions for launching Receivers,
Senders, and the Scheduler: `mega_launch_receivers`, `mega_launch_senders`, and
`mega_launch_scheduler`. You can edit `context->core_id` assignment to change the core
mapping for these threads.

    To maximize the resource utilization and system utilization, Hyper-threading is
recommended. The Nth Receiver and the Nth Sender can be assignment to two virtual
cores that locate on the same physical core. Please note that one physical core
should be reserved for the Scheduler so that it will not be affected by other
threads.

    Corresponding DPDK parameters may also need to be modified in line 527.


6. Run `make`. This should compile Mega-KV. Then Mega-KV can be run with
`./build/megakv`

    The above currently defaults to `insert` jobs for about a minute, prints
    `==========================     Hash table has been loaded     ==========================`
    and then switches to `search` jobs, periodically reporting 
    statistics to the terminal. 


7. Benchmark.

    Go to `benchmark` directory. This is also based on Intel DPDK 1.7.1. Modify macros in
    `benchmark.h`, and modify CPU core mappings between the line 792 and the line 815.

    Run `make`, then run `sudo ./build/benchmark`

    This benchmark currently only support for 8 byte key and 8 byte value generation.
    NOTE: LOAD_FACTOR, PRELOAD_CNT, and TOTAL_CNT should be the same with Mega-KV if
    Mega-KV preloads key-value items locally for testing.

    Some important macros in `benchmark.h`:

    * DIS_ZIPF/DIS_UNIFORM: key popularity distribution.
    * WORKLOAD_ID: 100% GET or 95% GET


## PERFORMANCE BOTTLENECKS

It should be possible to run the following Linux system utility programs
to identify the system's performance bottlenecks:

 1. CPU/RAM bottlenecks - [`top`][top] or [`htop`][htop]
 2. GPU bottlenecks - [`nvidia-smi`][nvidia-smi]

There may also be a need for additional specific tools to investigate 
performance bottlenecks, for a brief overview please see 
[this AskUbuntu][ask-ubuntu-performance].
 

## LIMITATIONS

1. Do not support UPDATE command yet.
2. Do not support other fields in memcached, such as expiration time. However, they
are easy to be implemented and have been planed in the roadmap.
3. LOCAL_TEST may not be accurate. Because the overhead of key generation is very
huge, especially with zipf key generation.


## DEVELOPMENT

Go to [http://kay21s.github.io/megakv](http://kay21s.github.io/megakv) for documentation and other
development notices. You can contact the author at `kay21s [AT] gmail [DOT] com`.


## DISCLAIMER

This software is not supported by [MongoDB, Inc.](https://www.mongodb.com)
under any of their commercial support subscriptions or otherwise. Any usage of
Mega-KV is at your own risk. Bug reports, feature requests and questions can be
posted in the [Issues](https://github.com/pzrq/megakv/issues?state=open) section on GitHub.


[mongodb-skunkworks]: https://www.mongodb.com/careers/departments/engineering
[aws-deep-learning-cuda-9]: https://aws.amazon.com/marketplace/pp/B076TGJHY1
[too-cheap-to-meter]: https://en.wikipedia.org/wiki/Too_cheap_to_meter
[spot-price]: https://aws.amazon.com/ec2/spot/pricing/
[top]: https://linux.die.net/man/1/top
[htop]: https://linux.die.net/man/1/htop
[nvidia-smi]: https://developer.nvidia.com/nvidia-system-management-interface
[macros-h-opt]: https://github.com/pzrq/megakv/blob/308713f364d4cb66f690722759d3d278a392ee8f/src/macros.h#L29-L31
[aws-dpdk-ena-x1-only]: https://aws.amazon.com/blogs/aws/elastic-network-adapter-high-performance-network-interface-for-amazon-ec2/
[aws-cuda-instances]: http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/accelerated-computing-instances.html
[ask-ubuntu-performance]: https://askubuntu.com/questions/1540/how-can-i-find-out-if-a-process-is-cpu-memory-or-disk-bound
