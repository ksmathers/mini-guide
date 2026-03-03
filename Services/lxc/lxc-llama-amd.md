---
title: lxc-llama-amd
description: 
published: true
date: 2026-01-17T00:00:00.000Z
tags: 
editor: markdown
dateCreated: 2026-01-17T00:00:00.000Z
---

# Llama with AMD GPU Acceleration in LXC Container (Mini Guide)

This guide covers running Llama with GPU acceleration in an LXC container on Proxmox 9.1 hosted on HP Z2 Mini G1A (AMD Strix Halo 395, 128GB RAM).

## 🖥️ Hardware Specifications

- **System**: HP Z2 Mini G1A
- **Processor**: AMD Strix Halo 395 (with integrated RDNA 3.5 graphics)
- **Memory**: 128GB RAM
- **Hypervisor**: Proxmox VE 9.1

## ⚙️ Part 1: BIOS Configuration

### 1. Access BIOS/UEFI Settings

**Boot into BIOS**:
- Power on the HP Z2 Mini G1A
- Press **F10** repeatedly during boot to enter BIOS
- Navigate using arrow keys and Enter

### 2. Enable Virtualization Features

Navigate to **Advanced** → **System Options** → **Virtualization Technology**:

```
✓ CPU Virtualization (AMD-V/AMD-Vi): Enabled (permanently enabled on the HP Z2 Mini G1A)
✓ Enable DMA Protection (HP's label for IOMMU)
```

**Note**: SR-IOV and SMT options are not available in the BIOS on this system. AMD-V is permanently enabled.

### 3. Graphics Configuration

Navigate to **Advanced** → **Built-In Device Options**:

```
Dedicated Graphics Memory: 32GB (default - this is optimal for AI workloads)
Primary Display: Auto or PCIe/PEG (if discrete GPU present)
```

**Note**: The 32GB default allocation for the integrated AMD Strix Halo GPU is excellent for running Llama models and should not be changed.

### 4. Power Management

Navigate to **Power** → **Hardware Power Management**:

```
✓ Enable Intel SpeedStep (AMD equivalent)
✓ Enable C-States
Runtime Power Management: Enable
```

### 5. Save and Exit

- Press **F10** to save changes
- Confirm and reboot

## ✅ Part 2: Proxmox Host Configuration

### 6. Verify IOMMU is Enabled

SSH into your Proxmox host:

```bash
# Check IOMMU support
dmesg | grep -i iommu

# Expected output should show:
# AMD-Vi: IOMMU enabled
# AMD-Vi: Device-Table[...] 
```

### 7. Configure GRUB for IOMMU

Edit GRUB configuration:

```bash
nano /etc/default/grub
```

Modify the `GRUB_CMDLINE_LINUX_DEFAULT` line:

```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet amdgpu.gttsize=131072"
```

**Kernel parameters explained**:
- `amdgpu.gttsize=131072` - Sets GTT (Graphics Translation Table) size to 128GB, allowing the integrated GPU to address all system memory. This is ideal for the Strix Halo APU with 128GB RAM, enabling the GPU to work with very large AI models that exceed the 32GB dedicated VRAM.

**Note for LXC vs VM**: Unlike VMs which require IOMMU/passthrough (`amd_iommu=on`, `iommu=pt`), LXC containers share the host kernel and access GPU devices directly through device binding. The only critical parameter here is `amdgpu.gttsize` which configures the AMD driver itself to utilize system memory.

Update GRUB and reboot:

```bash
update-grub
grub-install
reboot
```

### 8. Verify GPU Devices

After reboot, check GPU devices:

```bash
# List the AMD Strix Halo integrated GPU device
lspci | grep 8060S

# Check render devices
ls -la /dev/dri/

# Expected output:
# /dev/dri/card1
# /dev/dri/renderD128
```

### 9. Install AMD GPU Drivers on Proxmox Host

