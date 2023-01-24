# NVIDIA vGPU on Proxmox

This document serves as a guide to install NVIDIA vGPU host drivers on the latest Proxmox VE version, at time of writing this its pve 7.3.

You can follow this guide if you have a vGPU supported card from [this list](https://docs.nvidia.com/grid/gpus-supported-by-vgpu.html), or if you are using a consumer GPU from the GeForce series or a non-vGPU qualified Quadro GPU. There are several sections with a title similar to "Have a vGPU supported GPU? Read here" in this document, make sure to read those very carefully as this is where the instructions differ for a vGPU qualified card and a consumer card.

## Supported cards

The following consumer/not-vGPU-qualified NVIDIA GPUs can be used with vGPU:
- Most GPUs from the Maxwell 2.0 generation (GTX 9xx, Quadro Mxxxx, Tesla Mxx) **EXCEPT the GTX 970**
- All GPUs from the Pascal generation (GTX 10xx, Quadro Pxxxx, Tesla Pxx)
- All GPUs from the Turing generation (GTX 16xx, RTX 20xx, Txxxx)

If you have GPUs from the Ampere and Ada Lovelace generation, you are out of luck, unless you have a vGPU qualified card from [this list](https://docs.nvidia.com/grid/gpus-supported-by-vgpu.html) like the A5000 or RTX 6000 Ada. If you have one of those cards, please consult the [NVIDIA documentation](https://docs.nvidia.com/grid/15.0/grid-vgpu-user-guide/index.html) for help with setting it up.

> **!!! THIS MEANS THAT YOUR RTX 30XX or 40XX WILL NOT WORK !!!**

This guide and all my tests were done on a RTX 2080 Ti which is based on the Turing architechture.

## Important notes before starting
- This tutorial assumes you are using a clean install of Proxmox VE 7.3.
- If you tried GPU-passthrough before, you absolutely **MUST** revert all of the steps you did to set that up.
- If you only have one GPU in your system with no iGPU, your local monitor will **NOT** give you any output anymore after the system boots up. Use SSH or a serial connection if you want terminal access to your machine.
- Most of the steps can be applied to other linux distributions, however I'm only covering Proxmox VE here.
- You **HAVE TO** use a supported linux kernel version. Something in the range of 5.14 up to 5.18 should work. Newer kernels like 5.19 or 6.1 do **NOT** work at this point in time.

> ## Are you upgrading from a previous version of this guide?
>
> If you are upgrading from a previous version of this guide, you should uninstall the old driver by running `nvidia-uninstall` first.
>
> Then you also have to make sure that you are using the latest version of `vgpu_unlock-rs`, otherwise it won't work with the latest driver.
>
> Either delete the folder `/opt/vgpu_unlock-rs` or enter the folder and run `git pull` and then recompile the library again using `cargo build --release`

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

We need to install a few more packages like git, a compiler and some other tools.
```bash
apt install -y git build-essential dkms pve-headers mdevctl
```

## Git repos and [Rust](https://www.rust-lang.org/) compiler

First, clone this repo to your home folder (in this case `/root/`)
```bash
git clone https://gitlab.com/polloloco/vgpu-proxmox.git
```

You also need the vgpu_unlock-rs repo
```bash
cd /opt
git clone https://github.com/mbilker/vgpu_unlock-rs.git
```

After that, install the rust compiler
```bash
curl https://sh.rustup.rs -sSf | sh -s -- -y --profile minimal
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

> ### Have a vgpu supported card? Read here!
>
> If you don't have a card like the Tesla P4, or any other gpu from [this list](https://docs.nvidia.com/grid/gpus-supported-by-vgpu.html), please continue reading at [Enabling IOMMU](#enabling-iommu)
>
> Disable the unlock part as doing this on a gpu that already supports vgpu, could break things as it introduces unnecessary complexity and more points of possible failure:
> ```bash
> echo "unlock = false" > /etc/vgpu_unlock/config.toml
> ```

## Enabling IOMMU
#### Note: Usually this isn't required for vGPU to work, but it doesn't hurt to enable it. You can skip this section, but if you run into problems later on, make sure to enable IOMMU.

To enable IOMMU you have to enable it in your BIOS/UEFI first. Due to it being vendor specific, I am unable to provide instructions for that, but usually for Intel systems the option you are looking for is called something like "Vt-d", AMD systems tend to call it "IOMMU".

After enabling it in your BIOS/UEFI, you also have to enable it in your kernel. Depending on how your system is booting, there are two ways to do that.

If you installed your system with ZFS-on-root and in UEFI mode, then you are using systemd-boot, everything else is GRUB. GRUB is way more common so if you are unsure, you are probably using that.

Depending on which system you are using to boot, you have to chose from the following two options:

<details>
  <summary>GRUB</summary>

  Open the file `/etc/default/grub` in your favorite editor
  ```bash
  nano /etc/default/grub
  ```

  The kernel parameters have to be appended to the variable `GRUB_CMDLINE_LINUX_DEFAULT`. On a clean installation that line should look like this
  ```
  GRUB_CMDLINE_LINUX_DEFAULT="quiet"
  ```

  If you are using an Intel system, append this after `quiet`:
  ```
  intel_iommu=on iommu=pt
  ```

  On AMD systems, append this after `quiet`:
  ```
  amd_iommu=on iommu=pt
  ```

  The result should look like this (for intel systems):
  ```
  GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"
  ```

  Now, save and exit from the editor using Ctrl+O and then Ctrl+X and then apply your changes:
  ```bash
  update-grub
  ```
</details>

<details>
  <summary>systemd-boot</summary>

  The kernel parameters have to be appended to the commandline in the file `/etc/kernel/cmdline`, so open that in your favorite editor:
  ```bash
  nano /etc/kernel/cmdline
  ```

  On a clean installation the file might look similar to this:
  ```
  root=ZFS=rpool/ROOT/pve-1 boot=zfs
  ```

  On Intel systems, append this at the end
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

  Now, save and exit from the editor using Ctrl+O and then Ctrl+X and then apply your changes:
  ```bash
  proxmox-boot-tool refresh
  ```
</details>

## Loading required kernel modules and blacklisting the open source nvidia driver

We have to load the `vfio`, `vfio_iommu_type1`, `vfio_pci` and `vfio_virqfd` kernel modules to get vGPU working
```bash
echo -e "vfio\nvfio_iommu_type1\nvfio_pci\nvfio_virqfd" >> /etc/modules
```

Proxmox comes with the open source nouveau driver for nvidia gpus, however we have to use our patched nvidia driver to enable vGPU. The next line will prevent the nouveau driver from loading
```bash
echo "blacklist nouveau" >> /etc/modprobe.d/blacklist.conf
```

## Applying our kernel configuration

I'm not sure if this is needed, but it doesn't hurt :)

```bash
update-initramfs -u -k all
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

This repo contains patches that allow you to use vGPU on not-qualified-vGPU cards (consumer GPUs). Those patches are binary patches, which means that each patch works **ONLY** for a specific driver version.

I've created patches for the following driver versions:
- 15.1 (525.85.07)
- 15.0 (525.60.12)
- 14.4 (510.108.03)
- 14.3 (510.108.03)
- 14.2 (510.85.03)

You can choose which of those you want to use, but generally its recommended to use the latest, most up-to-date version (15.1 in this case).

However, with the 15.0 version, some Proxmox users were experiencing issues after restarting VMs multiple times, the only way to resolve those was to reboot the whole host machine. If you are affected by this, I recommend that you downgrade to 14.4.

If you have a vGPU qualified GPU, you can use other versions too, because you don't need to patch the driver. However, you still have to make sure they are compatible with your proxmox version and kernel. Also I would not recommend using any older versions unless you have a very specific requirement.

### Obtaining the driver

NVIDIA doesn't let you freely download vGPU drivers like they do with GeForce or normal Quadro drivers, instead you have to download them through the [NVIDIA Licensing Portal](https://nvid.nvidia.com/dashboard/) (see: [https://www.nvidia.com/en-us/drivers/vgpu-software-driver/](https://www.nvidia.com/en-us/drivers/vgpu-software-driver/)). You can sign up for a free evaluation to get access to the download page.

NB: When applying for an eval license, do NOT use your personal email or other email at a free email provider like gmail.com. You will probably have to go through manual review if you use such emails. I have very good experience using a custom domain for my email address, that way the automatic verification usually lets me in after about five minutes.

I've created a small video tutorial to find the right driver version on the NVIDIA Enterprise Portal. In the video I'm downloading the 15.0 driver, if you want a different one just replace 15.0 with the version you want:

![Video Tutorial to find the right driver](downloading_driver.mp4)

After downloading, extract the zip file and then copy the file called `NVIDIA-Linux-x86_64-DRIVERVERSION-vgpu-kvm.run` (where DRIVERVERSION is a string like `525.85.27`) from the `Host_Drivers` folder to your Proxmox host into the `/root/` folder using tools like FileZilla, WinSCP, scp or rsync.

### ⚠️ From here on, I will be using the 15.1 driver, but the steps are the same for other driver versions

For example when I run a command like `chmod +x NVIDIA-Linux-x86_64-525.85.07-vgpu-kvm.run`, you should replace `525.85.07` with the driver version you are using (if you are using a different one). You can get the list of version numbers [here](#nvidia-driver).

Every step where you potentially have to replace the version name will have this warning emoji next to it: ⚠️

> ### Have a vgpu supported card? Read here!
>
> If you don't have a card like the Tesla P4, or any other gpu from [this list](https://docs.nvidia.com/grid/gpus-supported-by-vgpu.html), please continue reading at [Patching the driver](#patching-the-driver)
>
> With a supported gpu, patching the driver is not needed, so you should skip the next section. You can simply install the driver package like this:
>
> ⚠️
> ```bash
> chmod +x NVIDIA-Linux-x86_64-525.85.27-vgpu-kvm.run
> ./NVIDIA-Linux-x86_64-525.85.27-vgpu-kvm.run --dkms
> ```
>
> To finish the installation, reboot the system
> ```bash
> reboot
> ```
>
> Now, skip the following two sections and continue at [Finishing touches](#finishing-touches)

### Patching the driver

Now, on the proxmox host, make the driver executable

⚠️
```bash
chmod +x NVIDIA-Linux-x86_64-525.85.27-vgpu-kvm.run
```

And then patch it

⚠️
```bash
./NVIDIA-Linux-x86_64-525.85.27-vgpu-kvm.run --apply-patch ~/vgpu-proxmox/525.85.27.patch
```
That should output a lot of lines ending with
```
Self-extractible archive "NVIDIA-Linux-x86_64-525.85.27-vgpu-kvm-custom.run" successfully created.
```

You should now have a file called `NVIDIA-Linux-x86_64-525.85.27-vgpu-kvm-custom.run`, that is your patched driver.

### Installing the driver

Now that the required patch is applied, you can install the driver

⚠️
```bash
./NVIDIA-Linux-x86_64-525.85.27-vgpu-kvm-custom.run --dkms
```

The installer will ask you `Would you like to register the kernel module sources with DKMS? This will allow DKMS to automatically build a new module, if you install a different kernel later.`, answer with `Yes`.

Depending on your hardware, the installation could take a minute or two.

If everything went right, you will be presented with this message.
```
Installation of the NVIDIA Accelerated Graphics Driver for Linux-x86_64 (version: 525.85.27) is now complete.
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
Tue Jan 24 20:21:28 2023
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 525.85.27    Driver Version: 525.85.27    CUDA Version: N/A      |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  NVIDIA GeForce ...  On   | 00000000:01:00.0 Off |                  N/A |
| 26%   33C    P8    43W / 260W |     85MiB / 11264MiB |      0%      Default |
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

Another command you can try to see if your card is recognized as being vgpu enabled is this one:
```bash
nvidia-smi vgpu
```

If everything worked right with the unlock, the output should be similar to this:
```
Tue Jan 24 20:21:43 2023
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 525.85.27              Driver Version: 525.85.27                 |
|---------------------------------+------------------------------+------------+
| GPU  Name                       | Bus-Id                       | GPU-Util   |
|      vGPU ID     Name           | VM ID     VM Name            | vGPU-Util  |
|=================================+==============================+============|
|   0  NVIDIA GeForce RTX 208...  | 00000000:01:00.0             |   0%       |
+---------------------------------+------------------------------+------------+
```

However, if you get this output, then something went wrong
```
No supported devices in vGPU mode
```

If any of those commands give the wrong output, you cannot continue. Please make sure to read everything here very carefully and when in doubt, create an issue or join the [discord server](#support) and ask for help there.

## vGPU overrides

Further up we have created the file `/etc/vgpu_unlock/profile_override.toml` and I didn't explain what it was for yet. Using that file you can override lots of parameters for your vGPU instances: For example you can change the maximum resolution, enable/disable the frame rate limiter, enable/disable support for CUDA or change the vram size of your virtual gpus.

If we take a look at the output of `mdevctl types` we see lots of different types that we can choose from. However, if we for example chose `GRID RTX6000-4Q` which gives us 4GB of vram in a VM, we are locked to that type for all of our VMs. Meaning we can only have 4GB VMs, its not possible to mix different types to have one 4GB VM, and two 2GB VMs.

> ### Important notes
>
> Q profiles *can* give you horrible performance in OpenGL applications/games. To fix that, switch to an equivalent A or B profile (for example `GRID RTX6000-4B`)
>
> C profiles (for example `GRID RTX6000-4C`) only work on Linux, don't try using those on Windows, it will not work - at all.
>
> A profiles (for example `GRID RTX6000-4A`) will NOT work on Linux, they only work on Windows.

All of that changes with the override config file. Technically we are still locked to only using one profile, but now its possible to change the vram of the profile on a VM basis so even though we have three `GRID RTX6000-4Q` instances, one VM can have 4GB or vram but we can override the vram size for the other two VMs to only 2GB.

Lets take a look at this example config override file (its in TOML format)
```toml
[profile.nvidia-259]
num_displays = 1          # Max number of virtual displays. Usually 1 if you want a simple remote gaming VM
display_width = 1920      # Maximum display width in the VM
display_height = 1080     # Maximum display height in the VM
max_pixels = 2073600      # This is the product of display_width and display_height so 1920 * 1080 = 2073600
cuda_enabled = 1          # Enables CUDA support. Either 1 or 0 for enabled/disabled
frl_enabled = 1           # This controls the frame rate limiter, if you enable it your fps in the VM get locked to 60fps. Either 1 or 0 for enabled/disabled
framebuffer = 0x74000000
framebuffer_reservation = 0xC000000   # In combination with the framebuffer size
                                      # above, these two lines will give you a VM
                                      # with 2GB of VRAM (framebuffer + framebuffer_reservation = VRAM size in bytes).
                                      # See below for some other sizes

[vm.100]
frl_enabled = 0
# You can override all the options from above here too. If you want to add more overrides for a new VM, just copy this block and change the VM ID
```

There are two blocks here, the first being `[profile.nvidia-259]` and the second `[vm.100]`.
The first one applies the overrides to all VM instances of the `nvidia-259` type (thats `GRID RTX6000-4Q`) and the second one applies its overrides only to one specific VM, that one with the proxmox VM ID `100`.

The proxmox VM ID is the same number that you see in the proxmox webinterface, next to the VM name.

You don't have to specify all parameters, only the ones you need/want. There are some more that I didn't mention here, you can find them by going through the source code of the `vgpu_unlock-rs` repo.

For a simple 1080p remote gaming VM I recommend going with something like this
```toml
[profile.nvidia-259] # choose the profile you want here
num_displays = 1
display_width = 1920
display_height = 1080
max_pixels = 2073600
```

### Common VRAM sizes

Here are some common framebuffer sizes that you might want to use:

- 512MB:
  ```toml
  framebuffer = 0x1A000000
  framebuffer_reservation = 0x6000000
  ```
- 1GB:
  ```toml
  framebuffer = 0x38000000
  framebuffer_reservation = 0x8000000
  ```
- 2GB:
  ```toml
  framebuffer = 0x74000000
  framebuffer_reservation = 0xC000000
  ```
- 3GB:
  ```toml
  framebuffer = 0xB0000000
  framebuffer_reservation = 0x10000000
  ```
- 4GB:
  ```toml
  framebuffer = 0xEC000000
  framebuffer_reservation = 0x14000000
  ```
- 5GB:
  ```toml
  framebuffer = 0x128000000
  framebuffer_reservation = 0x18000000
  ```
- 6GB:
  ```toml
  framebuffer = 0x164000000
  framebuffer_reservation = 0x1C000000
  ```
- 8GB:
  ```toml
  framebuffer = 0x1DC000000
  framebuffer_reservation = 0x24000000
  ```
- 10GB:
  ```toml
  framebuffer = 0x254000000
  framebuffer_reservation = 0x2C000000
  ```
- 12GB:
  ```toml
  framebuffer = 0x2CC000000
  framebuffer_reservation = 0x34000000
  ```
- 16GB:
  ```toml
  framebuffer = 0x3BC000000
  framebuffer_reservation = 0x44000000
  ```
- 20GB:
  ```toml
  framebuffer = 0x4AC000000
  framebuffer_reservation = 0x54000000
  ```
- 24GB:
  ```toml
  framebuffer = 0x59C000000
  framebuffer_reservation = 0x64000000
  ```
- 32GB:
  ```toml
  framebuffer = 0x77C000000
  framebuffer_reservation = 0x84000000
  ```
- 48GB:
  ```toml
  framebuffer = 0xB2D200000
  framebuffer_reservation = 0xD2E00000
  ```

`framebuffer` and `framebuffer_reservation` will always equal the VRAM size in bytes when added together.

### Spoofing your vGPU instance

#### Note: This only works on Windows guests, don't bother trying on Linux.

You can very easily spoof your virtual GPU to a different card, so that you could install normal quadro drivers instead of the GRID drivers that require licensing.

For that you just have to add two lines to the override config. In this example I'm spoofing my Turing based card to a normal RTX 6000 Quadro card:
```toml
[profile.nvidia-259]
# insert all of your other overrides here too
pci_device_id = 0x1E30
pci_id = 0x1E3012BA
```

`pci_device_id` is the pci id from the card you want to spoof to. In my case its `0x1E30` which is the `Quadro RTX 6000/8000`.

`pci_id` can be split in two parts: `0x1E30 12BA`, the first part `0x1E30` has to be the same as `pci_device_id`. The second part is the subdevice id. In my case `12BA` means its a RTX 6000 card and not RTX 8000.

You can get the IDs from [here](https://pci-ids.ucw.cz/read/PC/10de/). Just Ctrl+F and search the card you want to spoof to, then copy the id it shows you on the left and use it for `pci_device_id`.

After doing that, click the same id, it should open a new page where it lists the subsystems. If there are none listed, you must use `0000` as the second value for `pci_id`. But if there are some, you have to select the one you want and use its id as the second value for `pci_id` (see above).

## Important note when spoofing

You have to pick a Quadro Driver from the same driver branch, so in this case R525. Using newer drivers will **NOT WORK** and maybe even make your VM crash.

If you accidentally installed such a driver, its best to either remove the driver completely using DDU or just install a fresh windows VM.

The quadro driver for R525 branch can be found [here (for 527.27)](https://www.nvidia.com/Download/driverResults.aspx/196728/en-us/).

## Drawbacks to spoofing

- You do not have **ANY** CUDA support
- It only works for Windows VMs
- FRL (Framerate limiter) does not work, so no matter what settings you use for `frl_config`, it doesn't apply

## Adding a vGPU to a Proxmox VM

Go to the proxmox webinterface, go to your VM, then to `Hardware`, then to `Add` and select `PCI Device`.
You should be able to choose from a list of pci devices. Choose your GPU there, its entry should say `Yes` in the `Mediated Devices` column.

Now you should be able to also select the `MDev Type`. Choose whatever profile you want, if you don't remember which one you want, you can see the list of all available types with `mdevctl types`.

Finish by clicking `Add`, start the VM and install the required drivers. After installing the drivers you can shut the VM down and remove the virtual display adapter by selecting `Display` in the `Hardware` section and selecting `none (none)`. ONLY do that if you have some other way to access the Virtual Machine like Parsec or Remote Desktop because the Proxmox Console won't work anymore.

Enjoy your new vGPU VM :)

## Common problems

Most problems can be solved by reading the instructions very carefully. For some very common problems, read here:

- The nvidia driver won't install/load
  - If you were using gpu passthrough before, revert **ALL** of the steps you did or start with a fresh proxmox installation. If you run `lspci -knnd 10de:` and see `vfio-pci` under `Kernel driver in use:` then you have to fix that
  - Make sure that you are using a supported kernel version (check `uname -a`)
- My OpenGL performance is absolute garbage, what can I do?
  - Read [here](#important-notes)
- `mdevctl types` doesn't output anything, how to fix it?
  - Make sure that you don't have unlock disabled if you have a consumer gpu ([more information](#have-a-vgpu-supported-card-read-here))
- vGPU doesn't work on my RTX 3080! What to do?
  - [Learn to read](#your-rtx-30xx-or-40xx-will-not-work-at-this-point-in-time)

## Support

If something isn't working, please create an issue or join the [Discord server](https://discord.gg/5rQsSV3Byq) and ask for help in the `#proxmox-support` channel so that the community can help you.

> ### DO NOT SEND ME A DM, I'M NOT YOUR PERSONAL SUPPORT

When asking for help, please describe your problem in detail instead of just saying "vgpu doesn't work". Usually a rough overview over your system (gpu, mainboard, proxmox version, kernel version, ...) and full output of `dmesg` and/or `journalctl --no-pager -b 0 -u nvidia-vgpu-mgr.service` (<-- this only after starting the VM that causes trouble) is helpful.
Please also provide the output of `uname -a` and `cat /proc/cmdline`

## Feed my coffee addiction ☕

If you found this guide helpful and want to support me, please feel free to [buy me a coffee](https://www.buymeacoffee.com/polloloco). Thank you very much!

## Further reading

Thanks to all these people (in no particular order) for making this project possible
- [DualCoder](https://github.com/DualCoder) for his original [vgpu_unlock](https://github.com/DualCoder/vgpu_unlock) repo with the kernel hooks
- [mbilker](https://github.com/mbilker) for the rust version, [vgpu_unlock-rs](https://github.com/mbilker/vgpu_unlock-rs)
- [KrutavShah](https://github.com/KrutavShah) for the [wiki](https://krutavshah.github.io/GPU_Virtualization-Wiki/)
- [HiFiPhile](https://github.com/HiFiPhile) for the [C version](https://gist.github.com/HiFiPhile/b3267ce1e93f15642ce3943db6e60776) of vgpu unlock
- [rupansh](https://github.com/rupansh) for the original [twelve.patch](https://github.com/rupansh/vgpu_unlock_5.12/blob/master/twelve.patch) to patch the driver on kernels >= 5.12
- mbuchel#1878 on the [GPU Unlocking discord](https://discord.gg/5rQsSV3Byq) for [fourteen.patch](https://gist.github.com/erin-allison/5f8acc33fa1ac2e4c0f77fdc5d0a3ed1) to patch the driver on kernels >= 5.14
- [erin-allison](https://github.com/erin-allison) for the [nvidia-smi wrapper script](https://github.com/erin-allison/nvidia-merged-arch/blob/d2ce752cd38461b53b7e017612410a3348aa86e5/nvidia-smi)
- LIL'pingu#9069 on the [GPU Unlocking discord](https://discord.gg/5rQsSV3Byq) for his patch to nop out code that NVIDIA added to prevent usage of drivers with a version 460 - 470 with consumer cards

If I forgot to mention someone, please create an issue or let me know otherwise.

## Contributing
Pull requests are welcome (factual errors, amendments, grammar/spelling mistakes etc).