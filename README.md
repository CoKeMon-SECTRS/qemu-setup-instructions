# CoKeMon with eMMC Storage Logging on QEMU

## Prerequisites

Tested on Debian 13 (trixie).

```bash
sudo apt install build-essential git python3 python3-pip python3-venv \
    gcc-aarch64-linux-gnu g++-aarch64-linux-gnu \
    device-tree-compiler gdisk gawk cmake \
    flex bison libssl-dev bc \
    libglib2.0-dev libpixman-1-dev libfdt-dev libslirp-dev ninja-build \
    libcap-ng-dev libattr1-dev uuid-dev acpica-tools \
    python3-pyelftools python3-cryptography python3-tomli meson \
    perl libgit-repository-perl libxml-parser-perl liburi-perl \
    cpio unzip wget curl rsync file xz-utils tmux

mkdir -p ~/.local/bin
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/.local/bin/repo
chmod a+x ~/.local/bin/repo
export PATH="$HOME/.local/bin:$PATH"

# Working directory that will hold everything
git clone https://github.com/CoKeMon-SECTRS/qemu-setup-instructions.git
export ROOT=$PWD
```

## 1. Linaro CCA stack

```bash
mkdir $ROOT/cca && cd $ROOT/cca
repo init -u $ROOT/qemu-setup-instructions -m qemu_v8_cca.xml
repo sync -j$(nproc)

cd $ROOT/cca/linux
git am $ROOT/qemu-setup-instructions/patches/0001-CoKeMon-Modifications-for-Linux-with-QEMU.patch
```

## 2. L4Re and Fiasco

### 2a. Sources

```bash
mkdir $ROOT/l4re_qemu && cd $ROOT/l4re_qemu
git clone https://github.com/kernkonzept/ham.git
./ham/ham init -u https://github.com/kernkonzept/manifest.git
cp $ROOT/qemu-setup-instructions/manifest.xml .ham/manifest.xml
./ham/ham sync

# Pin the Kernkonzept packages to match the fork's build system
cd $ROOT/l4re_qemu/l4/pkg
for d in acpica ada ahci-driver atkins cons emmc-driver gnu-efi hello \
         virtio-net-switch l4re-core l4virtio libblock-device libfdt libvcpu \
         nvme-driver readline rtc tinit tvmm uvmm virtio-net zlib; do
  git -C $d checkout -q $(git -C $d rev-list -1 --before='2025-06-10 23:59' HEAD)
done

# Upstream mk commit required by the pinned packages
cd $ROOT/l4re_qemu/l4
git checkout -b cokemon-local
git fetch https://github.com/kernkonzept/mk.git master
git cherry-pick 49119d0

# Place the osswitch modules at 0xc1000000
sed -i '/^entry osswitch$/a modaddr 0x81000000' conf/modules.list
git commit -qam "QEMU CCA: place osswitch modules at 0xc1000000"
```

### 2b. Patched io, drivers-frst and bootstrap

```bash
cd $ROOT/l4re_qemu/l4/pkg/io
git init -q
git fetch https://github.com/kernkonzept/io.git ca4f36d7a552f0db8c413840947a0842eb47e6fb
git checkout -f FETCH_HEAD
git am CokeMon-modifications.patch

cd $ROOT/l4re_qemu/l4/pkg/drivers-frst
git init -q
git fetch https://github.com/kernkonzept/drivers-frst.git 70000c2cd4ff8dc78b670208b7c612b756d689bd
git checkout -f FETCH_HEAD
git am CoKeMon-Modifications.patch

cd $ROOT/l4re_qemu/l4/pkg/bootstrap
git init -q
git fetch https://github.com/kernkonzept/bootstrap.git f9f0f20aa52ccb50ea3cc6d0547ec05473ed8179
git checkout -f FETCH_HEAD
git am CokeMon-modifications.patch
```

### 2c. Build

```bash
cd $ROOT/l4re_qemu
cat > env.sh << EOF
export L4RE_SRCDIR="$(pwd)/l4"
export L4RE_OBJDIR_ARM64="$(pwd)/l4re_builds/arm64"
export KERNEL_SRCDIR="$(pwd)/fiasco"
export KERNEL_OBJDIR_ARM64="$(pwd)/fiasco_builds/arm64"
EOF
source env.sh

# Fiasco, loaded at 0xc8001000
cd $ROOT/l4re_qemu/fiasco
make B=../fiasco_builds/arm64 T=arm64-virt-el2
cd ../fiasco_builds/arm64
echo 'CONFIG_ARM_SMC_USER=y' >> globalconfig.out
echo 'override RAM_PHYS_BASE := 0xc8000000' > Makeconf.local
make olddefconfig
make -j$(nproc)

# L4Re, bootstrap linked at 0xc0000000
cd $ROOT/l4re_qemu/l4
make B=../l4re_builds/arm64 T=arm64-virt-v8a
cd ../l4re_builds/arm64
mkdir -p conf
echo 'DEFAULT_RELOC_arm64 := 0x80000000' > conf/Makeconf.boot
make olddefconfig
make -j$(nproc)
```

## 3. Disk image

```bash
mkdir -p $ROOT/cca/fake-disk && cd $ROOT/cca/fake-disk
dd if=/dev/zero of=log.img bs=1M count=64
sgdisk -n 1:2048:+60M -t 1:8300 log.img
sgdisk -u 1:115715E5-F934-471E-9A9F-AEB7DE29A878 log.img
```

## 4. Build the CCA system and images

```bash
cd $ROOT/cca/build
BR_FLAGS="BR2_PACKAGE_KVMTOOL_CCA=n BR2_PACKAGE_QEMU_CCA=n \
          BR2_PACKAGE_CCA_WORKLOAD_ATTESTATION=n"
make -j$(nproc) toolchains
make -j$(nproc) $BR_FLAGS
dtc -I dts -O dtb -o ../out/bin/cca.dtb cca.dts

cd $ROOT/l4re_qemu/l4re_builds/arm64
source ../../env.sh
make E=osswitch rawimage \
  MODULE_SEARCH_PATH=$ROOT/l4re_qemu/fiasco_builds/arm64:$ROOT/l4re_qemu/l4/conf/examples
cp images/bootstrap_osswitch.raw $ROOT/cca/out/bin/Image_Micro_Switch
cp $ROOT/cca/linux/arch/arm64/boot/Image $ROOT/cca/out/bin/Image_Linux
```

## 5. Run

From inside a tmux session:

```bash
cd $ROOT/cca/build
./run.sh
```

Each console opens in its own tmux window. In the Linux console, log in as
`root` and test the logging with `echo "Test" > /dev/logmicro`.

## 6. Verify logged data

After shutting down QEMU:

```bash
dd if=$ROOT/cca/fake-disk/log.img bs=512 skip=2048 count=20 2>/dev/null | strings -n 1
```