```bash
# Update package lists
apt update

# Install required dependencies for repository management
apt install -y wget gnupg2

# Add ROCm repository GPG key
mkdir -p /etc/apt/keyrings
wget https://repo.radeon.com/rocm/rocm.gpg.key -O - | gpg --dearmor | tee /etc/apt/keyrings/rocm.gpg > /dev/null

# Add ROCm repository (ROCm 7.1.1 - latest release)
# Using noble (Ubuntu 24.04) repo as it's compatible with Debian 13 Trixie
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/rocm.gpg] https://repo.radeon.com/rocm/apt/7.1.1 noble main" | tee /etc/apt/sources.list.d/rocm.list

# Configure APT pinning to prefer ROCm repository over Debian packages
cat > /etc/apt/preferences.d/rocm-pin << 'EOF'
Package: *
Pin: origin repo.radeon.com
Pin-Priority: 600

Package: rocm* rocminfo
Pin: origin repo.radeon.com
Pin-Priority: 1001
EOF

# Update package lists with new repository and pinning in place
apt update

# Install AMD ROCm drivers and dependencies
# Using the meta-package which handles dependencies correctly
apt install -y rocm-hip-sdk

# Alternatively, if the above fails, install individual packages:
# apt install -y rocm-core rocm-hip-libraries rocm-smi-lib

# Load kernel modules
modprobe amdgpu

# Verify driver loaded
lsmod | grep amdgpu

# Verify KFD (Kernel Fusion Driver) is active - required for ROCm compute
# Note: KFD is usually built-in to the kernel, not a separate module
dmesg | grep -i kfd
# Should show output like:
# kfd kfd: amdgpu: Allocated ... bytes on gart
# kfd kfd: amdgpu: added device 1002:1586

# Ensure amdgpu module loads on boot
echo "amdgpu" >> /etc/modules

# Verify KFD device node exists and has correct permissions
ls -la /dev/kfd
# Should show: crw-rw---- 1 root render 234, 0
```

### 10. Check GPU Information

```bash
# Check GPU details with rocm-smi
rocm-smi

# Check with lspci for more details
lspci -vnn | grep -A 12 'VGA\|Display'
```

## ✅ Part 3: LXC Container Creation

### 11. Download Container Template

```bash
# List available templates
pveam update
pveam available | grep ubuntu

# Download Ubuntu 24.04 template
pveam download local ubuntu-24.04-standard_24.04-2_amd64.tar.zst
```

### 12. Create LXC Container

Using Proxmox Web UI:

1. Navigate to **Datacenter** → **[Your Node]**
2. Click **Create CT** button
3. Configure as follows:

**General Tab**:
```
Node: [Your Proxmox node]
CT ID: 100 (or next available)
Hostname: llama-gpu
Unprivileged container: UNCHECKED (must be privileged for GPU access)
Password: [Set a secure password]
```

**Template Tab**:
```
Storage: local
Template: ubuntu-24.04-standard_24.04-2_amd64.tar.zst
```

**Root Disk Tab**:
```
Storage: local-lvm (or your preferred storage)
Disk size: 100 GB
```

**CPU Tab**:
```
Cores: 16 (adjust based on your needs, Strix Halo has up to 16 cores)
CPU limit: unlimited
CPU units: 1024
```

**Memory Tab**:
```
Memory (MiB): 65536 (64GB - adjust based on model size)
Swap (MiB): 8192
```

**Network Tab**:
```
Bridge: vmbr0
IPv4: DHCP or Static (configure as needed)
IPv6: DHCP or Static (configure as needed)
```

Click **Finish** but **DO NOT START** the container yet.

### 13. Configure Container for GPU Access (CLI)

On the Proxmox host, edit the container configuration:

```bash
# Replace 100 with your CT ID
CTID=100
nano /etc/pve/lxc/$CTID.conf
```

Add the following lines at the end:

```bash
# GPU Passthrough Configuration
lxc.cgroup2.devices.allow: c 226:0 rwm
lxc.cgroup2.devices.allow: c 226:1 rwm
lxc.cgroup2.devices.allow: c 226:128 rwm
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir 0 0

# KFD (Kernel Fusion Driver) device access - CRITICAL for ROCm compute
lxc.cgroup2.devices.allow: c 234:0 rwm
lxc.mount.entry: /dev/kfd dev/kfd none bind,optional,create=file 0 0

# Additional device access for ROCm
lxc.cgroup2.devices.allow: c 235:* rwm
lxc.mount.entry: /dev/dri/card1 dev/dri/card1 none bind,optional,create=file 0 0
lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file 0 0

# Increase shared memory for AI workloads
lxc.mount.entry: tmpfs dev/shm tmpfs rw,nosuid,nodev,size=32G,create=dir 0 0

# AppArmor profile (allows device access)
lxc.apparmor.profile: unconfined
```

