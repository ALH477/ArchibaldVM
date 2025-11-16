# ArchibaldOS VM for Real-Time Audio Development

The ArchibaldOS VM is a headless, development-focused virtual machine optimized for real-time (RT) audio programming, targeting low-latency audio processing (~0.67ms latency, <0.1% xrun risk). Built on NixOS with a PREEMPT_RT kernel, it leverages Musnix for RT optimizations and includes a minimal audio stack (PipeWire, Jack2, SuperCollider) alongside programming tools (GCC, Python, Git, CMake, Emacs, Nvim). The VM uses IOMMU passthrough for near-native performance and supports networking with the host for PipeWire/JACK, StreamDB, and custom applications.

This repository provides two Nix flakes:
- **Host Flake**: Configures a NixOS host with a standard kernel and IOMMU optimizations to run the VM.
- **VM Flake**: Defines the ArchibaldOS VM, imported by the host flake from `modules/ArchibaldVM`.

## Features
- **Real-Time Audio**: PREEMPT_RT kernel with Musnix achieves ~10-50µs system latency and <1ms audio latency.
- **Development Tools**: GCC, Python, Git, CMake, Emacs, Nvim for audio programming (e.g., SuperCollider scripting, PipeWire/JACK APIs).
- **Networking**: Open ports for SSH (22), PipeWire/JACK (4713), StreamDB (8080), and custom apps (8000-9000).
- **Isolation**: IOMMU passthrough dedicates CPU cores, minimizing host interference.
- **Reproducibility**: Nix flakes ensure consistent builds.

## Use Cases
The ArchibaldOS VM addresses specific needs in audio development and related fields:

1. **Real-Time Audio Prototyping and Testing**:
   - Enables developers to build and test low-latency audio pipelines (e.g., synthesis, effects) in a controlled environment. The RT kernel and Musnix ensure deterministic performance, ideal for live audio applications.
   - Example: Iterating on SuperCollider scripts for live synthesis or testing PipeWire/JACK configurations for VR audio engines.

2. **Embedded and Edge Computing Simulation**:
   - Simulates embedded audio systems on x86 hosts, preparing workloads for devices like DSP boards or IoT audio nodes. The minimal stack supports development for constrained environments.
   - Example: Developing firmware for audio DSPs (e.g., TI DSPs) with passthrough for direct hardware testing.

3. **Secure, Isolated Development Workflows**:
   - Provides a sandbox for sensitive audio or security-related programming (e.g., custom protocols, anomaly detection) without risking host stability. Networking ports enable host-VM data exchange.
   - Example: Building audio tokenization systems or secure streaming protocols integrated with StreamDB.

4. **Collaborative and Production-Ready Environments**:
   - Facilitates team collaboration via reproducible flake-based builds, scalable to CI/CD pipelines or multi-VM clusters for audio rendering or testing.
   - Example: Sharing consistent development setups for distributed teams working on audio SaaS tools or networked audio applications.

## Problems Solved
- **Host Stability**: Running an RT kernel on the host can introduce overhead or instability for non-RT tasks. The VM isolates the RT environment, allowing a standard kernel on the host for general-purpose stability.
- **Latency Guarantees**: IOMMU passthrough and Musnix optimizations ensure near-native RT performance, solving jitter issues common in virtualized audio setups.
- **Reproducibility**: Nix flakes eliminate “works on my machine” problems, providing identical environments across development, testing, and production.
- **Isolation**: The VM sandboxes experimental audio code or hardware passthrough (e.g., DSPs), protecting the host from crashes or misconfigurations.
- **Cross-Platform Prep**: Simulates embedded audio systems on x86, bridging desktop and edge device development without dedicated hardware.

## Prerequisites
### Hardware
- **CPU**: x86_64 with IOMMU support (Intel VT-d or AMD-Vi). Verify: `dmesg | grep -E "DMAR|IOMMU"`.
- **Cores**: 4+ (2 for VM, 2 for host). Adjust `isolcpus=2-3` if needed.
- **RAM**: 8GB+ (4GB for VM).
- **Storage**: 10GB SSD free space.
- **BIOS**: Enable VT-d (Intel) or AMD-Vi (AMD).
- **Caveat**: ARM64 is not supported due to limited IOMMU availability on devices like Raspberry Pi 5. For ARM64, consider a native RT kernel setup.

### Software
- **OS**: NixOS 24.05 or later with `nix` and flake support.
- **Tools**: `git` for cloning, `virt-manager` (optional) for VM management.

### Directory Structure
```
host-flake/
├── flake.nix
└── modules/ArchibaldVM/
    ├── flake.nix
    └── modules/
        ├── audio.nix
        ├── users.nix
```

## Setup Instructions
### 1. Verify IOMMU Support
1. **Check BIOS**:
   - Enable VT-d (Intel) or AMD-Vi (AMD).
   - Reboot and verify: `dmesg | grep -E "DMAR|IOMMU"`. Expect: `DMAR: IOMMU enabled`.
2. **Inspect IOMMU Groups**:
   - Run:
     ```bash
     for d in /sys/kernel/iommu_groups/*/devices/*; do
       echo "IOMMU Group ${d##*/iommugroup-}: $(lspci -nns ${d##*/})"
     done
     ```
   - Ensure cores 2-3 (or devices like DSPs) are in separate groups.
