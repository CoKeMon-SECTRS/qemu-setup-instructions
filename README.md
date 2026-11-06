# ARM CCA OS-Switching with eMMC Block Storage Logging

OS-switching between Linux and L4Re/Fiasco on ARM CCA via SMC calls through
TF-A. An L4Re application (`osswitch`) captures Linux kernel logs after each
OS switch and writes them to a persistent eMMC block device.

## Prerequisites

Linux x86_64 host (Ubuntu/Debian) with at least 16 GB RAM.

```bash
sudo apt install build-essential git python3 python3-pip \
    gcc-aarch64-linux-gnu g++-aarch64-linux-gnu \
    device-tree-compiler gdisk \
    flex bison libssl-dev bc \
    libglib2.0-dev libpixman-1-dev ninja-build \
    perl libgit-repository-perl libxml-parser-perl liburi-perl \
    cpio unzip wget curl
```

Install the `repo` tool:
```bash
mkdir -p ~/.local/bin
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/.local/bin/repo
chmod a+x ~/.local/bin/repo
export PATH="$HOME/.local/bin:$PATH"
```

## Directory layout

All commands below assume the following directory structure, rooted in a
working directory of your choice (referred to as `$ROOT`):

```
$ROOT/
├── cca/                     # Linaro CCA stack (section 1)
│   ├── build/
│   ├── linux/
│   ├── trusted-firmware-a/
│   ├── out/bin/
│   └── fake-disk/
├── qemu-setup-instructions/ # This repository (section 2)
└── l4re_qemu/               # L4Re + Fiasco (section 3)
    ├── l4/
    ├── fiasco/
    ├── ham/
    ├── fiasco_builds/arm64/
    └── l4re_builds/arm64/
```

## 1. Set up the Linaro CCA base system

