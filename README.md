# Tesla P100 on Ubuntu 26.04 with ESXi 8 Passthrough

This repository documents the complete process of getting an NVIDIA Tesla P100 GPU working inside an Ubuntu VM running on VMware ESXi 8 using PCI passthrough.

The guide includes:

- ESXi passthrough configuration
- VM advanced configuration parameters
- Ubuntu installation
- NVIDIA driver installation
- CUDA installation
- Docker + NVIDIA runtime setup
- Troubleshooting common issues
- Validation/testing procedures

---

# Hardware Used

| Component | Details |
|---|---|
| Hypervisor | VMware ESXi 8 |
| GPU | NVIDIA Tesla P100 PCIe 16GB |
| Guest OS | Ubuntu Server 26.04 |
| VM Firmware | EFI |
| CUDA | 12.x |
| Driver Branch | 535/550/570 |

---

# Important ESXi VM Settings

These VMX parameters are critical for large BAR Tesla cards like the P100:

```ini
pciPassthru.use64bitMMIO = "TRUE"
pciPassthru.64bitMMIOSizeGB = "64"
hypervisor.cpuid.v0 = "FALSE"
```

Recommended additional settings:

- Reserve all guest memory
- Disable Secure Boot
- EFI firmware only
- Add BOTH:
  - GPU PCI device
  - GPU Audio PCI device

---

# Step-by-Step Guide

## 1. Enable Passthrough in ESXi

1. Navigate to:
   Host → Manage → Hardware → PCI Devices

2. Enable passthrough for:
   - Tesla P100 GPU
   - Tesla P100 Audio Device

3. Reboot ESXi host

Detailed instructions:
- `docs/02-esxi-passthrough.md`

---

## 2. Configure VM

Before powering on the VM:

- Set firmware to EFI
- Reserve all guest memory
- Add PCI devices
- Add VMX advanced parameters

---

## 3. Install Ubuntu

Install Ubuntu Server 24.04 or Ubuntu 26.04.

After installation:

```bash
sudo apt update && sudo apt upgrade -y
```

---

## 4. Disable Nouveau

```bash
sudo nano /etc/modprobe.d/blacklist-nouveau.conf
```

Add:

```conf
blacklist nouveau
options nouveau modeset=0
```

Then:

```bash
sudo update-initramfs -u
sudo reboot
```

---

## 5. Verify GPU Detection

```bash
lspci | grep -i nvidia
```

Expected output should show:

```text
NVIDIA Corporation GP100GL [Tesla P100 PCIe 16GB]
```

---

## 6. Install NVIDIA Drivers

Recommended:

```bash
sudo ubuntu-drivers autoinstall
```

Or install manually:

```bash
sudo apt install nvidia-driver-550 -y
```

Reboot:

```bash
sudo reboot
```

Validate:

```bash
nvidia-smi
```

---

## 7. Install CUDA

Add NVIDIA CUDA repository:

```bash
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/cuda-keyring_1.1-1_all.deb

sudo dpkg -i cuda-keyring_1.1-1_all.deb

sudo apt update

sudo apt install cuda -y
```

Verify:

```bash
nvcc --version
```

---

## 8. Docker NVIDIA Runtime

Install NVIDIA container toolkit:

```bash
sudo apt install nvidia-container-toolkit -y
```

Configure runtime:

```bash
sudo nvidia-ctk runtime configure --runtime=docker
```

Restart Docker:

```bash
sudo systemctl restart docker
```

Test:

```bash
docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu24.04 nvidia-smi
```

---

# Common Problems

## `nvidia-smi` fails

Usually caused by:

- Nouveau not disabled
- Secure Boot enabled
- Missing MMIO configuration
- Missing memory reservation
- Wrong driver version
- GPU audio device not passed through

---

## VM Boot Hang

Try:

```ini
pciPassthru.disableFLR = "TRUE"
```

Also ensure:

- EFI firmware enabled
- Full memory reservation enabled

---

## GPU Visible in `lspci` but Drivers Fail

Usually fixed by:

- `hypervisor.cpuid.v0 = "FALSE"`
- Disabling Secure Boot
- Using 64-bit MMIO

---

# Validation

## GPU Status

```bash
nvidia-smi
```

## CUDA Test

```bash
deviceQuery
```

## PyTorch Test

```python
import torch
print(torch.cuda.is_available())
```

---

# References

## NVIDIA Documentation

- https://docs.nvidia.com/
- https://developer.nvidia.com/cuda-downloads

## ESXi + GPU Passthrough Resources

- https://forums.developer.nvidia.com/t/dell-r730-and-tesla-p100-cuda-and-driver-install-information/50781
- https://serverfault.com/questions/958239/pci-at-nvida-tesla-p-100-in-shared-pass-through-mode-is-disabled
- https://ubuntu.com/server/docs/how-to/graphics/gpu-virtualization-with-qemu-kvm/
- https://www.dell.com/support/kbdoc/en-ie/000106925/how-to-configure-a-gpu-using-discrete-device-assignment-dda-on-ubuntu-guest-operating-system

## Community References

- https://www.reddit.com/r/homelab/comments/1rb47ab/nvidia_tesla_p40_drivers_on_ubuntu_server_2404/
- https://www.reddit.com/r/esxi/comments/1knrpy7/
- https://www.reddit.com/r/PleX/comments/10jfo3m/

## Original Discussion

- https://claude.ai/chat/d1b4f2c2-bdb9-417d-8c6c-ff3fac9e03b5

---

# Useful Commands

## Check GPU

```bash
lspci | grep -i nvidia
```

## Check Driver

```bash
nvidia-smi
```

## Check CUDA

```bash
nvcc --version
```

## Check Kernel Modules

```bash
lsmod | grep nvidia
```

## Verify the installation
```bashmarlon@ai-server:~$ mokutil --sb-state
sudo modprobe nvidia
nvidia-smi
SecureBoot disabled
```
[sudo: authenticate] Password:
Thu May  7 23:24:23 2026
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.142                Driver Version: 580.142        CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  Tesla P100-PCIE-16GB           Off |   00000000:03:00.0 Off |                    0 |
| N/A   51C    P0             29W /  250W |       0MiB /  16384MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
---

# License

MIT
