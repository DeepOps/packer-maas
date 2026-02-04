# DGX OS 5 Packer Template for MAAS

> **Note:** For newer DGX systems (H100, H200, B200, B300) and newer DGX A100 systems running DGX OS 7 (Ubuntu 24.04), use the [dgxos7](../dgxos7/) template instead. This template is for legacy DGX-1, DGX-2, and older DGX A100 systems running DGX OS 5 (Ubuntu 20.04).

## Prerequisites

Install dependencies (Ubuntu 18.04 and 20.04)

```sh
sudo apt -y install qemu qemu-utils qemu-system-x86
```

Install Packer

```sh
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt-get update && sudo apt-get install packer
```

Go to Packer template directory for DGX OS 5, i.e

```sh
cd packer-maas/dgxos5
```

Download DGX OS somewhere (i.e. /work)

Generate checksum of image

```sh
sha256sum /work/DGXOS-5.0.0-2020-10-01-18-07-44.iso
```
Build image

```sh
# When building the image, you can pass variables to the `build` command or edit the dgxos5.json file
# At minimum update the ISO location and SHA sum (dgxos5_iso and dgxos5_sha256sum)
# i.e.
#    "variables":
#        {
#            "platform": "dgx1",
#            "dgxos5_iso": "/work/DGXOS-5.0.0-2020-10-01-18-07-44.iso",
#            "dgxos5_sha256sum": "6e5c7ba2024640b3f23ec8681c15c8ccf8997a23da91c7e9d4eacf73bb564bee"
#        },
sudo packer build dgxos5.json

# Optionally, instead of modifying config file:
sudo packer build -var 'dgxos5_iso=/path/to/dgx_iso' -var 'dgxos5_sha256sum=<dgx_os_iso_sha256_sum>' dgxos5.json

# Available platforms: dgx1, dgx2, dgx_a100

# For more verbosity set `PACKER_LOG=1`, i.e sudo PACKER_LOG=1 build ...
```

Go grab a beer while the image builds...

Add image to MAAS:

```sh
# Be sure to substitute the proper platform name, i.e. dgx1, dgx2, dgx_a100
maas $PROFILE boot-resources create name='custom/dgx1-5.0' title='NVIDIA DGX-1 5.0' architecture='amd64/generic' filetype='tgz' content@=dgxos5.tar.gz
```

If using MAAS 3.0 or later, you will need to use the following command, which adds the base_image argument:

```sh
# Be sure to substitute the proper platform name, i.e. dgx1, dgx2, dgx_a100
maas $PROFILE boot-resources create name='custom/dgx_a100-5.1' title='NVIDIA DGX-A100 5.1' architecture='amd64/generic' filetype='tgz' base_image=ubuntu/focal content@=dgxos5.tar.gz
```

Ensure that you are authenticated via the commandline for your MaaS user. If you are not, you may encounter an error like this:

```sh
argument COMMAND: invalid choice: 'boot-resources' (choose from 'login', 'logout', 'list', 'refresh', 'init', 'config', 'status', 'migrate', 'reconfigure-supervisord', 'apikey', 'configauth', 'createadmin', 'changepassword')
```

To authenticate:

```sh
maas login $PROFILE http://<MAAS url>
#upon executing the above command, the following prompt will present
API key (leave empty for anonymous access):
```

At the prompt, enter an API key, which can be acquired with the following command:

```sh
maas apikey --username $PROFILE
```

Re-run the boot-resources command once authenticated.

Boot machines in EFI mode

In maas, create an EFI partition in addition to other partitions, i.e:
```sh
# NAME    SIZE     FILESYSTEM   MOUNT POINT
sda-part1 511.7 MB fat32        /boot/efi
sda-part2 63.9 GB  ext4         /
```

Troubleshooting:

```sh
# Sometimes nbd devices don't get unmounted between builds with packer
# so run as root:
umount /dev/nbd*

# between builds, remove artifacts:
sudo rm -rf output-qemu/ dgxos5.tar.gz
```

If you see an error like this from the boot-resources command:

```sh
[Errno 13] Permission denied: './dgxos5.tar.gz'
```

You can update the ownership of the tar.gz file, like this:

```sh
sudo chown <target user>:<target group> dgxos5.tar.gz
```

TODO Next:
* kernel parameters in MAAS (w/ tags)
* document generate one image per DGX type


<!--

## debug stuff
# to manually test qemu steps for debug purposes:
mkdir ~/output-qemu
qemu-img create -f qcow2 output-qemu/packer-qemu 9G
qemu-img convert -O qcow2 output-qemu/packer-qemu output-qemu/packer-qemu.convert
/usr/bin/qemu-system-x86_64 -name packer-qemu -boot once=d -drive file=~/output-qemu/packer-qemu,if=virtio,cache=writeback,discard=ignore,format=qcow2 -drive file=/work/DGXOS-5.0.0-2020-10-01-18-07-44.iso,index=0,media=cdrom -serial stdio -m 20
48M -vnc 0.0.0.0:81 -machine type=pc,accel=tcg -netdev user,id=user.0 -device virtio-net,netdev=user.0
# connect to VNC on: <machine>:5981 (no password)