**Note**: The Strix Halo integrated GPU appears as `/dev/dri/card1` (not card0). Make sure the configuration above matches your actual device.

Save and exit (Ctrl+X, Y, Enter).

### 14. Start the Container

```bash
# Start the container
pct start $CTID

# Verify it's running
pct status $CTID

# Enter the container
pct enter $CTID
```

## ✅ Part 4: Container Setup and GPU Configuration

### 15. Update Container and Install Base Packages

Inside the container:

```bash
# Update package lists
apt update && apt upgrade -y

# Install essential packages
apt install -y \
  build-essential \
  cmake \
  git \
  wget \
  curl \
  python3 \
  python3-pip \
  python3-venv \
  vim \
  htop \
  tmux

# Optional: Install additional build optimization tools
# ccache speeds up recompilation (useful if you rebuild llama.cpp frequently)
# libssl-dev enables SSL/TLS support in llama-server
apt install -y ccache libssl-dev
```

### 16. Install AMD ROCm in Container

```bash
# Add ROCm repository (ROCm 7.1.1 - latest release)
mkdir -p /etc/apt/keyrings
wget https://repo.radeon.com/rocm/rocm.gpg.key -O - | gpg --dearmor | tee /etc/apt/keyrings/rocm.gpg > /dev/null

echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/rocm.gpg] https://repo.radeon.com/rocm/apt/7.1.1 noble main" | tee /etc/apt/sources.list.d/rocm.list

# Configure APT pinning to prefer ROCm repository over Ubuntu packages
cat > /etc/apt/preferences.d/rocm-pin << 'EOF'
Package: *
Pin: origin repo.radeon.com
Pin-Priority: 600

Package: rocm* rocminfo
Pin: origin repo.radeon.com
Pin-Priority: 1001
EOF

# Update package lists with new repository and pinning in place
apt update

# Install ROCm packages including development libraries for building llama.cpp
apt install -y rocm-hip-runtime rocm-opencl-runtime rocm-smi rocm-dev hipblas-dev rocblas-dev

# Add user to render and video groups
usermod -aG render,video root

# IMPORTANT: pct enter does NOT properly initialize supplementary groups
# You must either:
# Option 1: Use newgrp to activate the render group in current shell
newgrp render

# Option 2: Or use su to start a new login shell
# su - root

# Note: After running newgrp render, you'll be in a new shell with render group active
```

### 17. Verify GPU Access in Container

After running `newgrp render`:

```bash
# Verify group membership is active
id
# Should show render as your current group: gid=993(render)
# Note: pct enter doesn't initialize groups properly, that's why we use newgrp

# Alternative: Check with getent to confirm user is in render group
getent group render
# Should show: render:x:993:root

# Check if GPU devices are accessible
ls -la /dev/dri/

# Should show:
# /dev/dri/card1
# /dev/dri/renderD128

# Check /dev/kfd permissions
ls -la /dev/kfd
# Should show: crw-rw---- 1 root render ...

# Test with rocm-smi
/opt/rocm/bin/rocm-smi

# Test with rocminfo (this will confirm GPU compute access)
/opt/rocm/bin/rocminfo

# Expected: GPU information displayed with no "Operation not permitted" errors
# Note: You may see a warning about "low-power state" - this is normal when idle.
# The GPU will automatically increase power when running inference.
```

### 18. Set ROCm Environment Variables

```bash
# Add to .bashrc
cat >> ~/.bashrc << 'EOF'

# ROCm Environment
export ROCM_PATH=/opt/rocm
export PATH=$ROCM_PATH/bin:$PATH
export LD_LIBRARY_PATH=$ROCM_PATH/lib:$LD_LIBRARY_PATH
export HSA_OVERRIDE_GFX_VERSION=11.5.1
EOF

# Reload environment
source ~/.bashrc
```

## ✅ Part 5: Install Llama.cpp with ROCm Support

