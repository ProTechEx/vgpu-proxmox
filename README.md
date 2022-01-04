# NVIDIA vGPU on PVE 7.1 with a NVIDIA T1000 GPU

This tutorial (and included patches) should allow you to use vGPU unlock on PVE 7.1 with the opt-in 5.15 Linux Kernel with a NVIDIA T1000 GPU. The GPU uses the TU117 Chip so other GPUs with the same Chip (T400, T600, GTX 1650 **NOT** Super) will probably work (no guarantees).

### This tutorial assumes you are using a clean install of PVE 7.1, or ymmv when using an existing installation. Make sure to always have backups!

## Packages

Make sure to add the community pve repo and get rid of the enterprise repo (you can skip this step if you have a valid enterprise subscription)

```bash
echo "deb http://download.proxmox.com/debian/pve bullseye pve-no-subscription" >> /etc/apt/sources.list
rm /etc/apt/sources.list.d/pve-enterprise.list
```

Update and upgrade
```bash
apt update
apt dist-upgrade
```

PVE 7.1 comes with version 5.13 of the Linux Kernel, that version is incompatible with vGPU. For this guide you will have to install version 5.15, which will probably come with PVE 7.2 (~Q2 2022) but is opt-in on current PVE versions
```bash
apt install -y pve-kernel-5.15 pve-headers-5.15
```

Next we need to install a few more packages like git, a compiler and some other tools
```bash
apt install -y git build-essential dkms jq pve-headers mdevctl
```

## Git repos and glorious [Rust](https://www.rust-lang.org/) compiler

First, clone this repo to your home folder (in this case `/root/`)
```bash
git clone https://gitlab.com/polloloco/vgpu-5.15.git
```

Clone two additional git repos for vGPU unlock
```bash
cd /opt
git clone https://github.com/DualCoder/vgpu_unlock
git clone https://github.com/p0lloloco/vgpu_unlock-rs
```

After that, install the rust compiler
```bash
curl https://sh.rustup.rs -sSf | sh -s -- -y
```

Now make the rust binaries available in your $PATH (you only have to do it the first time after installing rust)
```bash
source $HOME/.cargo/env
```

Enter the `vgpu_unlock-rs` directory and compile the library. Depending on your hardware and internet connection that may take a while
```bash
cd vgpu_unlock-rs/
cargo build --release
```

## Create files for vGPU unlock

The vgpu_unlock-rs library requires a few files and folders in order to work properly, lets create those

First create the folder for your vgpu unlock config and create an empty config file
```bash
mkdir /etc/vgpu_unlock
touch /etc/vgpu_unlock/profile_override.toml
```

Then, create folders and files for systemd to load the vgpu_unlock-rs library when starting the nvidia vgpu services
```bash
mkdir /etc/systemd/system/{nvidia-vgpud.service.d,nvidia-vgpu-mgr.service.d}
echo -e "[Service]\nEnvironment=LD_PRELOAD=/opt/vgpu_unlock-rs/target/release/libvgpu_unlock_rs.so" > /etc/systemd/system/nvidia-vgpud.service.d/vgpu_unlock.conf
echo -e "[Service]\nEnvironment=LD_PRELOAD=/opt/vgpu_unlock-rs/target/release/libvgpu_unlock_rs.so" > /etc/systemd/system/nvidia-vgpu-mgr.service.d/vgpu_unlock.conf
```


## Enabling IOMMU
#### Note: Usually this isn't required for vGPU to work, but it doesn't hurt to enable it. You can skip this section, but if you run into problems later on, make sure to enable IOMMU.

Assuming you installed PVE with ZFS-on-root and efi, you are booting with systemd-boot. All other installations use grub. The following instructions *ONLY* apply to systemd-boot, grub is different.

To enable IOMMU you have to enable it in your UEFI first. Due to it being vendor specific, I am unable to provide instructions for that, but usually for Intel systems the option you are looking for is called something like "Vt-d", AMD systems tend to call it "IOMMU".

