# NVIDIA vGPU with the GRID 15.0 driver

Two days ago, NVIDIA released their latest enterprise GRID driver. I created a patch that allows the use of most consumer GPUs for vGPU. One notable exception from that list is every officially unsupported Ampere GPU and GPUs from the Ada Lovelace generation.

This guide and all my tests were done on a RTX 2080 Ti which is based on the Turing architechture.

### This tutorial assumes you are using a clean install of Proxmox 7.3, or ymmv when using an existing installation. Make sure to always have backups!

This guide should work for other linux systems with a recent kernel (5.15 to 5.19) but I have only tested it on the current proxmox version.
If you are not using proxmox, you have to adapt some parts of this tutorial to work for your distribution.

> # Are you upgrading from a previous version of this guide?
>
> If you are upgrading from a previous version of this guide, you should uninstall the old driver first:
> ```
> nvidia-uninstall
> ```
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

As of the time of this writing (December 2022), the latest available GRID driver is 15.0 with vGPU driver version 525.60.12. You can check for the latest version [here](https://docs.nvidia.com/grid/). I cannot guarantee that newer versions would work without additional patches, the patch in this guide works **ONLY** on 15.0 (525.60.12).

### Obtaining the driver

NVIDIA doesn't let you freely download vGPU drivers like they do with GeForce or normal Quadro drivers, instead you have to download them through the [NVIDIA Licensing Portal](https://nvid.nvidia.com/dashboard/) (see: [https://www.nvidia.com/en-us/drivers/vgpu-software-driver/](https://www.nvidia.com/en-us/drivers/vgpu-software-driver/)). You can sign up for a free evaluation to get access to the download page.

NB: When applying for an eval license, do NOT use your personal email or other email at a free email provider like gmail.com. You will probably have to go through manual review if you use such emails. I have very good experience using a custom domain for my email address, that way the automatic verification usually lets me in after about five minutes.

The file you are looking for is called `NVIDIA-GRID-Linux-KVM-525.60.12-525.60.13-527.41.zip`, you can get it from the download portal by downloading version 14.3 for `Linux KVM`.

For those who want to find the file somewhere else, here are some checksums :)
```
sha1: e4147e1dcebfc5459759ea013b56bca1d30f3578
md5: 0e2be7de643b99a62a1cca6ca37fd1ee
```

After downloading, extract that and copy the file `NVIDIA-Linux-x86_64-525.60.12-vgpu-kvm.run` to your Proxmox host into the `/root/` folder
```bash
scp NVIDIA-Linux-x86_64-525.60.12-vgpu-kvm.run root@pve:/root/
```

> ### Have a vgpu supported card? Read here!
>
> If you don't have a card like the Tesla P4, or any other gpu from [this list](https://docs.nvidia.com/grid/gpus-supported-by-vgpu.html), please continue reading at [Patching the driver](#patching-the-driver)
>
> With a supported gpu, patching the driver is not needed, so you should skip the next section. You can simply install the driver package like this:
> ```bash
> chmod +x NVIDIA-Linux-x86_64-525.60.12-vgpu-kvm.run
> ./NVIDIA-Linux-x86_64-525.60.12-vgpu-kvm.run --dkms
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
```bash
chmod +x NVIDIA-Linux-x86_64-525.60.12-vgpu-kvm.run
```

And then patch it
```bash
./NVIDIA-Linux-x86_64-525.60.12-vgpu-kvm.run --apply-patch ~/vgpu-proxmox/525.60.12.patch
```
That should output a lot of lines ending with
```
Self-extractible archive "NVIDIA-Linux-x86_64-525.60.12-vgpu-kvm-custom.run" successfully created.
```

You should now have a file called `NVIDIA-Linux-x86_64-525.60.12-vgpu-kvm-custom.run`, that is your patched driver.

### Installing the driver

Now that the required patch is applied, you can install the driver
```bash
./NVIDIA-Linux-x86_64-525.60.12-vgpu-kvm-custom.run --dkms
```

The installer will ask you `Would you like to register the kernel module sources with DKMS? This will allow DKMS to automatically build a new module, if you install a different kernel later.`, answer with `Yes`.

Depending on your hardware, the installation could take a minute or two.

If everything went right, you will be presented with this message.
```
Installation of the NVIDIA Accelerated Graphics Driver for Linux-x86_64 (version: 525.60.12) is now complete.
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
Sun Dec  4 12:54:59 2022
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 525.60.12    Driver Version: 525.60.12    CUDA Version: N/A      |
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
Sun Dec  4 12:55:09 2022
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 525.60.12              Driver Version: 525.60.12                 |
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