### 19. Clone and Build llama.cpp

```bash
# Create workspace directory
mkdir -p /opt/llama
cd /opt/llama

# Clone llama.cpp repository
git clone https://github.com/ggerganov/llama.cpp.git
cd llama.cpp

# Build with ROCm/HIP support
mkdir build
cd build

cmake .. \
  -DGGML_HIP=ON \
  -DCMAKE_C_COMPILER=/opt/rocm/llvm/bin/clang \
  -DCMAKE_CXX_COMPILER=/opt/rocm/llvm/bin/clang++ \
  -DAMDGPU_TARGETS="gfx1151;gfx1150;gfx1100" \
  -DCMAKE_BUILD_TYPE=Release

# Target architecture explained:
# gfx1151 - Native target for AMD Strix Halo (Radeon 8060S, RDNA 3.5)
# gfx1150 - Fallback for Strix Point variants
# gfx1100 - Additional fallback for broader compatibility

# Compile (use -j to speed up with parallel compilation)
cmake --build . --config Release -j$(nproc)
```

### 20. Verify llama.cpp Build

```bash
# Check if binaries were created
ls -lh bin/

# Should see executables like:
# llama-cli
# llama-server
# llama-bench
```

## ✅ Part 6: Download and Run Llama Models

### 21. Download a Model

```bash
# Create models directory
mkdir -p /opt/llama/models
cd /opt/llama/models

# Download a model (example: Llama 3.2 3B Instruct GGUF)
# Visit https://huggingface.co for more models
wget https://huggingface.co/bartowski/Llama-3.2-3B-Instruct-GGUF/resolve/main/Llama-3.2-3B-Instruct-Q5_K_M.gguf

# Or use huggingface-cli for easier downloads
# Option 1: Install using pipx (recommended for command-line tools)
apt install -y pipx
pipx install huggingface-hub
pipx ensurepath
# Reload your shell or run: source ~/.bashrc

# Download using huggingface-cli
hf download \
  bartowski/Llama-3.2-3B-Instruct-GGUF \
  Llama-3.2-3B-Instruct-Q5_K_M.gguf \
  --local-dir /opt/llama/models

# Note: Files will be downloaded directly to the specified directory

# Option 2: Use a virtual environment (alternative approach)
# python3 -m venv /opt/llama/venv
# source /opt/llama/venv/bin/activate
# pip install huggingface-hub
# huggingface-cli download ...
# deactivate
```

### 22. Test Model with GPU Acceleration

Before running the model, verify your environment:

```bash
# Verify ROCm environment variables are set
echo $ROCM_PATH
echo $HSA_OVERRIDE_GFX_VERSION
echo $LD_LIBRARY_PATH

# If not set, reload your bashrc or set them manually:
source ~/.bashrc

# Verify GPU is accessible
ls -la /dev/dri/
rocm-smi

# Check if llama-cli can see HIP/ROCm libraries
ldd ./bin/llama-cli | grep -i hip
ldd ./bin/llama-cli | grep -i rocm
```

If the environment is correct, run the model:

```bash
# Navigate to llama.cpp build directory
cd /opt/llama/llama.cpp/build

# Run a simple prompt with GPU offloading
./bin/llama-cli \
  -m /opt/llama/models/Llama-3.2-3B-Instruct-Q5_K_M.gguf \
  -p "What is the capital of France?" \
  -n 128 \
  -ngl 99

# Parameters explained:
# -m: model file path
# -p: prompt text
# -n: number of tokens to generate
# -ngl: number of GPU layers (99 = offload all possible layers)
```

**Troubleshooting "no ROCm-capable device is detected"**:

If you see this error, check the following:

```bash
# 1. Verify environment variables
env | grep -E 'ROCM|HSA|HIP'

# 2. Check if GPU devices are accessible
ls -la /dev/dri/
# Should show card1 and renderD128

# 3. Test ROCm directly
rocm-smi
# Should show GPU information

# 4. Check if the binary was built with HIP support
./bin/llama-cli --version
# Look for "HIP" or "ROCm" in the output

# 5. Verify ROCm libraries are found
export LD_LIBRARY_PATH=/opt/rocm/lib:$LD_LIBRARY_PATH
ldconfig -p | grep rocm

# 6. If still not working, rebuild with verbose output
cd /opt/llama/llama.cpp/build
rm -rf *
cmake .. \
  -DGGML_HIP=ON \
  -DCMAKE_C_COMPILER=/opt/rocm/llvm/bin/clang \
  -DCMAKE_CXX_COMPILER=/opt/rocm/llvm/bin/clang++ \
  -DAMDGPU_TARGETS="gfx1151;gfx1150;gfx1100" \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_VERBOSE_MAKEFILE=ON
cmake --build . --config Release -j$(nproc)
```