After enabling IOMMU in your UEFI, you have to add some options to your kernel to enable it in proxmox. Edit the kernel command line like this
```bash
nano /etc/kernel/cmdline
```

On a clean installation the file might look similar to this:
```
root=ZFS=rpool/ROOT/pve-1 boot=zfs
```

On Intel systems, append this line at the end
```
intel_iommu=on iommu=pt
```

For AMD, use this
```
amd_iommu=on iommu=pt
```

After editing the file, it should look similar to this
```
root=ZFS=rpool/ROOT/pve-1 boot=zfs intel_iommu=on iommu=pt
```

Save and exit using Ctrl+O and then Ctrl+X

## Loading required kernel modules and blacklisting the open source nvidia driver

We have to load the `vfio`, `vfio_iommu_type1`, `vfio_pci` and `vfio_virqfd` kernel modules to get vGPU working
```bash
echo -e "vfio\nvfio_iommu_type1\nvfio_pci\nvfio_virqfd" >> /etc/modules
```

Proxmox comes with the open source nouveau driver for nvidia gpus, however we have to use our patched nvidia driver to enable vGPU. The next line will prevent the nouveau driver from loading
```bash
echo "blacklist nouveau" >> /etc/modprobe.d/blacklist.conf
```

## IMPORTANT: Apply our kernel configuration
#### Note: This only applies to systemd-boot, if you are using grub, you can't use these instructions

```bash
proxmox-boot-tool refresh
```

...and reboot
```bash
reboot
```

## Check if IOMMU is enabled
#### Note: See section "Enabling IOMMU", this is optional

Wait for your server to restart, then type this into a root shell
```bash
dmesg | grep -e DMAR -e IOMMU
```

On my Intel system the output looks like this
```
[    0.007235] ACPI: DMAR 0x000000009CC98B68 0000B8 (v01 INTEL  BDW      00000001 INTL 00000001)
[    0.007255] ACPI: Reserving DMAR table memory at [mem 0x9cc98b68-0x9cc98c1f]
[    0.020766] DMAR: IOMMU enabled
[    0.062294] DMAR: Host address width 39
[    0.062296] DMAR: DRHD base: 0x000000fed90000 flags: 0x0
[    0.062300] DMAR: dmar0: reg_base_addr fed90000 ver 1:0 cap c0000020660462 ecap f0101a
[    0.062302] DMAR: DRHD base: 0x000000fed91000 flags: 0x1
[    0.062305] DMAR: dmar1: reg_base_addr fed91000 ver 1:0 cap d2008c20660462 ecap f010da
[    0.062307] DMAR: RMRR base: 0x0000009cc18000 end: 0x0000009cc25fff
[    0.062309] DMAR: RMRR base: 0x0000009f000000 end: 0x000000af1fffff
[    0.062312] DMAR-IR: IOAPIC id 8 under DRHD base  0xfed91000 IOMMU 1
[    0.062314] DMAR-IR: HPET id 0 under DRHD base 0xfed91000
[    0.062315] DMAR-IR: x2apic is disabled because BIOS sets x2apic opt out bit.
[    0.062316] DMAR-IR: Use 'intremap=no_x2apic_optout' to override the BIOS setting.
[    0.062797] DMAR-IR: Enabled IRQ remapping in xapic mode
[    0.302431] DMAR: No ATSR found
[    0.302432] DMAR: No SATC found
[    0.302433] DMAR: IOMMU feature pgsel_inv inconsistent
[    0.302435] DMAR: IOMMU feature sc_support inconsistent
[    0.302436] DMAR: IOMMU feature pass_through inconsistent
[    0.302437] DMAR: dmar0: Using Queued invalidation
[    0.302443] DMAR: dmar1: Using Queued invalidation
[    0.333474] DMAR: Intel(R) Virtualization Technology for Directed I/O
[    3.990175] i915 0000:00:02.0: [drm] DMAR active, disabling use of stolen memory
```

