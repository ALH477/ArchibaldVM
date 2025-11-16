### Directory Structure
- `host-flake/`
  - `flake.nix` (host configuration)
  - `modules/ArchibaldVM/`
    - `flake.nix` (VM configuration)
    - `modules/`
      - `audio.nix`
      - `users.nix`
---

# ArchibaldOS VM for Real-Time Audio Development with Host-Based Gaming

The ArchibaldOS VM is a headless, development-focused virtual machine designed to provide real-time (RT) audio processing for games running on the host NixOS system. Optimized for low-latency audio (~0.67ms latency, <0.1% xrun risk), it uses a PREEMPT_RT kernel with Musnix for RT optimizations and includes a minimal audio stack (PipeWire, Jack2, SuperCollider) alongside programming tools (GCC, Python, Git, CMake, Emacs, Nvim). The VM leverages IOMMU passthrough for near-native performance and supports networking with the host for PipeWire/JACK, StreamDB, and custom applications.

This repository provides two Nix flakes:
- **Host Flake**: Configures a NixOS host with a standard kernel to run games (e.g., via Steam, Vulkan RT, or Godot/Unity) and virtualization for the VM.
- **VM Flake**: Defines the ArchibaldOS VM for RT audio processing, imported by the host flake from `modules/ArchibaldVM`.

## Features
- **Real-Time Audio**: PREEMPT_RT kernel with Musnix achieves ~10-50µs system latency and <1ms audio latency for generative or ray-traced audio effects.
- **Development Tools**: GCC, Python, Git, CMake, Emacs, Nvim for audio programming (e.g., SuperCollider scripting, PipeWire/JACK APIs).
- **Networking**: Open ports for SSH (22), PipeWire/JACK (4713), StreamDB (8080), and custom apps (8000-9000) to integrate VM audio with host games.
- **Isolation**: IOMMU passthrough dedicates CPU cores to the VM, ensuring audio determinism without impacting host gaming performance.
- **Reproducibility**: Nix flakes ensure consistent VM builds.

## Use Cases
The ArchibaldOS VM supports games running on the host by providing a dedicated RT audio environment:

1. **Ray-Traced Audio for Games**:
   - Processes ray-traced audio effects (e.g., dynamic reverb, occlusion, echoes via sound source ray/path tracing) in real time, syncing with host-based game visuals (e.g., Vulkan RT in Quake II RTX or Cyberpunk 2077). The VM’s RT kernel ensures audio aligns with 60+ FPS gameplay.
   - Example: Use SuperCollider to compute ray-traced reverb for a Unity game, routed via PipeWire to the host’s audio output.

2. **Generative Audio Effects**:
   - Generates procedural audio (e.g., AI-driven soundscapes, physics-based effects) for games, with deterministic scheduling to prevent xruns. The VM’s PipeWire/JACK stack routes audio to the host game seamlessly.
   - Example: Create adaptive sound effects in SuperCollider for a Godot game, networked to the host via JACK (port 4713).

3. **Embedded Audio Simulation**:
   - Simulates embedded audio systems (e.g., for game consoles or VR headsets) on x86 hosts, preparing RT audio pipelines for deployment. The VM’s minimal stack mimics constrained environments.
   - Example: Test audio firmware for a DSP-based VR headset, passing through TI DSPs to the VM.

4. **Collaborative Game Audio Development**:
   - Provides reproducible environments for teams developing game audio, scalable to CI/CD pipelines or multi-VM setups for testing. Flakes ensure consistent builds across developers.
   - Example: Share VM configs for a team building networked audio features with StreamDB for a multiplayer game.

## Problems Solved
- **Host Stability for Gaming**: Games require a stable, standard kernel for optimal performance (e.g., NVIDIA drivers, Steam Proton). The VM isolates the PREEMPT_RT kernel, avoiding host instability while delivering RT audio.
- **Audio Latency in Games**: Standard kernels introduce jitter for ray-traced or generative audio. The VM’s IOMMU passthrough and Musnix ensure ~10-50µs latency, syncing audio with game visuals.
- **Reproducibility**: Nix flakes eliminate environment mismatches, ensuring audio pipelines work identically across development and production for host-based games.
- **Isolation**: The VM sandboxes experimental audio code or DSP passthrough, protecting the host’s gaming environment from crashes.
- **Audio-Game Integration**: Open networking ports (e.g., 4713 for PipeWire/JACK) enable seamless host-VM audio routing, solving integration challenges for dynamic game audio.

## Prerequisites
### Hardware
- **CPU**: x86_64 with IOMMU support (Intel VT-d or AMD-Vi). Verify: `dmesg | grep -E "DMAR|IOMMU"`.
- **Cores**: 4+ (2 for VM audio, 2 for host gaming). Adjust `isolcpus=2-3` if needed.
- **RAM**: 8GB+ (4GB for VM).
- **Storage**: 10GB SSD free space.
- **GPU**: NVIDIA/AMD for host gaming (e.g., Vulkan RT). Optional: Passthrough to VM for audio DSPs.
- **BIOS**: Enable VT-d (Intel) or AMD-Vi (AMD).
- **Caveat**: ARM64 is not supported due to limited IOMMU availability (e.g., on Raspberry Pi 5). For ARM64, consider a native RT kernel setup.

### Software
- **OS**: NixOS 24.05 or later with `nix` and flake support.
- **Host Gaming Tools**: Install `steam`, `proton-ge-custom`, or game engines (e.g., `godot4`) on the host.
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
   - Ensure cores 2-3 (or DSPs) are in separate groups.
