# SPDK for Arm Morello

The 'morello' branch of this repository contains changes to SPDK release v23.01 to support native builds on the Arm Morello platform running CheriBSD.

This repository is not intended to be a permanent fork. It provides a convenient place to make work in progress visible to the community but the intent is upstream as much as possible.

[rtegrity](https://rtegrity.com/) is leading the ongoing development of the project, as part of the [Digital Security by Design](https://www.dsbd.tech/) Technology Access Programme.

## Prerequisites

The examples below assume the use of 'sudo' which can be installed and configured as 'root' using:

~~~{.sh}
pkg64c install sudo
visudo
~~~

To perform a native build on CheriBSD, the following packages are required:

~~~{.sh}
sudo pkg64c install -y bash git e2fsprogs-libuuid
sudo pkg64 install -y meson ninja python llvm llvm-base py39-pip gdb-cheri
sudo pkg64 install -y pkgconf gmake cunit openssl e2fsprogs-libuuid ncurses
sudo pkg64 install -y py39-pyelftools autoconf automake libtool help2man
sudo pkg64 install -y doxygen mscgen graphviz
pip install pyelftools
~~~

## Source Code

The patched SPDK source code can be obtained using:
~~~{.sh}
git clone https://github.com/spdk-morello/spdk
cd spdk
git checkout morello
git submodule update --init
~~~

## CheriBSD Source Code

To build the DPDK kernel modules, the source for CheriBSD needs to be installed in /usr/src:

~~~{.sh}
sudo git clone https://github.com/CTSRD-CHERI/cheribsd /usr/src
~~~

With CheriBSD 22.12, kernel modules fail to build with 'is ZFSTOP set?'.

As a temporary workaround, run the following after installing the source:

~~~{.sh}
echo 'ZFSTOP=${SYSDIR}/contrib/subrepo-openzfs' | sudo tee -a /usr/share/mk/bsd.sysdir.mk
~~~

Memory allocated with the contigmem kernel module does not support storing capabilities.
To work around this, the temporary hack in dpdk/kernel/freebsd/contigmem/cheribsd.patch can be
applied:

~~~{.sh}
sudo patch -d /usr/src < dpdk/kernel/freebsd/contigmem/cheribsd.patch
~~~

The kernel can then be rebuilt and installed following the instructions in
https://github.com/CTSRD-CHERI/cheripedia/wiki/HOWTO:-Build-CheriBSD-natively-on-Morello.

## Kernel Modules

DPDK kernel modules are required to manage the allocation of physically contiguous memory and for direct access to PCI devices.

Note that the modules need to match the type of kernel that is running (hybrid or purecap), which may be different to the type of the required SPDK libraries.

The modules can be installed in /boot/modules using:

~~~{.sh}
sudo cp dpdk/build/kmod/*.ko /boot/modules
sudo kldxref /boot/modules
~~~

The kernel modules need to be loaded with:

~~~{.sh}
sudo kldload contigmem nic_uio
~~~

To load them automatically after every boot, add the following line to /etc/rc.conf:

~~~{.sh}
kld_list="contigmem nic_uio"
~~~

## Hybrid Build

To create a hybrid build:

~~~{.sh}
export PKG_CONFIG_PATH=`pwd`/dpdk/config/hybrid
CC=clang CXX=clang++ ./configure --target-arch=armv8-a
gmake -j4
~~~

For a hybrid debug build:

~~~{.sh}
export PKG_CONFIG_PATH=`pwd`/dpdk/config/hybrid
CC=clang CXX=clang++ ./configure --target-arch=armv8-a --enable-debug
gmake -j4
~~~

## PureCap Build

There is no purecap version of CUnit which is a requirement for unit testing.
A temporary hacked build can be downloaded from:

https://github.com/spdk-morello/CUnit/releases/download/1.0/CUnit.tgz

This can be installed with:

~~~{.sh}
curl -L https://github.com/spdk-morello/CUnit/releases/download/1.0/CUnit.tgz --output /tmp/CUnit.tgz
sudo tar xv -C / -f /tmp/CUnit.tgz 
~~~

To create a purecap build:

~~~{.sh}
export PKG_CONFIG_PATH=`pwd`/dpdk/config/purecap
./configure --target-arch=morello
gmake -j4
~~~

For a purecap debug build:

~~~{.sh}
export PKG_CONFIG_PATH=`pwd`/dpdk/config/purecap
./configure --target-arch=morello --enable-debug
gmake -j4
~~~

## Unit Tests

Firstly, ensure that the DPDK physically contiguous memory driver is installed and loaded as described above.

In order to successfully run the sock_ut tests, kern.ipc.maxsockbuf needs to be increased:

~~~{.sh}
sudo sysctl kern.ipc.maxsockbuf=4194304
~~~

This can be made permanent by adding the following to /etc/sysctl.conf:

~~~{.sh}
kern.ipc.maxsockbuf=4194304
~~~

Then run the unit tests with:
~~~{.sh}
./test/unit/unittest.sh
~~~

## Example

The SPDK binaries in build/bin can be used to create an example target stack:

~~~{.sh}
# NVMe over TCP RAM disk (replace LOCAL-IP in sample-nvmf.json)
sudo build/bin/nvmf_tgt -c test/sample-nvmf.json
~~~

To connect to the target from a Linux host:

~~~{.sh}
sudo modprobe nvme_tcp
sudo nvme connect -t tcp -a <TARGET-IP> -n nqn.2016-06.io.spdk:cnode1
~~~

This will create an NVMe device such as /dev/nvme0n1 which can be partitioned with fdisk.

To disconnect the disk:

~~~{.sh}
nvme disconnect -d /dev/nvme0
~~~

## Known Issues

A number of hacks have been used in order to move past blocking problems. These changes, which will need to be revisited, have been identified with:

~~~{.sh}
SPDK_ARM_MORELLO_HACK
SPDK_ARM_PURECAP_HACK
~~~

## Acknowledgements

[rtegrity](https://rtegrity.com/) has been able to develop this project thanks to the support of the [Digital Security by Design](https://www.dsbd.tech/) Technology Access Programme.

## Core Maintainers

The [core maintainers](https://github.com/spdk-morello/spdk/blob/master/MAINTAINERS.md) primary responsibility is to provide technical oversight for the SPDK-Morello project.
The current list includes:
* [Nick Connolly](https://github.com/nconnolly1), [rtegrity](https://rtegrity.com/)

---

# Storage Performance Development Kit

[![Build Status](https://travis-ci.org/spdk/spdk.svg?branch=master)](https://travis-ci.org/spdk/spdk)

NOTE: The SPDK mailing list has moved to a new location. Please visit
[this URL](https://lists.linuxfoundation.org/mailman/listinfo/spdk) to subscribe
at the new location. Subscribers from the old location will not be automatically
migrated to the new location.

The Storage Performance Development Kit ([SPDK](http://www.spdk.io)) provides a set of tools
and libraries for writing high performance, scalable, user-mode storage
applications. It achieves high performance by moving all of the necessary
drivers into userspace and operating in a polled mode instead of relying on
interrupts, which avoids kernel context switches and eliminates interrupt
handling overhead.

The development kit currently includes:

* [NVMe driver](http://www.spdk.io/doc/nvme.html)
* [I/OAT (DMA engine) driver](http://www.spdk.io/doc/ioat.html)
* [NVMe over Fabrics target](http://www.spdk.io/doc/nvmf.html)
* [iSCSI target](http://www.spdk.io/doc/iscsi.html)
* [vhost target](http://www.spdk.io/doc/vhost.html)
* [Virtio-SCSI driver](http://www.spdk.io/doc/virtio.html)

## In this readme

* [Documentation](#documentation)
* [Prerequisites](#prerequisites)
* [Source Code](#source)
* [Build](#libraries)
* [Unit Tests](#tests)
* [Vagrant](#vagrant)
* [AWS](#aws)
* [Advanced Build Options](#advanced)
* [Shared libraries](#shared)
* [Hugepages and Device Binding](#huge)
* [Example Code](#examples)
* [Contributing](#contributing)

<a id="documentation"></a>
## Documentation

[Doxygen API documentation](http://www.spdk.io/doc/) is available, as
well as a [Porting Guide](http://www.spdk.io/doc/porting.html) for porting SPDK to different frameworks
and operating systems.

<a id="source"></a>
## Source Code

~~~{.sh}
git clone https://github.com/spdk/spdk
cd spdk
git submodule update --init
~~~

<a id="prerequisites"></a>
## Prerequisites

The dependencies can be installed automatically by `scripts/pkgdep.sh`.
The `scripts/pkgdep.sh` script will automatically install the bare minimum
dependencies required to build SPDK.
Use `--help` to see information on installing dependencies for optional components

~~~{.sh}
./scripts/pkgdep.sh
~~~

<a id="libraries"></a>
## Build

Linux:

~~~{.sh}
./configure
make
~~~

FreeBSD:
Note: Make sure you have the matching kernel source in /usr/src/ and
also note that CONFIG_COVERAGE option is not available right now
for FreeBSD builds.

~~~{.sh}
./configure
gmake
~~~

<a id="tests"></a>
## Unit Tests

~~~{.sh}
./test/unit/unittest.sh
~~~

You will see several error messages when running the unit tests, but they are
part of the test suite. The final message at the end of the script indicates
success or failure.

<a id="vagrant"></a>
## Vagrant

A [Vagrant](https://www.vagrantup.com/downloads.html) setup is also provided
to create a Linux VM with a virtual NVMe controller to get up and running
quickly.  Currently this has been tested on MacOS, Ubuntu 16.04.2 LTS and
Ubuntu 18.04.3 LTS with the VirtualBox and Libvirt provider.
The [VirtualBox Extension Pack](https://www.virtualbox.org/wiki/Downloads)
or [Vagrant Libvirt] (https://github.com/vagrant-libvirt/vagrant-libvirt) must
also be installed in order to get the required NVMe support.

Details on the Vagrant setup can be found in the
[SPDK Vagrant documentation](http://spdk.io/doc/vagrant.html).

<a id="aws"></a>
## AWS

The following setup is known to work on AWS:
Image: Ubuntu 18.04
Before running  `setup.sh`, run `modprobe vfio-pci`
then: `DRIVER_OVERRIDE=vfio-pci ./setup.sh`

<a id="advanced"></a>
## Advanced Build Options

Optional components and other build-time configuration are controlled by
settings in the Makefile configuration file in the root of the repository. `CONFIG`
contains the base settings for the `configure` script. This script generates a new
file, `mk/config.mk`, that contains final build settings. For advanced configuration,
there are a number of additional options to `configure` that may be used, or
`mk/config.mk` can simply be created and edited by hand. A description of all
possible options is located in `CONFIG`.

Boolean (on/off) options are configured with a 'y' (yes) or 'n' (no). For
example, this line of `CONFIG` controls whether the optional RDMA (libibverbs)
support is enabled:

~~~{.sh}
CONFIG_RDMA?=n
~~~

To enable RDMA, this line may be added to `mk/config.mk` with a 'y' instead of
'n'. For the majority of options this can be done using the `configure` script.
For example:

~~~{.sh}
./configure --with-rdma
~~~

Additionally, `CONFIG` options may also be overridden on the `make` command
line:

~~~{.sh}
make CONFIG_RDMA=y
~~~

Users may wish to use a version of DPDK different from the submodule included
in the SPDK repository.  Note, this includes the ability to build not only
from DPDK sources, but also just with the includes and libraries
installed via the dpdk and dpdk-devel packages.  To specify an alternate DPDK
installation, run configure with the --with-dpdk option.  For example:

Linux:

~~~{.sh}
./configure --with-dpdk=/path/to/dpdk/x86_64-native-linuxapp-gcc
make
~~~

FreeBSD:

~~~{.sh}
./configure --with-dpdk=/path/to/dpdk/x86_64-native-bsdapp-clang
gmake
~~~

The options specified on the `make` command line take precedence over the
values in `mk/config.mk`. This can be useful if you, for example, generate
a `mk/config.mk` using the `configure` script and then have one or two
options (i.e. debug builds) that you wish to turn on and off frequently.

<a id="shared"></a>
## Shared libraries

By default, the build of the SPDK yields static libraries against which
the SPDK applications and examples are linked.
Configure option `--with-shared` provides the ability to produce SPDK shared
libraries, in addition to the default static ones.  Use of this flag also
results in the SPDK executables linked to the shared versions of libraries.
SPDK shared libraries by default, are located in `./build/lib`.  This includes
the single SPDK shared lib encompassing all of the SPDK static libs
(`libspdk.so`) as well as individual SPDK shared libs corresponding to each
of the SPDK static ones.

In order to start a SPDK app linked with SPDK shared libraries, make sure
to do the following steps:

- run ldconfig specifying the directory containing SPDK shared libraries
- provide proper `LD_LIBRARY_PATH`

If DPDK shared libraries are used, you may also need to add DPDK shared
libraries to `LD_LIBRARY_PATH`

Linux:

~~~{.sh}
./configure --with-shared
make
ldconfig -v -n ./build/lib
LD_LIBRARY_PATH=./build/lib/:./dpdk/build/lib/ ./build/bin/spdk_tgt
~~~

<a id="huge"></a>
## Hugepages and Device Binding

Before running an SPDK application, some hugepages must be allocated and
any NVMe and I/OAT devices must be unbound from the native kernel drivers.
SPDK includes a script to automate this process on both Linux and FreeBSD.
This script should be run as root.

~~~{.sh}
sudo scripts/setup.sh
~~~

Users may wish to configure a specific memory size. Below is an example of
configuring 8192MB memory.

~~~{.sh}
sudo HUGEMEM=8192 scripts/setup.sh
~~~

There are a lot of other environment variables that can be set to configure
setup.sh for advanced users. To see the full list, run:

~~~{.sh}
scripts/setup.sh --help
~~~

<a id="examples"></a>
## Example Code

Example code is located in the examples directory. The examples are compiled
automatically as part of the build process. Simply call any of the examples
with no arguments to see the help output. You'll likely need to run the examples
as a privileged user (root) unless you've done additional configuration
to grant your user permission to allocate huge pages and map devices through
vfio.

<a id="contributing"></a>
## Contributing

For additional details on how to get more involved in the community, including
[contributing code](http://www.spdk.io/development) and participating in discussions and other activities, please
refer to [spdk.io](http://www.spdk.io/community)