Depending on your mainboard and cpu, the output will be different, in my output the important line is the third one: `DMAR: IOMMU enabled`. If you see something like that, IOMMU is enabled.

## NVIDIA Driver

### Choosing the right driver version

This is the tricky part, at the time of writing (Jan 2022), there are three active branches of the NVIDIA vGPU driver. The latest is branch 13 (long term support branch until mid 2024) with driver version 470. I had no luck getting *any* version of that driver to work with vGPU at all but as always - ymmv.

Branch 12 is a "regular" production branch with support until January of 2022 and has driver version number 460. Lots of people are running that driver in combination with the Linux Kernel 5.15. I got it installed with my gpu, but as soon as I tried to use the gpu in my VM, the display would freeze every 30-ish seconds and `nvidia-vgpu-mgr.service` would report an error similar to `error: vmiop_log: (0x0): XID 43 detected on physical_chid:0x1c, guest_chid:0x14`. At first I thought I messed up some of the driver patches required to get the driver working on kernels newer than 5.11 - so I tried on PVE 6.4 without any patches (5.4 kernel) but got the same errors there. If anyone knows what's causing this error, or even how to fix it, **please** let me know :)

Ruling out those two branches only leaves the older long term support branch 11: It is supported until mid 2023 and has the driver version 450. Like the other branch (12), you have to patch some parts of the driver to get it working on the Linux Kernel 5.15. I tried every patch I could find on the Internet (mostly twelve.patch and fourteen.patch and their variations) but no combination of them allowed me to install the driver - the installer would always complain about my system being incompatible. So I spent a few hours looking at the existing patches and reviewing the files they patch to finally come up with my own patch: Basically, it adapts twelve.patch and fourteen.patch to this older driver (they seem to be designed for the branch 12 driver) and merges them into a single patch.

### Obtaining the driver

I will be using the latest driver from branch 11 (at the time of writing that would be 11.6 / 450.156).

NVIDIA doesn't let you freely download vGPU drivers like they do with GeForce or normal Quadro drivers, instead you have to download them through the [NVIDIA Licensing Portal](https://nvid.nvidia.com/dashboard/) (see: [https://www.nvidia.com/en-us/drivers/vgpu-software-driver/](https://www.nvidia.com/en-us/drivers/vgpu-software-driver/)). You can sign up for a free evaluation to get access to the download page.

After downloading version 11.6 you should have a zip file called `NVIDIA-GRID-Linux-KVM-450.156-450.156.00-453.23.zip`, extract that and copy the file `NVIDIA-Linux-x86_64-450.156-vgpu-kvm.run` to your PVE host into the `/root/` folder
```bash
scp NVIDIA-Linux-x86_64-450.156-vgpu-kvm.run root@pve:/root/
```

### Patching the driver

Now, on the proxmox host, make the driver executable
```bash
chmod +x NVIDIA-Linux-x86_64-450.156-vgpu-kvm.run
```

And then unpack it
```bash
./NVIDIA-Linux-x86_64-450.156-vgpu-kvm.run -x
```

Go inside the extracted folder
```bash
cd NVIDIA-Linux-x86_64-450.156-vgpu-kvm/
```

To be able to install the driver on your proxmox host, apply the driver patch
```bash
patch -p0 < ~/vgpu-5.15/450_5.15.patch
```

If everything went right (and you are using the exact same nvidia driver version 11.6), the output should be exactly this
```
patching file ./kernel/Kbuild
patching file ./kernel/conftest.sh
patching file ./kernel/nvidia-vgpu-vfio/nvidia-vgpu-vfio.c
patching file ./kernel/nvidia-vgpu-vfio/nvidia-vgpu-vfio.h
patching file ./kernel/nvidia/nv-frontend.c
```

