# DGX OS 7 Packer Template for MAAS

> **EULA Notice:** By building and deploying images with this template, you accept the NVIDIA DGX Software License Agreement. The build process automatically accepts the EULA on your behalf.

## Overview

This directory contains a Packer template for building NVIDIA DGX OS 7 images deployable via MAAS (Metal as a Service). DGX OS 7 is based on Ubuntu 24.04 LTS with kernel 6.8 and includes NVIDIA-optimized configurations for the latest DGX hardware platforms.

## Supported Platforms

| Platform    | Hardware       | DGX OS 7 Support |
|-------------|----------------|------------------|
| dgx_h100    | DGX H100       | ✓                |
| dgx_h200    | DGX H200       | ✓                |
| dgx_b200    | DGX B200       | ✓                |
| dgx_b300    | DGX B300       | ✓                |
| dgx_a100    | DGX A100       | ✓                |

**Note:** For older DGX-1, DGX-2, or legacy DGX A100 systems running DGX OS 5, use the [dgxos5](../dgxos5/) template instead.

## Prerequisites

### Software Requirements

- **Packer** 1.7.0 or later
- **QEMU/KVM** with UEFI support (OVMF)
- **Linux build environment** (Debian/Ubuntu recommended)
- **MAAS** 3.0 or later (for deployment)

### Build Dependencies

Install the required packages for image creation:

```bash
# Debian/Ubuntu
apt-get install -y qemu-system-x86 qemu-utils ovmf packer \
    nbdkit libnbd-bin fuse2fs
```

### DGX OS 7 ISO

DGX OS 7 ISOs are available to customers with an NVIDIA Enterprise Support account:

1. Log in to the [NVIDIA Enterprise Support Portal](https://nvid.nvidia.com/)
2. Navigate to: **Software Downloads** → **DGX OS**
3. Download the latest DGX OS 7.x ISO
4. Verify the ISO integrity with the provided SHA256 checksum

## Building the Image

### Using Make (Recommended)

```bash
export DGXOS7_ISO_PATH=/path/to/DGXOS-7.x.x.iso
export DGXOS7_SHA256SUM=$(sha256sum $DGXOS7_ISO_PATH | cut -d' ' -f1)

# Build (defaults to dgx_h100 platform)
sudo make
```

### Direct Packer Command

```bash
export DGXOS7_ISO_PATH=/path/to/DGXOS-7.x.x.iso
export DGXOS7_SHA256SUM=$(sha256sum $DGXOS7_ISO_PATH | cut -d' ' -f1)

sudo PACKER_LOG=1 packer build dgxos7.json
```

### Build Output

After a successful build (~60 minutes), you will have:
- **dgxos7.tar.gz** - Deployable MAAS image (~2.5GB)

## Uploading to MAAS

**Note:** The MAAS CLI requires the image file to be in your current working directory. Copy the image to your home directory and run the command from there:

```bash
# Copy image to MAAS server home directory
scp dgxos7.tar.gz maas@<maas-server>:~/

# SSH to MAAS server and upload
ssh maas@<maas-server>
PROFILE=admin

maas $PROFILE boot-resources create \
    name='custom/dgxos7' \
    title='NVIDIA DGX OS 7' \
    architecture='amd64/generic' \
    filetype='tgz' \
    base_image='ubuntu/noble' \
    content@=dgxos7.tar.gz
```

To replace an existing image, first delete the old one:

```bash
# Find the image ID
maas $PROFILE boot-resources read | jq '.[] | select(.name | contains("dgxos7")) | {id, name}'

# Delete the old image
maas $PROFILE boot-resource delete <id>

# Upload the new image (as above)
```

## Deployment

### UEFI Boot Requirement

DGX OS 7 **requires UEFI boot**. Ensure your MAAS machines are configured for UEFI in the machine's Configuration → Firmware settings.

### Deploy via CLI

```bash
maas $PROFILE machine deploy <system_id> \
    osystem='custom' \
    distro_series='dgxos7'
```

## How It Works

This template uses the DGX OS 7 installer's native autoinstall mechanism:

1. **Boot**: GRUB is edited to add kernel parameters (`force-ai`, `force-platform`, `nooemconfig`, etc.)
2. **Download**: The DGX installer's `preseed.sh` downloads our custom autoinstall config (`http/packer-ai.yaml`)
3. **Install**: Our config sets up partitions, installs NVIDIA packages (with EULA bypassed), creates an `ubuntu` user, enables SSH
4. **Complete**: After install, Packer connects via SSH, installs curtin hooks, resets cloud-init state for MAAS, and creates the tarball

### Key Boot Parameters

- `force-ai=http://...` - Points to custom autoinstall config
- `force-platform=dgx_h100` - Platform type (can be changed via `platform` variable)
- `force-bootdisk=vda` - Specifies boot disk
- `nooemconfig` - Disables OEM config packages that require interactive EULA acceptance
- `no-mlnx-fw-update` - Skips Mellanox firmware updates
- `ip=dhcp` - Enables networking for HTTP config fetch

## Customization

### Modifying the Autoinstall Config

Edit `http/packer-ai.yaml` to customize:
- Partition layout
- User accounts and passwords
- SSH configuration
- Late-command scripts

### Adjusting VM Resources

Edit `dgxos7.json`:
- **Disk size**: `disk_size` (default: 12G)
- **Memory**: `memory` (default: 4096 MB)
- **CPUs**: `-smp` in `qemuargs` (default: 8)

## Troubleshooting

### SSH Connection Timeout

If Packer times out waiting for SSH:
1. Set `"headless": false` in dgxos7.json to enable VNC
2. Connect to VNC to observe the installation
3. Check that the autoinstall config was downloaded (look for preseed.sh output)

### YAML Parse Errors

If you see YAML errors in the installer:
1. Validate your config: `python3 -c "import yaml; yaml.safe_load(open('http/packer-ai.yaml'))"`
2. Ensure no CHANGE_* placeholders remain (we bypass preseed.sh's substitution)

### Wrong Partition in Tarball

For GPT layouts, root is on partition 2 (partition 1 is EFI). The `ROOT_PARTITION=2` variable in the post-processor ensures the correct partition is extracted.

## Technical Notes

### DGX OS 7 vs DGX OS 5

| Feature           | DGX OS 5        | DGX OS 7        |
|-------------------|-----------------|-----------------|
| Base OS           | Ubuntu 20.04    | Ubuntu 24.04    |
| Installer         | Curtin          | Subiquity       |
| Config delivery   | force-curtin=   | force-ai=       |
| Kernel            | 5.4.x           | 6.8.x           |

### Why force-ai Instead of Standard Autoinstall

DGX OS 7 includes an embedded autoinstall config that takes precedence over standard cloud-init datasources. The `force-ai` parameter tells the DGX installer's `preseed.sh` to download and use our custom config instead.

## License

This Packer template follows the same license as the parent packer-maas repository.