# if you're connected to the VNC console, you can ctrl-alt-F2 to get another TTY, log in with the user 'root' and no password
# ubuntu kernel cmd args/boot params:
#  https://manpages.ubuntu.com/manpages/focal/en/man7/kernel-command-line.7.html
#  https://manpages.ubuntu.com/manpages/focal/en/man7/bootparam.7.html
#  https://www.kernel.org/doc/html/latest/admin-guide/kernel-parameters.html
```
> https://discourse.maas.io/t/creating-a-custom-ubuntu-image/1652

```sh
sudo mount /work/DGXOS-5.0.0-2020-10-01-18-07-44.iso /mnt/dgxiso-5.0

mkdir /work/dgxos-5
cd /work/dgxos-5
unsquashfs /mnt/dgxiso-5.0/live/filesystem.squashfs

mkdir /tmp/work
cd /tmp/work
sudo tar xf /work/dgxos-5/squashfs-root/curtin/ubuntu-20.04-server-cloudimg-amd64-root.tar.xz

sudo mount -o bind /proc /tmp/work/proc
sudo mount -o bind /dev /tmp/work/dev
sudo mount -o bind /sys /tmp/work/sys
sudo mv /tmp/work/etc/resolv.conf /tmp/work/etc/resolv.conf.bak
sudo cp /etc/resolv.conf /tmp/work/etc/
sudo chroot /tmp/work /bin/bash
```

missing dpkg in dgx os root fs (exists in focal current img):
  python3-pexpect
  python3-ptyprocess

Existing MAAS/DGX-2 stuff:

  docs: https://dgxdownloads.nvidia.com/custhelp/dgx2/Knowledgebase/DGX-MAAS-Setup-Guide.pdf
  curtin file: https://dgxdownloads.nvidia.com/custhelp/dgx2/Knowledgebase/curtin-dgx-19.07.1


Steps:
* download ISO to MAAS machine and mount
* unsquash root image
* run nginx container serving up squash fs contents
* during deployment, curtin file pulls down squash fs contents to target machine
* curtin file runs 'preseed.sh', etc. to do DGX-specific install

Host setup:
```sh
cd /work/dgxos-5/squashfs-root
#docker run -it --rm -d -p 8080:80 --name web -v ${PWD}:/usr/share/nginx/html nginx
docker run -it --rm -d -p 8080:80 --name web -v ${PWD}:/usr/share/nginx/html jorgeandrada/nginx-autoindex
```

Curtin script:
```sh
# remote machine as root:
wget -P /curtin -nd -nH -r --no-parent maas.lab:8080/curtin/
wget -P /usr/local/sbin/nv_scripts -nd -nH -r --no-parent maas.lab:8080/usr/local/sbin/nv_scripts/
wget -o /bin/live-medium-eject maas.lab:8080/bin/live-medium-eject

sudo chmod +x /usr/local/sbin/nv_scripts/*
sudo chmod +x /bin/live-medium-eject

cd /curtin
sed -i 's_$(cat /proc/cmdline)_"$@"_g' preseed.sh
sed -i 's/parse_cmdline$/parse_cmdline "$@"/g' preseed.sh
bash ./preseed.sh # dgx force-platform=dgx1 force-curtin=${PWD}/dgx1-curtin.yaml
```

```sh
# tmp stuff
"force-curtin=http://{{ .HTTPIP }}:{{ .HTTPPort }}/curtin.yaml ",
"force-platform=dgx-vbox ",
                "offwhendone ",
            # newer dgx os 5 image:
            "dgxos5_iso": "/scratch/DGXOS-5.0.0-2020-10-23-18-34-11.iso",
            "dgxos5_sha256sum": "2eefe51fea356642cbe087db6bac802179db6290db2fc192e81a4ed67b5ee30b"
            # regular dgx os 5 image:
            "dgxos5_iso": "/scratch/DGXOS-5.0.0-2020-10-01-18-07-44.iso",
            "dgxos5_sha256sum": "6e5c7ba2024640b3f23ec8681c15c8ccf8997a23da91c7e9d4eacf73bb564bee"
            # egx test image: ( has different grub menu entries)
            "dgxos5_iso": "/scratch/egxtest-5.0.0-2020-10-23-14-40-24.iso",
            "dgxos5_sha256sum": "d7de20b8922fc7c3cf319afebe1a1b51a96f6af3989f12149445831e39098649"

            "qemuargs": [
                [ "-serial", "stdio" ],
                [ "-smbios", "type=0,uefi=on" ],
                [ "-smp", "8"]
            ]
# foo
using preseed, get on console, stop sshd service, run: dhclient ens3
```
-->