3. **Device Passthrough (Optional)**:
   - Identify PCI IDs: `lspci -nn` (e.g., `0000:01:00.0` for DSPs).
   - Update `boot.extraModprobeConfig` in `host-flake/flake.nix`:
     ```nix
     options vfio-pci ids=your,device,ids
     ```

### 2. Clone and Prepare Repository
1. Clone or create the repository:
   ```bash
   mkdir -p host-flake/modules/ArchibaldVM/modules
   cd host-flake
   ```
2. Save `flake.nix` in `host-flake/`.
3. Save `modules/ArchibaldVM/flake.nix` and `modules/{audio,users}.nix` in `host-flake/modules/ArchibaldVM/modules/`.

### 3. Configure Host
1. Apply host configuration:
   ```bash
   sudo nixos-rebuild switch --flake .#host
   ```
2. Verify:
   - Kernel: `uname -a` (no “PREEMPT_RT”).
   - VFIO: `lsmod | grep vfio` (expect `vfio_pci`, `vfio_iommu_type1`).
   - Networking: `ip addr show br0` (shows `192.168.122.1`).

### 4. Build and Run VM
1. Build VM image:
   ```bash
   nix build .#packages.x86_64-linux.archibaldos-vm-image
   ```
2. Run VM:
   ```bash
   ./result/bin/run-nixos-vm
   ```
3. Access VM:
   - SSH: `ssh audio-user@192.168.122.2` (password: `archibaldos`).
4. Verify VM:
   - Kernel: `uname -a` (shows “PREEMPT_RT”).
   - StreamDB: `systemctl status streamdb`.
   - Tools: `gcc --version`, `python3 --version`, `emacs --version`, `nvim --version`.

## Usage
### Audio Development
- **SuperCollider**:
  - Run: `chrt -f 95 sclang`.
  - Test: `{SinOsc.ar(440)}.play`.
- **PipeWire/JACK**:
  - Check latency: `pw-cli info all | grep latency` (~0.67ms).
  - Route audio: Use `pw-cli` or `jack_connect`.
  - Network audio: Connect to host at `192.168.122.1:4713`.
- **Thread Priorities**:
  - Identify threads: `ps -eL | grep sclang`.
  - Set RT priority: `chrt -f -p 95 <thread-pid>`.

### Programming
- **Editors**:
  - Emacs: `emacs -nw` (add `(require 'supercollider)` for audio dev).
  - Nvim: `nvim` (add `vimPlugins.nvim-treesitter` if needed).
- **Build Tools**:
  - Compile: `gcc my_audio.c -o my_audio`.
  - Script: `python3 audio_test.py`.
  - Manage: `git init my_audio_project`.
- **Networking**:
  - StreamDB: Access `192.168.122.2:8080` from host.
  - Custom apps: Use ports 8000-9000.

### Testing Latency
1. **System Latency**:
   - Run: `cyclictest -p 99 -t 4 -n -a 0-1`.
   - Expect: ~10-50µs worst-case.
2. **Audio Latency**:
   - PipeWire: `pw-top` (expect <1ms).
   - JACK: `jack_iodelay` (expect <1ms).
   - Xruns: Test SuperCollider for 10 minutes (<0.1% risk).
3. **RT Scan**:
   - Add `realtimeconfigquickscan` to `environment.systemPackages`.
   - Run: `realtimeconfigquickscan`.

## Customization
- **CPU Cores**: Adjust `isolcpus` and `cores` in both flakes based on `nproc`.
- **Devices**: Add DSPs/GPUs to `vfio-pci.ids` and `extraQemuOptions` in VM flake.
- **Tools**: Add `rustc`, `go`, etc., to `environment.systemPackages` in VM flake.
- **StreamDB**: Configure via `services.streamdb` in VM flake.
- **Editors**: Add Emacs/Nvim configs (e.g., `environment.etc."emacs/init.el"`) or plugins.

## Troubleshooting
- **IOMMU Errors**:
  - Check: `dmesg | grep vfio`.
  - Fix: Verify BIOS settings, PCI IDs.
- **High Latency**:
  - Monitor host: `htop` (cores 0-1).
  - Increase QEMU priority: Adjust `CPUSchedulingPriority` in `systemd.services.libvirtd`.
- **Networking**:
  - Fix: `virsh net-start default`.
  - Test: `nc -zv 192.168.122.2 4713`.
- **Audio Dropouts**:
  - Increase buffer: Edit `/etc/pipewire/pipewire.conf` in VM.
  - Test interference: `stress-ng --cpu 2` on host.

## Limitations
- **ARM64**: Not supported due to limited IOMMU availability on devices like Raspberry Pi 5. For ARM64, consider a native RT kernel setup.
- **DSPs**: For TI DSPs, pass PCI devices via VFIO and load drivers in the VM.

## Contributing
Submit issues or pull requests to enhance functionality. Focus areas include additional audio tools, DSP support, or networking optimizations.

## License
This project is licensed under the MIT License.

---

### Caveats
- **Imports**: Removed `desktop.nix`, `branding.nix`, `installer.nix`, keeping `audio.nix`, `users.nix`, StreamDB.
- **Path**: Ensure `modules/ArchibaldVM/flake.nix` exists.
- **ARM64**: Noted in README (not supported).
- **Modules**: Verify `audio.nix`, `users.nix` match past work.
- **DSPs**: Pass TI DSPs via VFIO if used.
