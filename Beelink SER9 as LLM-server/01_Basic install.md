Basic install
This guide starts from a fresh Ubuntu Server 26.04 installation and stops after llama.cpp has been built under /opt and verified from the dedicated llama service account.

## 1. Update the system

```
sudo apt update
sudo apt upgrade
```

## 2. Install useful base packages

Install the editor, monitoring tools, general utilities, hardware tools, AMD GPU monitor, and the zram generator in one step.

```
sudo apt install nano htop btop tmux ncdu curl wget git jq unzip lm-sensors iotop-c nvme-cli smartmontools radeontop systemd-zram-generator
```

## 3. Configure 1 GiB zram

```
sudo nano /etc/systemd/zram-generator.conf
```

Enter the following configuration:

```
[zram0]
zram-size = 1024
compression-algorithm = zstd
swap-priority = 100
```

Important: for systemd-zram-generator, use 1024 here for 1 GiB. Do not use 1G.

REBOOT

## 4. Activate and verify zram

```
sudo systemctl daemon-reload
sudo systemctl start dev-zram0.swap
```

Verify the device and swap priority:

```
zramctl
swapon --show
```

The expected result is a 1 GiB /dev/zram0 device using zstd, with swap priority 100. The existing /swap.img may remain enabled at its lower priority.

## 5. Install the llama.cpp build and Vulkan packages

```
sudo apt install build-essential cmake ninja-build libvulkan-dev glslc spirv-headers vulkan-tools libssl-dev mesa-vulkan-drivers
```

## 6. Create the dedicated llama service user

```
sudo useradd --system --no-create-home --shell /usr/sbin/nologin llama
sudo usermod -aG render,video llama
```

Verify the account and its supplementary groups:

```
id llama
```

## 7. Create the application and model directories

```
sudo mkdir -p /opt/llama.cpp /opt/llama-models
sudo chown $USER:$USER /opt/llama.cpp
sudo chown llama:llama /opt/llama-models
```

The source, build, and executable files stay together under /opt/llama.cpp. GGUF model files will be kept under /opt/llama-models.

## 8. Verify Vulkan access as the service user

```
sudo -u llama vulkaninfo --summary
```

Confirm that the AMD GPU appears as a RADV Vulkan device. Seeing llvmpipe in addition to the real GPU is normal; the important part is that the AMD GPU is listed.

## 9. Download llama.cpp

```
git clone https://github.com/ggml-org/llama.cpp /opt/llama.cpp
cd /opt/llama.cpp
```

## 10. Configure the Vulkan build

```
cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Release -DGGML_VULKAN=ON
```

## 11. Compile llama.cpp

```
cmake --build build -j $(nproc)
```

## 12. Verify the build

```
./build/bin/llama-cli --version
./build/bin/llama-server --version
```

Then verify GPU detection using the same account that will eventually run the boot service:

```
sudo -u llama /opt/llama.cpp/build/bin/llama-cli --list-devices
```

The AMD Radeon GPU should be listed as an available Vulkan device. At this point the base OS, zram, Vulkan access, directory layout, service account, and llama.cpp build are ready. Model download and the systemd llama-server service belong in the next section.