This project is built on top of the Linaro OP-TEE QEMU v8 CCA reference stack.
The original Linaro CCA build instructions are at
[gitlab.com/Linaro/cca-public/build-instructions](https://gitlab.com/Linaro/cca-public/build-instructions).
The manifest is hosted at `git.codelinaro.org` and uses the `cca/v7` branch.

```bash
# From $ROOT
mkdir cca && cd cca
repo init -u https://git.codelinaro.org/linaro/dcap/op-tee-4.2.0/manifest.git \
    -b cca/v7
repo sync -j$(nproc)
cd ..
```

This clones all components (TF-A, Linux, QEMU, buildroot, etc.) into `cca/`.
The key versions pulled by this manifest are:

| Component | Source | Revision |
|-----------|--------|----------|
| Build system | `git.codelinaro.org/linaro/dcap/op-tee-4.2.0/build.git` | `cca/v5` |
| TF-A | `git.codelinaro.org/linaro/dcap/TF-A/trusted-firmware-a.git` | `cca/v4` |
| RMM | `git.codelinaro.org/linaro/dcap/rmm` | `cca/v4` |
| Linux | `git.gitlab.arm.com/linux-arm/linux-cca` | `cca-host/v7` |
| QEMU | `github.com/qemu/qemu.git` | `stable-9.2` |
| Buildroot | `github.com/buildroot/buildroot.git` | `2024.08` |
| OP-TEE OS | `github.com/OP-TEE/optee_os.git` | `4.2.0` |
| Hafnium | `git.trustedfirmware.org/hafnium/hafnium.git` | `v2.10` |

## 2. Apply patches to Linux, TF-A, and the build system

Clone this repository and apply the CoKeMon patches on top of the Linaro repos:

```bash
# From $ROOT
git clone <this-repo-url> qemu-setup-instructions

cd cca/trusted-firmware-a
git am ../../qemu-setup-instructions/patches/0001-CoKeMon-Modifications-for-TF-A-with-QEMU.patch
cd ..

cd linux
git am ../../qemu-setup-instructions/patches/0001-CoKeMon-Modifications-for-Linux-with-QEMU.patch
cd ..

cd build
git am ../../qemu-setup-instructions/patches/0001-Modifications-for-Cokemon.patch
cd ../..
```

## 3. Set up L4Re and Fiasco

L4Re and Fiasco are developed by Kernkonzept. The original setup instructions
are at [kernkonzept.com/L4Re/guides/l4re](https://kernkonzept.com/L4Re/guides/l4re).
The source repositories are at [github.com/kernkonzept](https://github.com/kernkonzept).

### 3a. Clone all repositories with ham

This repository includes a custom `manifest.xml` that points the modified
packages (L4Re build system, bootstrap, io, drivers-frst, osswitch) and Fiasco
to the CoKeMon forks, while fetching all other packages from Kernkonzept:

```bash
# From $ROOT
mkdir l4re_qemu && cd l4re_qemu
git clone https://github.com/kernkonzept/ham.git
./ham/ham init -u https://github.com/kernkonzept/manifest.git
cp ../qemu-setup-instructions/manifest.xml .ham/manifest.xml
./ham/ham sync
```

### 3b. Set up the build environment

Create the environment file and build Fiasco and L4Re:

```bash
# From $ROOT/l4re_qemu

# Create env.sh with absolute paths
cat > env.sh << EOF
export L4RE_SRCDIR="$(pwd)/l4"
export L4RE_OBJDIR_ARM64="$(pwd)/l4re_builds/arm64"
export KERNEL_SRCDIR="$(pwd)/fiasco"
export KERNEL_OBJDIR_ARM64="$(pwd)/fiasco_builds/arm64"
EOF
source env.sh

# Create and build Fiasco
cd fiasco
make B=../fiasco_builds/arm64 T=arm64-virt-el2
cd ../fiasco_builds/arm64
echo 'CONFIG_ARM_SMC_USER=y' >> globalconfig.out
make olddefconfig
make -j$(nproc)

# Create and build L4Re
cd ../../l4
make B=../l4re_builds/arm64 T=arm64
cd ../l4re_builds/arm64
make olddefconfig
make -j$(nproc)
cd ../..
```

## 4. Create the disk image

```bash
# From $ROOT
mkdir -p cca/fake-disk
dd if=/dev/zero of=cca/fake-disk/log.img bs=1M count=64
sgdisk -n 1:2048:+60M -t 1:8300 cca/fake-disk/log.img
sgdisk -u 1:115715E5-F934-471E-9A9F-AEB7DE29A878 cca/fake-disk/log.img
```

The partition UUID `115715E5-F934-471E-9A9F-AEB7DE29A878` must match the
`--device` argument in `osswitch-blk.cfg`.

## 5. Build the CCA system

```bash
# From $ROOT

# Build TF-A, Linux, QEMU, buildroot
cd cca/build
make -j$(nproc)

# Compile the device tree
dtc -I dts -O dtb -o ../out/bin/cca.dtb cca.dts

# Build the L4Re rawimage
cd ../../l4re_qemu/l4re_builds/arm64
make E=osswitch rawimage \
  MODULE_SEARCH_PATH=../../fiasco_builds/arm64:../../l4/conf/examples

# Copy images to the CCA output directory
cp images/rawimage ../../../cca/out/bin/Image_Micro_Switch
cp ../../../cca/linux/arch/arm64/boot/Image ../../../cca/out/bin/Image_Linux
cd ../../..
```

## 6. Run

```bash
# From $ROOT
cd cca/build
make run-only
```

Connect to the L4Re console:
```bash
telnet localhost 54321
```

## 7. Verify logged data

After shutting down QEMU, read the log data from the disk image:

```bash
# From $ROOT
dd if=cca/fake-disk/log.img bs=512 skip=2048 count=20 2>/dev/null | strings -n 1
```

To reset the image for a fresh run:
```bash
dd if=/dev/zero of=cca/fake-disk/log.img bs=512 seek=2048 count=20 conv=notrunc
```

## Repository structure

```
qemu-setup-instructions/
├── README.md                 # This file
├── manifest.xml              # Custom ham manifest for L4Re + Fiasco
├── read.sh                   # Script to read logged data from the disk image
└── patches/
    ├── 0001-CoKeMon-Modifications-for-Linux-with-QEMU.patch
    ├── 0001-CoKeMon-Modifications-for-TF-A-with-QEMU.patch
    └── 0001-Modifications-for-Cokemon.patch
```