### 23. Monitor GPU Usage During Inference

Open a second terminal to the container:

```bash
# On Proxmox host
pct enter 100

# Inside container, monitor GPU
watch -n 1 rocm-smi
```

You should see GPU utilization increase during inference.

## ✅ Part 7: Setup Llama Server (Optional)

### 24. Create Systemd Service for Llama Server

```bash
# Create systemd service file
cat > /etc/systemd/system/llama-server.service << 'EOF'
[Unit]
Description=Llama.cpp Server with ROCm GPU Acceleration
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/llama/llama.cpp/build
Environment="ROCM_PATH=/opt/rocm"
Environment="LD_LIBRARY_PATH=/opt/rocm/lib"
Environment="HSA_OVERRIDE_GFX_VERSION=11.5.1"
ExecStart=/opt/llama/llama.cpp/build/bin/llama-server \
  -m /opt/llama/models/Llama-3.2-3B-Instruct-Q5_K_M.gguf \
  --host 0.0.0.0 \
  --port 8080 \
  -ngl 99 \
  -c 8192 \
  --threads 8
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

# Reload systemd
systemctl daemon-reload

# Enable and start service
systemctl enable llama-server
systemctl start llama-server

# Check status
systemctl status llama-server
```

### 25. Test API Server

```bash
# Test the server endpoint
curl http://localhost:8080/v1/models

# Send a completion request
curl http://localhost:8080/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "What is artificial intelligence?",
    "n_predict": 128,
    "temperature": 0.7
  }'
```

## ✅ Part 8: Install Ollama (Alternative)

### 26. Install Ollama with ROCm Support

```bash
# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Set ROCm environment for Ollama
export OLLAMA_COMPUTE=rocm
export HSA_OVERRIDE_GFX_VERSION=11.5.1

# Add to system environment
cat >> /etc/environment << 'EOF'
OLLAMA_COMPUTE=rocm
HSA_OVERRIDE_GFX_VERSION=11.5.1
EOF
```

### 27. Pull and Run Models with Ollama

```bash
# Start Ollama service
systemctl start ollama
systemctl enable ollama

# Pull a model
ollama pull llama3.2:3b

# Run interactive session
ollama run llama3.2:3b

# Test with a prompt
ollama run llama3.2:3b "Explain quantum computing in simple terms"
```

### 28. Verify GPU Usage with Ollama

```bash
# Run model and monitor GPU in another terminal
ollama run llama3.2:3b

# In another terminal:
rocm-smi --showuse
```

## ✅ Part 8b: Install Lemonade Server (Alternative)

Lemonade Server is another alternative that supports both ROCm (GPU) and potentially NPU acceleration in the future.

### 29. Install Lemonade Server

```bash
# Download Lemonade Server package
cd /tmp
wget https://github.com/lemonade-sdk/lemonade/releases/latest/download/lemonade-server-minimal_9.1.3_amd64.deb

# Install the package
dpkg -i lemonade-server-minimal_9.1.3_amd64.deb

# Update PCI IDs for GPU detection
update-pciids

# Set ROCm environment for Lemonade
export HSA_OVERRIDE_GFX_VERSION=11.5.1

# Add to .bashrc if not already present
grep -q "HSA_OVERRIDE_GFX_VERSION" ~/.bashrc || echo "export HSA_OVERRIDE_GFX_VERSION=11.5.1" >> ~/.bashrc
```

### 30. Test Lemonade Server

