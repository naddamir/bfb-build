To build a BlueField boot stream (BFB), run:

````
<distro>/<version>/bfb-build
````

The BFB is created under `/tmp/<distro>/<version>.<pid>` directory.


# 1. Contents

Common files are located under `<distro>/<version>`:<br>
- bfb-build<br>
- create_bfb<br>
- Dockerfile<br>
- install.sh<br>

## Dockerfile
Contains commands to build a container that will represent the target OS that
runs on BlueField using the following steps:
- Container is created from the base target OS image for ARM64.
- The tools required to be installed on the target OS are added.
- Add repository with DOCA packages that includes kernel for BlueField,
  MLNX_OFED drivers, and other DOCA packages.
- Install doca-runtime, doca-tools and doca-sdk meta-packages that bring and
  install all the packages from the DOCA repository.

## install.sh
This script is run on the BlueField during bfb-install (`cat <BFB> > /dev/rshim0/boot`).
It creates and formats partitions on eMMC device and extracts OS tarball.<br>

## create_bfb
This script is run on the Docker container to turn its file system into the target
BFB.

First, it creates initramfs that are loaded on BlueField during bfb-install
(`cat <BFB> > /dev/rshim0/boot`) and adds all the tools necessary to access eMMC
device on BlueField, creates and formats partitions and extracts the target OS
file system. A tarball file that includes the containers file system and
the install.sh script are also packed into the initramfs.<br>
Then it runs the mlx-mkbfb command to create the BFB file.

The BFB file name is based on the content of `/etc/mlnx-release` file which is
included in bf-release package.

## bfb-build
Runs docker commands to build target OS container and run it to create the BFB.


# 2. Customizations for Target OS Image
## 2.1 User Space Packages and Services
Changes in user space package selection can be done in Dockerfile.

To decrease the BFB size and target OS footprint, consider removing the "doca-sdk"
package that brings the development environment required to build DOCA related
software.

To install a smaller subset of the DOCA packages, use the direct list of the
required packages instead of doca-runtime, doca-sdk, and doca-tools.

The online repository with DOCA packages is available under
https://linux.mellanox.com/public/repo/doca/latest.

## 2.2 Kernel changes
To install a customized kernel on the BlueField OS, it is required to rebuild
MLNX_OFED driver packages and other BlueField SoC drivers.

The relevant source packages are available under
https://linux.mellanox.com/public/repo/bluefield/latest/extras/.


**Example for RPM based Distros:**<br>
The following steps can be added to the Dockerfile based on the real kernel and
MLNX_OFED versions.

After installing the customized kernel and kernel-devel packages, download and
build MLNX_OFED drivers:

````
wget https://linux.mellanox.com/public/repo/bluefield/3.8.5/extras/mlnx_ofed/5.5-2.1.7.0/MLNX_OFED_SRC-5.5-2.1.7.0.tgz<br>
tar xzf MLNX_OFED_SRC-5.5-2.1.7.0.tgz<br>
cd xzf MLNX_OFED_SRC-5.5-2.1.7.0<br>
./install.pl -k <kernel version> --kernel-sources /lib/modules/<kernel version>/build \\<br>
	--kernel-extra-args '--with-sf-cfg-drv --without-xdp --without-odp' \\<br>
	--kernel-only --build-only<br>
````

Binary RPMS can be found under `MLNX_OFED_SRC-5.5-2.1.7.0/RPMS` directory. 
Find `MLNX_OFED_SRC-5.5-2.1.7.0/RPMS -name '*rpm' -exec rpm -ihv '{}' \;`.

Build and install BlueField SoC drivers:

````
cd /tmp && wget -r -np -nH --cut-dirs=3 -R "index.html*" https://linux.mellanox.com/public/repo/bluefield/3.8.5/extras/SRPMS/<br>
mkdir -p /tmp/3.8.5/extras/{SPECS,RPMS,SOURCES,BUILD}<br>
for p in 3.8.5/extras/SRPMS/*.src.rpm; do rpmbuild --rebuild -D "debug_package %{nil}" -D "KVERSION <kernel version>" --define "_topdir /tmp/3.8.5/extras" $p;done<br>
rpm -ivh --force /tmp/3.8.5/extras/RPMS/aarch64/*.rpm<br>
````

After installing MLNX_OFED drivers and BlueField SoC drivers, install DOCA user
space packages individually:

**DOCA runtime packages:**<br>

````
yum install -y libpka mlxbf-bootctl rxp-compiler mlx-OpenIPMI mlnx-libsnap \\<br>
mlnx-dpdk mlx-regex spdk hyperscan mlxbf-bfscripts libvma mlnx-snap doca-dpi \\<br>
doca-flow doca-utils doca-apsh doca-regex dpcp rxpbench virtio-net-controller \\<br>
mlnx-iproute2 rdma-core mlnx-nvme ucx-rdmacm ibacm ofed-scripts ucx-knem \\<br>
mstflint iser isert libxpmem mlnx-ethtool perftest mlnx-tools knem ucx ucx-cma \\<br>
openvswitch python3-openvswitch python3-grpcio python3-protobuf \\<br>
openvswitch-ipsec ucx-xpmem libibverbs librdmacm xpmem ucx-ib libibumad mft-oem \\<br>
mft mlnx-fw-updater mlxbf-bootimages collectx-clxapi bf-release
````

**DOCA SDK packages:**<br>

````
yum install -y librxpcompiler-devel mlnx-dpdk-devel hyperscan-devel \\<br>
libvma-devel doca-utils-devel doca-flow-devel doca-dpi-devel doca-apsh-devel \\<br>
grpc-devel openvswitch-devel ucx-devel opensm-devel rdma-core-devel \\<br>
libxpmem-devel
````

**DOCA tools packages:**<br>

````
yum install -y libvma-utils doca-dpi-tools librdmacm-utils opensm-libs opensm \\<br>
srp_daemon infiniband-diags-compat opensm-static libibverbs-utils \\<br>
infiniband-diags
````

Update the kernel version parameter for the `create_bfb` command at the end of the
Dockerfile.

To change the resulted BFB name and version edit `/etc/mlnx-release` file after
bf-release RPM installation.