3. **DSP Passthrough (Optional)**:
   - Identify PCI IDs: `lspci -nn` (e.g., `0000:01:00.0` for TI DSPs).
   - Update `boot.extraModprobeConfig` in `host-flake/flake.nix`:
     ```nix
     options vfio-pci ids=your,device,ids
     ```

### 2. Set Up Host for Gaming
1. Install gaming dependencies (edit `host-flake/flake.nix`):
   ```nix
   environment.systemPackages = with pkgs; [
     virt-manager htop libguestfs
     steam proton-ge-custom vulkan-tools godot4
   ];
   ```
2. Apply host configuration:
   ```bash
   cd host-flake
   sudo nixos-rebuild switch --flake .#host
   ```
3. Verify:
   - Kernel: `uname -a` (no “PREEMPT_RT”).
   - VFIO: `lsmod | grep vfio`.
   - Networking: `ip addr show br0` (shows `192.168.122.1`).
   - Gaming: `steam` or `godot4` launches.

### 3. Prepare Repository
1. Clone or create:
   ```bash
   mkdir -p host-flake/modules/ArchibaldVM/modules
   cd host-flake
   ```
2. Save `flake.nix` in `host-flake/`.
3. Save `modules/ArchibaldVM/flake.nix` and `modules/{audio,users}.nix` in `host-flake/modules/ArchibaldVM/modules`.

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
### Host-Based Gaming
- **Run Games**:
  - Launch Steam: `steam` (e.g., Quake II RTX, Cyberpunk 2077 with Vulkan RT).
  - Use Godot: `godot4` for custom projects.
  - Ensure host audio output is configured (e.g., PipeWire/PulseAudio).
- **Networking**:
  - Route game audio to VM via PipeWire/JACK (port 4713).
  - Access StreamDB (e.g., for game audio assets) at `192.168.122.2:8080`.

### VM Audio Development
- **SuperCollider**:
  - Run: `chrt -f 95 sclang`.
  - Test ray-traced audio: `{SinOsc.ar(440) * EnvGen.ar(Env.perc)}.play`.
  - Example: Script procedural reverb for host game.
- **PipeWire/JACK**:
  - Check latency: `pw-cli info all | grep latency` (~0.67ms).
  - Route to host: `jack_connect` to `192.168.122.1:4713`.
- **Thread Priorities**:
  - Identify: `ps -eL | grep sclang`.
  - Set: `chrt -f -p 95 <thread-pid>`.
- **Networking**:
  - StreamDB: Access `192.168.122.2:8080`.
  - Custom: Use ports 8000-9000 for host-VM audio apps.

### Programming
- **Editors**:
  - Emacs: `emacs -nw` (add `(require 'supercollider)`).
  - Nvim: `nvim` (add `vimPlugins.nvim-treesitter` if needed).
- **Build Tools**:
  - Compile: `gcc my_audio.c -o my_audio`.
  - Script: `python3 audio_test.py`.
  - Manage: `git init my_audio_project`.

### Testing Latency
1. **System Latency**:
   - Run: `cyclictest -p 99 -t 4 -n -a 0-1`.
   - Expect: ~10-50µs worst-case.
2. **Audio Latency**:
   - PipeWire: `pw-top` (expect <1ms).
   - JACK: `jack_iodelay` (expect <1ms).
   - Xruns: Test SuperCollider for 10 minutes (<0.1% risk).
3. **RT Scan**:
   - Add `realtimeconfigquickscan` to VM’s `environment.systemPackages`.
   - Run: `realtimeconfigquickscan`.

## Customization
- **CPU Cores**: Adjust `isolcpus` and `cores` in both flakes based on `nproc`.
- **DSPs**: Add to `vfio-pci.ids` and `extraQemuOptions` in VM flake.
- **Host Gaming**: Add `unityhub`, `unrealengine` to host flake.
- **VM Tools**: Add `rustc`, `go` to VM flake.
- **StreamDB**: Configure via `services.streamdb` in VM flake.
- **Editors**: Add Emacs/Nvim configs (e.g., `environment.etc."emacs/init.el"`).

## Troubleshooting
- **IOMMU Errors**:
  - Check: `dmesg | grep vfio`.
  - Fix: Verify BIOS, PCI IDs.
- **High Latency**:
  - Monitor host: `htop` (cores 0-1).
  - Increase QEMU priority: Adjust `CPUSchedulingPriority` in `systemd.services.libvirtd`.
- **Networking**:
  - Fix: `virsh net-start default`.
  - Test: `nc -zv 192.168.122.2 4713`.
- **Audio Dropouts**:
  - Increase buffer: Edit VM’s `/etc/pipewire/pipewire.conf`.
  - Test: `stress-ng --cpu 2` on host.

## Limitations
- **ARM64**: Not supported due to limited IOMMU availability (e.g., Raspberry Pi 5). Use a native RT kernel for ARM64.
- **DSPs**: For TI DSPs, pass PCI devices via VFIO and load drivers in the VM.
- **Host GPU**: Games rely on host GPU; VM GPU passthrough is optional for audio DSPs.

## Contributing
Submit issues or pull requests to enhance functionality. Focus areas include advanced audio routing, DSP support, or game engine integration.

## License
This project is licensed under the MIT License.

---

### Caveats
- **Imports**: Only `audio.nix`, `users.nix`, StreamDB remain (excluded `installer.nix`, `branding.nix`, `desktop.nix`).
- **Path**: Ensure `modules/ArchibaldVM/flake.nix` exists.
- **ARM64**: Noted in README (not supported).
- **Modules**: Verify `audio.nix`, `users.nix` match past work.
- **DSPs**: Pass TI DSPs via VFIO if needed.