```bash
# Check available options
lemonade-server -h

# Run a model with ROCm backend (GPU acceleration)
# Note: You'll need to have a GGUF model downloaded (see Part 6, step 21)
lemonade-server run /opt/llama/models/Llama-3.2-3B-Instruct-Q5_K_M.gguf --llamacpp rocm

# Alternative: Use vulkan backend
# lemonade-server run /opt/llama/models/Llama-3.2-3B-Instruct-Q5_K_M.gguf --llamacpp vulkan
```

**Note**: As of January 2026, Lemonade Server on Linux primarily supports GPU acceleration via ROCm. NPU support may be added in future releases, making this setup forward-compatible for when NPU acceleration becomes available on Linux.

### 31. Verify GPU Usage with Lemonade

```bash
# In another terminal, monitor GPU usage
watch -n 1 rocm-smi
```

## ✅ Part 9: Optimization and Performance Tuning

### 32. Optimize Container Resources

If you experience performance issues, adjust container resources:

```bash
# On Proxmox host, edit container config
nano /etc/pve/lxc/100.conf

# Adjust CPU pinning for better performance
# Add after existing configuration:
lxc.cgroup2.cpuset.cpus: 0-15  # Use all 16 cores
```

### 33. Optimize Model Loading

```bash
# For large models, ensure enough RAM is allocated
# Use mmap for memory efficiency

# Run with mmap enabled
./bin/llama-cli \
  -m /opt/llama/models/your-model.gguf \
  --mmap 1 \
  -ngl 99 \
  -p "Your prompt here"
```

### 34. Benchmark Performance

```bash
# Run benchmark to test GPU performance
cd /opt/llama/llama.cpp/build

./bin/llama-bench \
  -m /opt/llama/models/Llama-3.2-3B-Instruct-Q5_K_M.gguf \
  -ngl 99 \
  -p 512,1024,2048 \
  -n 128,256,512

# This will test different prompt and generation lengths
```

## 📊 Part 10: Monitoring and Troubleshooting

### 35. Setup Monitoring Scripts

```bash
# Create GPU monitoring script
cat > /usr/local/bin/gpu-monitor.sh << 'EOF'
#!/bin/bash
while true; do
  clear
  echo "=== GPU Status ==="
  rocm-smi
  echo ""
  echo "=== Memory Usage ==="
  free -h
  echo ""
  echo "=== CPU Usage ==="
  top -bn1 | head -20
  sleep 2
done
EOF

chmod +x /usr/local/bin/gpu-monitor.sh

# Run monitoring
/usr/local/bin/gpu-monitor.sh
```

### 36. Common Troubleshooting

**Issue: GPU not detected in container**
```bash
# Verify devices on Proxmox host
ls -la /dev/dri/

# Verify container config has correct permissions
cat /etc/pve/lxc/100.conf | grep -A 5 "cgroup2.devices"

# Restart container
pct stop 100
pct start 100
```

**Issue: ROCm initialization fails ("Operation not permitted" on /dev/kfd)**
```bash
# This is usually caused by group membership not being active, NOT missing KFD
# First, verify KFD is actually working on Proxmox host:
exit  # Exit container

# On host, verify KFD is active (built into kernel)
dmesg | grep -i kfd
# Should show: "kfd kfd: amdgpu: added device ..."

# If KFD messages are missing, reload amdgpu:
modprobe -r amdgpu
modprobe amdgpu
dmesg | grep -i kfd | tail -5

# Verify /dev/kfd exists
ls -la /dev/kfd

# Re-enter container
pct enter 101

# CRITICAL: Activate render group (pct enter doesn't do this automatically)
newgrp render

# Verify group is active
id
# Should now show gid=993(render)

# Test ROCm access
rocminfo
# Should work without "Operation not permitted"
```

**Issue: Group membership not active in container**
```bash
# pct enter doesn't initialize supplementary groups properly
# Use newgrp to activate render group:
newgrp render

# Verify with id command
id
# Should show gid=993(render)

# Now test ROCm access
rocminfo
```

**Issue: General ROCm environment issues**
```bash
# Check ROCm environment
echo $ROCM_PATH
echo $HSA_OVERRIDE_GFX_VERSION

# Reinstall ROCm runtime
apt install --reinstall rocm-hip-runtime

# Check kernel modules on host
lsmod | grep amdgpu
```