There is a second patch you need to apply.

#### Warning: If you followed every step of this tutorial it should be safe to just apply it, but if you did anything different than I, you should check if the paths in the patch are valid for you.

```bash
patch -p0 < ~/vgpu-5.15/unlock.patch
```

The output should be exactly this
```
patching file ./kernel/nvidia/nvidia.Kbuild
patching file ./kernel/nvidia/os-interface.c
```

### Installing the driver

Now that all the required patches are applied, you can install the driver
```bash
./nvidia-installer --dkms
```

The installer will ask you `Would you like to register the kernel module sources with DKMS? This will allow DKMS to automatically build a new module, if you install a different kernel later.`, answer with `Yes`.

Depending on your hardware, the installation could take a minute or two.

If everything went right, you will be presented with this message.
```
Installation of the NVIDIA Accelerated Graphics Driver for Linux-x86_64 (version: 450.156) is now complete.
```

Click `Ok` to exit the installer.

To finish the installation, reboot.
```bash
reboot
```

### Finishing touches

Wait for your server to reboot, then type this into the shell to check if the driver install worked
```bash
nvidia-smi
```

You should get an output similar to this one
```
Mon Jan  3 20:41:15 2022
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 450.156      Driver Version: 450.156      CUDA Version: N/A      |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  T1000               On   | 00000000:01:00.0 Off |                  N/A |
| 32%   39C    P8    N/A /  50W |     30MiB /  4095MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

To verify if the vGPU unlock worked, type this command
```bash
mdevctl types
```

The output will be similar to this
```
0000:01:00.0
  nvidia-256
    Available instances: 24
    Device API: vfio-pci
    Name: GRID RTX6000-1Q
    Description: num_heads=4, frl_config=60, framebuffer=1024M, max_resolution=5120x2880, max_instance=24
  nvidia-257
    Available instances: 12
    Device API: vfio-pci
    Name: GRID RTX6000-2Q
    Description: num_heads=4, frl_config=60, framebuffer=2048M, max_resolution=7680x4320, max_instance=12
  nvidia-258
    Available instances: 8
    Device API: vfio-pci
    Name: GRID RTX6000-3Q
    Description: num_heads=4, frl_config=60, framebuffer=3072M, max_resolution=7680x4320, max_instance=8
---SNIP---
```

If this command doesn't return any output, vGPU unlock isn't working.

### Bonus: working `nvidia-smi vgpu` command

I've included an adapted version of the `nvidia-smi` [wrapper script](https://github.com/erin-allison/nvidia-merged-arch/blob/d2ce752cd38461b53b7e017612410a3348aa86e5/nvidia-smi) to get useful output from `nvidia-smi vgpu`.

Without that wrapper script, running `nvidia-smi vgpu` in your shell results in this output
```
No supported devices in vGPU mode
```

With the wrapper script, the output looks similar to this
```
Mon Jan  3 20:54:35 2022
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 450.156                Driver Version: 450.156                   |
|---------------------------------+------------------------------+------------+
| GPU  Name                       | Bus-Id                       | GPU-Util   |
|      vGPU ID     Name           | VM ID     VM Name            | vGPU-Util  |
|=================================+==============================+============|
|   0  T1000                      | 00000000:01:00.0             |   0%       |
+---------------------------------+------------------------------+------------+
```

To install this script, copy the `nvidia-smi` file from this repo to `/usr/local/bin` and make it executable
```bash
cp ~/vgpu-5.15/nvidia-smi /usr/local/bin/
chmod +x /usr/local/bin/nvidia-smi
```

Run this in your shell (you might have to logout and back in first) to see if it worked
```bash
nvidia-smi vgpu
```

## TODO (soon tm)

- Add references to vgpu projects
- Add basic profile_override.toml config
- Add proxmox VM installation guide

## Contributing
Pull requests are welcome (factual errors, amendments, grammar/spelling mistakes etc).