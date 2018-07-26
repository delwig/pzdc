Gitian building
================

*Setup instructions for a gitian build of PZDC using a Debian VM or physical system.*

Gitian is the deterministic build process that is used to build the PZDC
Core executables. It provides a way to be reasonably sure that the
executables are really built from source on GitHub. It also makes sure that
the same, tested dependencies are used and statically built into the executable.

Multiple developers build the source code by following a specific descriptor
("recipe"), cryptographically sign the result, and upload the resulting signature.
These results are compared and only if they match, the build is accepted and uploaded
to pzdc.org.

More independent gitian builders are needed, which is why I wrote this
guide. It is preferred to follow these steps yourself instead of using someone else's
VM image to avoid 'contaminating' the build.

Table of Contents
------------------

- [Create a new VirtualBox VM](#create-a-new-virtualbox-vm)
- [Connecting to the VM](#connecting-to-the-vm)
- [Setting up Debian for gitian building](#setting-up-debian-for-gitian-building)
- [Installing gitian](#installing-gitian)
- [Setting up gitian images](#setting-up-gitian-images)
- [Getting and building the inputs](#getting-and-building-the-inputs)
- [Building PZDC](#building-pzdc)
- [Building an alternative repository](#building-an-alternative-repository)
- [Signing externally](#signing-externally)
- [Uploading signatures](#uploading-signatures)

Preparing the Gitian builder host
---------------------------------

The first step is to prepare the host environment that will be used to perform the Gitian builds.
This guide explains how to set up the environment, and how to start the builds.

Debian Linux was chosen as the host distribution because it has a lightweight install (in contrast to Ubuntu) and is readily available.
Any kind of virtualization can be used, for example:
- [VirtualBox](https://www.virtualbox.org/), covered by this guide
- [KVM](http://www.linux-kvm.org/page/Main_Page)
- [LXC](https://linuxcontainers.org/), see also [Gitian host docker container](https://github.com/gdm85/tenku/tree/master/docker/gitian-bitcoin-host/README.md).

You can also install on actual hardware instead of using virtualization.

# Setup Debian virtual machine on VirtualBox

Table of Contents
------------------

- [Create a new VirtualBox VM](#create-a-new-virtualbox-vm)
- [Connecting to the VM](#connecting-to-the-vm)

Create a new VirtualBox VM
---------------------------
In the VirtualBox GUI click "New" and choose the following parameters in the wizard:

![](gitian-building/create_new_vm_debian.png)

- Type: Linux, Debian (64-bit)

![](gitian-building/create_vm_memsize.png)

- Memory Size: at least 3000MB, anything less and the build might not complete.

![](gitian-building/create_vm_hard_disk.png)

- Hard Disk: Create a virtual hard disk now

![](gitian-building/create_vm_hard_disk_file_type.png)

- Hard Disk file type: Use the default, VDI (VirtualBox Disk Image)

![](gitian-building/create_vm_storage_physical_hard_disk.png)

- Storage on physical hard disk: Dynamically Allocated

![](gitian-building/create_vm_file_location_size.png)

- File location and size: at least 40GB; as low as 20GB *may* be possible, but better to err on the safe side
- Click `Create`

After creating the VM, we need to configure it.

- Click the `Settings` button, then go to `System` tab and `Processor` sub-tab. Increase the number of processors to the number of cores on your machine if you want builds to be faster.

![](gitian-building/system_settings.png)

- Go to the `Network` tab. Adapter 1 should be attached to `NAT`.

![](gitian-building/network_settings.png)

- Click `Advanced`, then `Port Forwarding`. We want to set up a port through which we can reach the VM to get files in and out.
- Create a new rule by clicking the plus icon.

![](gitian-building/port_forwarding_rules.png)

- Set up the new rule the following way:
  - Name: `SSH`
  - Protocol: `TCP`
  - Leave Host IP empty
  - Host Port: `22222`
  - Leave Guest IP empty
  - Guest Port: `22`

- Click `Ok` twice to save.

Get the [Debian 8.x net installer](https://cdimage.debian.org/cdimage/archive/8.10.0/amd64/iso-cd/debian-8.10.0-amd64-netinst.iso) (a more recent minor version should also work, see also [Debian Network installation](https://www.debian.org/CD/netinst/)).
This DVD image can be [validated](https://www.debian.org/CD/verify) using a SHA256 hashing tool, for example on
Unixy OSes by entering the following in a terminal:

    echo "896cc42998edf65f1db4eba83581941fb2a584f2214976432b841af96b17ccda  debian-8.10.0-amd64-netinst.iso" | sha256sum -c
    # (must return OK)

Replace `sha256sum` with `shasum` on OSX.

Then start the VM. On the first launch you will be asked for a CD or DVD image. Choose the downloaded ISO.

![](gitian-building/select_startup_disk_debian.png)

Installing Debian
------------------

This section will explain how to install Debian on the newly created VM.

- Choose the non-graphical installer.  We do not need the graphical environment; it will only increase installation time and disk usage.

![](gitian-building/debian_install_1_boot_menu.png)

**Note**: Navigating in the Debian installer:
To keep a setting at the default and proceed, just press `Enter`.
To select a different button, press `Tab`.

- Choose locale and keyboard settings (doesn't matter, you can just go with the defaults or select your own information)

![](gitian-building/debian_install_2_select_a_language.png)
![](gitian-building/debian_install_3_select_location.png)
![](gitian-building/debian_install_4_configure_keyboard.png)

- The VM will detect network settings using DHCP, this should all proceed automatically
- Configure the network:
  - Hostname `debian`.
  - Leave domain name empty.

![](gitian-building/debian_install_5_configure_the_network.png)
![](gitian-building/debian_install_6_domain_name.png)

- You can leave the root password empty. Otherwise, enter it twice and remember it for later.

![](gitian-building/debian_install_6a_set_up_root_password.png)

- Name the new user `gitianuser` (the full name doesn't matter, you can leave it empty)
- Set the account username as `gitianuser`

![](gitian-building/debian_install_7_set_up_user_fullname.png)
![](gitian-building/debian_install_8_set_up_username.png)

- Choose a user password and enter it twice (remember it for later)

![](gitian-building/debian_install_9_user_password.png)

- The installer will set up the clock using a time server; this process should be automatic
- Set up the clock: choose a time zone (depends on the locale settings that you picked earlier; specifics don't matter)  

![](gitian-building/debian_install_10_configure_clock.png)

- Disk setup
  - Partitioning method: Guided - Use the entire disk

![](gitian-building/debian_install_11_partition_disks.png)

  - Select disk to partition: SCSI1 (0,0,0)

![](gitian-building/debian_install_12_choose_disk.png)

  - Partition Disks -> *All files in one partition*

![](gitian-building/all_files_in_one_partition.png)

  - Finish partitioning and write changes to disk -> *Yes* (`Tab`, `Enter` to select the `Yes` button)

![](gitian-building/debian_install_14_finish.png)
![](gitian-building/debian_install_15_write_changes.png)

- The base system will be installed, this will take a minute or so
- Choose a mirror (any will do)

![](gitian-building/debian_install_16_choose_a_mirror.png)

- Enter proxy information (unless you are on an intranet, leave this empty)

![](gitian-building/debian_install_18_proxy_settings.png)

- Wait a bit while 'Select and install software' runs
- Participate in popularity contest -> *No*
- Choose software to install. We need just the base system.
- Make sure only 'SSH server' and 'Standard System Utilities' are checked
- Uncheck 'Debian Desktop Environment' and 'Print Server'

![](gitian-building/debian_install_19_software_selection.png)

- Install the GRUB boot loader to the master boot record? -> Yes

![](gitian-building/debian_install_20_install_grub.png)

- Device for boot loader installation -> ata-VBOX_HARDDISK

![](gitian-building/debian_install_21_install_grub_bootloader.png)

- Installation Complete -> *Continue*
- After installation, the VM will reboot and you will have a working Debian VM. Congratulations!

![](gitian-building/debian_install_22_finish_installation.png)

Connecting to the VM
----------------------

After the VM has booted you can connect to it using SSH, and files can be copied from and to the VM using a SFTP utility.
Connect to `localhost`, port `22222` (or the port configured when installing the VM).
On Windows you can use [putty](http://www.chiark.greenend.org.uk/~sgtatham/putty/download.html) and [WinSCP](http://winscp.net/eng/index.php).

For example, to connect as `gitianuser` from a Linux command prompt use

    $ ssh gitianuser@localhost -p 22222
    The authenticity of host '[localhost]:22222 ([127.0.0.1]:22222)' can't be established.
    RSA key fingerprint is ae:f5:c8:9f:17:c6:c7:1b:c2:1b:12:31:1d:bb:d0:c7.
    Are you sure you want to continue connecting (yes/no)? yes
    Warning: Permanently added '[localhost]:22222' (RSA) to the list of known hosts.
    gitianuser@localhost's password: (enter gitianuser password configured during install)

    The programs included with the Debian GNU/Linux system are free software;
    the exact distribution terms for each program are described in the
    individual files in /usr/share/doc/*/copyright.

    Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
    permitted by applicable law.
    gitianuser@debian:~$

Use `sudo` to execute commands as root.

Optional - Easier login to the VM
---------------------------------

You'll need to generate an SSH key, e.g. by following the instructions under "Generating a new SSH key" [here](https://help.github.com/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent).

After that, login to the VM and enter:

```bash
mkdir .ssh
```

On your machine edit or create `~/.ssh/config` and add:

```bash
Host gitian
    HostName localhost
    Port 22222
    User gitianuser
```

Open a new terminal tab and enter:

```bash
scp ~/.ssh/id_rsa.pub gitian:.ssh/authorized_keys
```

Next time you need to login to the VM, just use: `ssh gitian`

Setting up Debian for gitian building
--------------------------------------

In this section we will be setting up the Debian installation for Gitian building.

First we need to log in as `root` to set up dependencies and make sure that our
user can use the sudo command. Type/paste the following in the terminal:

```bash
apt-get install make git ruby sudo apt-cacher-ng qemu-utils debootstrap lxc python-cheetah parted kpartx bridge-utils
adduser debian sudo
```

When you get a colorful screen with a question about the 'LXC directory', just
go with the default (`/var/lib/lxc`).

Then set up LXC and the rest with the following, which is a complex jumble of settings and workarounds:

```bash
# the version of lxc-start in Debian 7.4 needs to run as root, so make sure
# that the build script can exectute it without providing a password
echo "%sudo ALL=NOPASSWD: /usr/bin/lxc-start" > /etc/sudoers.d/gitian-lxc
# add cgroup for LXC
echo "cgroup  /sys/fs/cgroup  cgroup  defaults  0   0" >> /etc/fstab
# make /etc/rc.local script that sets up bridge between guest and host
echo '#!/bin/sh -e' > /etc/rc.local
echo 'brctl addbr br0' >> /etc/rc.local
echo 'ifconfig br0 10.0.3.2/24 up' >> /etc/rc.local
echo 'exit 0' >> /etc/rc.local
# make sure that USE_LXC is always set when logging in as debian,
# and configure LXC IP addresses
echo 'export USE_LXC=1' >> /home/debian/.profile
echo 'export GITIAN_HOST_IP=10.0.3.2' >> /home/debian/.profile
echo 'export LXC_GUEST_IP=10.0.3.5' >> /home/debian/.profile
reboot
```

At the end the VM is rebooted to make sure that the changes take effect. The steps in this
section need only to be performed once.

Installing gitian
------------------

Re-login as the user `debian` that was created during installation.
The rest of the steps in this guide will be performed as that user.

There is no `python-vm-builder` package in Debian, so we need to install it from source ourselves,

```bash
wget http://archive.ubuntu.com/ubuntu/pool/universe/v/vm-builder/vm-builder_0.12.4+bzr489.orig.tar.gz
echo "ec12e0070a007989561bfee5862c89a32c301992dd2771c4d5078ef1b3014f03  vm-builder_0.12.4+bzr489.orig.tar.gz" | sha256sum -c
# (verification -- must return OK)
tar -zxvf vm-builder_0.12.4+bzr489.orig.tar.gz
cd vm-builder-0.12.4+bzr489
sudo python setup.py install
cd ..
```

**Note**: When sudo asks for a password, enter the password for the user *debian* not for *root*.

Clone the git repositories for pzdc and gitian and then checkout the pzdc version that you want to build.

```bash
git clone https://github.com/devrandom/gitian-builder.git
git clone https://github.com/pzdc-project/pzdc.git
cd pzdc
git checkout v${VERSION}
cd ..
```

**Note**: if you've installed Gitian before May 16, 2015, please update to the latest version, see https://github.com/devrandom/gitian-builder/issues/86


Setting up gitian images
-------------------------

Gitian needs virtual images of the operating system to build in.
Currently this is Ubuntu Precise for x86_64.
These images will be copied and used every time that a build is started to
make sure that the build is deterministic.
Creating the images will take a while, but only has to be done once.

Execute the following as user `debian`:

```bash
cd gitian-builder
bin/make-base-vm --lxc --arch amd64 --suite trusty
```

There will be a lot of warnings printed during build of the images. These can be ignored.

**Note**: When sudo asks for a password, enter the password for the user *debian* not for *root*.

**Note**: Repeat this step when you have upgraded to a newer version of Gitian.

**Note**: if you get the error message *"bin/make-base-vm: mkfs.ext4: not found"* during this process you have to make the following change in file *"gitian-builder/bin/make-base-vm"* at line 117:
```bash
# mkfs.ext4 -F $OUT-lxc
/sbin/mkfs.ext4 -F $OUT-lxc # (some Gitian environents do NOT find mkfs.ext4. Some do...)
```

Getting and building the inputs
--------------------------------

Follow the instructions in [doc/release-process.md](release-process.md) in the pzdc repository
under 'Fetch and build inputs' to install sources which require manual intervention. Also follow
the next step: 'Seed the Gitian sources cache', which will fetch all necessary source files allowing
for gitian to work offline.

Building PZDC
----------------

To build PZDC (for Linux, OSX and Windows) just follow the steps under 'perform
gitian builds' in [doc/release-process.md](release-process.md) in the pzdc repository.

This may take a long time as it also builds the dependencies needed for each descriptor.
These dependencies will be cached after a successful build to avoid rebuilding them when possible.

At any time you can check the package installation and build progress with

```bash
tail -f var/install.log
tail -f var/build.log
```

Output from `gbuild` will look something like

```bash
    Initialized empty Git repository in /home/debian/gitian-builder/inputs/pzdc/.git/
    remote: Reusing existing pack: 35606, done.
    remote: Total 35606 (delta 0), reused 0 (delta 0)
    Receiving objects: 100% (35606/35606), 26.52 MiB | 4.28 MiB/s, done.
    Resolving deltas: 100% (25724/25724), done.
    From https://github.com/pzdc-project/pzdc
    ... (new tags, new branch etc)
    --- Building for precise x86_64 ---
    Stopping target if it is up
    Making a new image copy
    stdin: is not a tty
    Starting target
    Checking if target is up
    Preparing build environment
    Updating apt-get repository (log in var/install.log)
    Installing additional packages (log in var/install.log)
    Grabbing package manifest
    stdin: is not a tty
    Creating build script (var/build-script)
    lxc-start: Connection refused - inotify event with no name (mask 32768)
    Running build script (log in var/build.log)
```

Building an alternative repository
-----------------------------------

If you want to do a test build of a pull on GitHub it can be useful to point
the gitian builder at an alternative repository, using the same descriptors
and inputs.

For example:
```bash
URL=https://github.com/crowning-pzdc/pzdc.git
COMMIT=b616fb8ef0d49a919b72b0388b091aaec5849b96
./bin/gbuild --commit pzdc=${COMMIT} --url pzdc=${URL} ../pzdc/contrib/gitian-descriptors/gitian-linux.yml
./bin/gbuild --commit pzdc=${COMMIT} --url pzdc=${URL} ../pzdc/contrib/gitian-descriptors/gitian-win.yml
./bin/gbuild --commit pzdc=${COMMIT} --url pzdc=${URL} ../pzdc/contrib/gitian-descriptors/gitian-osx.yml
```

Signing externally
-------------------

If you want to do the PGP signing on another device that's also possible; just define `SIGNER` as mentioned
and follow the steps in the build process as normal.

    gpg: skipped "crowning-": secret key not available

When you execute `gsign` you will get an error from GPG, which can be ignored. Copy the resulting `.assert` files
in `gitian.sigs` to your signing machine and do

```bash
    gpg --detach-sign ${VERSION}-linux/${SIGNER}/pzdc-build.assert
    gpg --detach-sign ${VERSION}-win/${SIGNER}/pzdc-build.assert
    gpg --detach-sign ${VERSION}-osx/${SIGNER}/pzdc-build.assert
```

This will create the `.sig` files that can be committed together with the `.assert` files to assert your
gitian build.

Uploading signatures (not yet implemented)
---------------------

In the future it will be possible to push your signatures (both the `.assert` and `.assert.sig` files) to the
[pzdc/gitian.sigs](https://github.com/pzdc-project/gitian.sigs/) repository, or if that's not possible to create a pull
request.
There will be an official announcement when this repository is online.