**Issue: Out of memory errors**
```bash
# Check available memory
free -h

# Reduce model layers offloaded to GPU
# Use -ngl with lower value (e.g., -ngl 30 instead of -ngl 99)

# Or use smaller quantized model (Q4 instead of Q5)
```

**Issue: Slow inference speed**
```bash
# Ensure GPU is being used
rocm-smi --showuse

# Check if layers are offloaded
# Look for "llama_model_load: offloaded" messages in llama output

# Increase batch size if memory allows
./bin/llama-cli -m model.gguf -b 512 -ngl 99
```

### 37. View Logs

```bash
# Llama server logs (if using systemd service)
journalctl -u llama-server -f

# Ollama logs
journalctl -u ollama -f

# Container logs from Proxmox host
pct exec 100 -- journalctl -n 100
```

## 🎯 Part 11: Additional Configuration

### 38. Setup Web UI (Optional)

Install Open WebUI for a user-friendly interface:

```bash
# Install Docker in the container
apt install -y docker.io
systemctl start docker
systemctl enable docker

# Run Open WebUI container
docker run -d \
  --name open-webui \
  -p 3000:8080 \
  -v open-webui:/app/backend/data \
  --add-host=host.docker.internal:host-gateway \
  -e OLLAMA_BASE_URL=http://host.docker.internal:11434 \
  ghcr.io/open-webui/open-webui:main
```

Access at: http://[container-ip]:3000

### 39. Configure Firewall Rules

```bash
# On Proxmox host, allow access to LXC container ports
# If using Proxmox firewall, add rules through Web UI

# Or use iptables in container
apt install -y iptables

# Allow llama-server port
iptables -A INPUT -p tcp --dport 8080 -j ACCEPT

# Allow Ollama API port
iptables -A INPUT -p tcp --dport 11434 -j ACCEPT

# Save rules
iptables-save > /etc/iptables/rules.v4
```

## ✅ Summary

You now have a fully functional LXC container running Llama with AMD GPU acceleration on Proxmox 9.1. Key achievements:

- ✓ BIOS configured for virtualization and IOMMU
- ✓ Proxmox host configured with AMD GPU drivers
- ✓ Privileged LXC container with GPU passthrough
- ✓ ROCm installed and configured in container
- ✓ llama.cpp built with ROCm/HIP support
- ✓ Models downloaded and running with GPU acceleration
- ✓ Optional Ollama installation
- ✓ Server mode configured for API access
- ✓ Monitoring and troubleshooting tools in place

## 📝 Performance Notes

With the AMD Strix Halo 395 integrated GPU and 128GB RAM:

- **Measured Performance** (Llama 3.2 3B Q5_K_M):
  - Prompt Processing: ~1000 tokens/second (with HSA_OVERRIDE_GFX_VERSION=11.5.1)
  - Text Generation: ~73 tokens/second
  - Full GPU offload with all 26 layers
  - **Note**: Using HSA_OVERRIDE_GFX_VERSION=11.5.1 provides 47% faster prompt processing vs 11.0.0
- **Expected Performance** for other models:
  - 3B models: 70-80 t/s generation
  - 7B models: 30-40 t/s generation (with Q4/Q5 quantization)
  - 13B models: 15-25 t/s generation (with Q4 quantization)
- **Recommended Models**: 
  - Llama 3.2 3B (runs entirely in GPU memory) ✓ Tested
  - Llama 3.1 8B (with Q4/Q5 quantization)
  - Mistral 7B variants
  - Qwen 2.5 7B
- **Memory Allocation**: 
  - 16GB+ for 3B models
  - 32GB+ for 7B models
  - 64GB+ for 13B models

## 🔗 Useful Resources

- [llama.cpp GitHub](https://github.com/ggerganov/llama.cpp)
- [Ollama Documentation](https://ollama.ai/docs)
- [ROCm Documentation](https://rocm.docs.amd.com/)
- [Proxmox LXC Documentation](https://pve.proxmox.com/wiki/Linux_Container)
- [Hugging Face Models](https://huggingface.co/models)

---

**Guide Version**: 1.0  
**Last Updated**: January 18, 2026  
**Tested On**: Proxmox VE 9.1, Ubuntu 24.04 LXC, ROCm 7.1.1, AMD Strix Halo 395