## vGPU overrides

Further up we have created the file `/etc/vgpu_unlock/profile_override.toml` and I didn't explain what it was for yet. Using that file you can override lots of parameters for your vGPU instances: For example you can change the maximum resolution, enable/disable the frame rate limiter, enable/disable support for CUDA or change the vram size of your virtual gpus.

If we take a look at the output of `mdevctl types` we see lots of different types that we can choose from. However, if we for example chose `GRID RTX6000-4Q` which gives us 4GB of vram in a VM, we are locked to that type for all of our VMs. Meaning we can only have 4GB VMs, its not possible to mix different types to have one 4GB VM, and two 2GB VMs.

> ### Important note
>
> C profiles (for example `GRID RTX6000-4C`) only work on Linux, don't try using those on Windows, it will not work - at all.

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
framebuffer = 0x76000000  # VRAM size for the VM. In this case its 2GB
                          # Other options:
                          # 1GB: 0x3B000000
                          # 2GB: 0x76000000
                          # 3GB: 0xB1000000
                          # 4GB: 0xEC000000
                          # 8GB: 0x1D8000000
                          # 16GB: 0x3B0000000
                          # These numbers may not be accurate for you, but you can always calculate the right number like this:
                          # The amount of VRAM in your VM = `framebuffer` + `framebuffer_reservation`

[mdev.00000000-0000-0000-0000-000000000100]
frl_enabled = 0
# You can override all the options from above here too. If you want to add more overrides for a new VM, just copy this block and change the UUID
```

There are two blocks here, the first being `[profile.nvidia-259]` and the second `[mdev.00000000-0000-0000-0000-000000000100]`.
The first one applies the overrides to all VM instances of the `nvidia-259` type (thats `GRID RTX6000-4Q`) and the second one applies its overrides only to one specific VM, that one with the uuid `00000000-0000-0000-0000-000000000100`.

You don't have to specify all parameters, only the ones you need/want. There are some more that I didn't mention here, you can find them by going through the source code of the `vgpu_unlock-rs` repo.

For a simple 1080p remote gaming VM I recommend going with something like this
```toml
[profile.nvidia-259] # choose the profile you want here
num_displays = 1
display_width = 1920
display_height = 1080
max_pixels = 2073600
```

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

## Adding a vGPU to a Proxmox VM

There is only one thing you have to do from the commandline: Open the VM config file and give the VM a uuid.

For that you need your VM ID, in this example I'm using `1000`.

```bash
nano /etc/pve/qemu-server/<VM-ID>.conf
```

So with the VM ID 1000, I have to do this:

```bash
nano /etc/pve/qemu-server/1000.conf
```

In that file, you have to add a new line at the end:
```
args: -uuid 00000000-0000-0000-0000-00000000XXXX
```

You have to replace `XXXX` with your VM ID. With my 1000 ID I have to use this line:
```
args: -uuid 00000000-0000-0000-0000-000000001000
```

Save and exit from the editor. Thats all you have to do from the terminal.

Now go to the proxmox webinterface, go to your VM, then to `Hardware`, then to `Add` and select `PCI Device`.
You should be able to choose from a list of pci devices. Choose your GPU there, its entry should say `Yes` in the `Mediated Devices` column.

Now you should be able to also select the `MDev Type`. Choose whatever profile you want, if you don't remember which one you want, you can see the list of all available types with `mdevctl types`.

Finish by clicking `Add`, start the VM and install the required drivers. After installing the drivers you can shut the VM down and remove the virtual display adapter by selecting `Display` in the `Hardware` section and selecting `none (none)`. ONLY do that if you have some other way to access the Virtual Machine like Parsec or Remote Desktop because the Proxmox Console won't work anymore.

Enjoy your new vGPU VM :)

## Support

If something isn't working, please create an issue or join the [Discord server](https://discord.gg/5rQsSV3Byq) and ask for help in the `#proxmox-support` channel.

When asking for help, please describe your problem in detail instead of just saying "vgpu doesn't work". Usually a rough overview over your system (gpu, mainboard, proxmox version, kernel version, ...) and full output of `dmesg` and/or `journalctl --no-pager -b 0 -u nvidia-vgpu-mgr.service` (<-- this only after starting the VM that causes trouble) is helpful.
Please also provide the output of `uname -a` and `cat /proc/cmdline`

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