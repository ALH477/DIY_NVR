# DIY NVR Setup Guides

This repository provides detailed setup guides for building cost-effective DIY Network Video Recorders (NVRs) using open-source software like Frigate. We cover x86-64 and ARM64 architectures with tiered options (budget, mid-range, high-end). These guides assume basic familiarity with Linux and Docker.

All configurations are designed for handling multiple camera streams with AI object detection. The guides are open source under the MIT License.

## x86-64 NVR Setup Guides
x86-64 setups leverage mature software support and hardware acceleration. These assume Frigate as the NVR software. Use Ubuntu Server 24.04 LTS for the OS.

### Budget Tier Setup (Beelink MINI S12)
1. **Hardware Assembly**: Unbox MINI S12. Connect power, monitor, keyboard. Add WD Purple 1TB HDD via USB (or internal if slot available). Plug in Coral TPU to USB port. Connect BV-Tech PoE switch to Ethernet; wire cameras to switch.
2. **OS Installation**: Boot from Ubuntu USB. Install to internal SSD. Update via `sudo apt update && sudo apt upgrade`.
3. **Software Setup**: Install Docker: `sudo apt install docker.io docker-compose`. Pull Frigate: Create `docker-compose.yml` with Frigate config (enable Intel Quick Sync via `devices: - /dev/dri`). Add Coral: Map `/dev/apex_0:/dev/apex_0` in compose file. Start: `docker-compose up -d`.
4. **Configuration**: Edit Frigate config.yml: Add cameras (RTSP URLs), enable object detection with Coral. Set storage paths to HDD. Access web UI at IP:5000. Test streams.
5. **Optimization**: Enable hardware acceleration: Add `ffmpeg.hwaccel_args: preset-vaapi` in config. Monitor via Home Assistant integration if needed.

### Mid-Range Tier Setup (Beelink EQ13)
1. **Hardware Assembly**: Unbox EQ13. Use dual Ethernet: One for WAN, one for PoE switch (TP-Link). Add 1TB SSD internally if upgradeable; connect 2TB HDD externally. Insert Coral TPU.
2. **OS Installation**: Same as budget: Ubuntu Server on SSD.
3. **Software Setup**: Install Docker as above. Use compose file with Quick Sync and Coral mappings. For dual NIC, configure networking in OS.
4. **Configuration**: In Frigate config, add more cameras (up to 12). Use SSD for OS/cache, HDD for footage. Enable 4K decoding if cameras support.
5. **Optimization**: Tune CPU governor for efficiency: `cpupower frequency-set -g powersave`. Add MQTT for smart home integration.

### High-End Tier Setup (Geekom A6 Mini PC)
1. **Hardware Assembly**: Unbox A6. Add 2TB SSD + 4TB HDD internally (multiple slots). Connect Coral TPU. Use Ubiquiti 8-port PoE for cameras.
2. **OS Installation**: Ubuntu Server; partition SSD for OS, HDD for storage.
3. **Software Setup**: Docker install. Compose file: Enable AMD GPU accel if available, plus Coral.
4. **Configuration**: Advanced Frigate setup: Multi-camera AI, zones, masks. Use managed switch for VLANs.
5. **Optimization**: Overclock if needed; monitor temps. Integrate with NAS for backups.

## ARM64 NVR Setup Guides
ARM64 focuses on low power but may need tweaks for software. Use Raspberry Pi OS or Armbian for compatibility.

### Budget Tier Setup (Raspberry Pi 5)
1. **Hardware Assembly**: Assemble Pi 5 in case with PSU/heatsink. Add microSD with SSD adapter for 500GB SSD. Connect 1TB HDD via USB. Plug Coral TPU. Link BV-Tech PoE to Ethernet.
2. **OS Installation**: Flash Raspberry Pi OS to microSD. Boot, update: `sudo apt update && sudo apt upgrade`.
3. **Software Setup**: Install Docker: Follow Pi docs for arm64. Create Frigate compose file (enable Coral via USB mapping).
4. **Configuration**: Config.yml: Add 2-4 cameras (lower res). Set storage to HDD. Access UI at IP:5000.
5. **Optimization**: Overclock Pi if needed; use Coral for all detection to offload CPU.

### Mid-Range Tier Setup (Orange Pi 5 Plus)
1. **Hardware Assembly**: Mount board; add enclosure if separate. Install 1TB SSD in M.2 slot, 2TB HDD via USB. Insert Coral. Connect TP-Link PoE.
2. **OS Installation**: Flash Armbian to eMMC/SD. Update system.
3. **Software Setup**: Docker install for arm64. Compose: Use RK3588 NPU for accel (`devices: - /dev/dri`).
4. **Configuration**: Frigate config for 4-8 cameras; leverage built-in NPU for AI alongside Coral.
5. **Optimization**: Enable hardware decoding in FFMPEG args.

### High-End Tier Setup (Firefly RK3588S)
1. **Hardware Assembly**: Enclose board; add cooling. Fit 2TB SSD + 4TB HDD. Coral via USB. Use Ubiquiti PoE.
2. **OS Installation**: Armbian or Ubuntu arm64; update.
3. **Software Setup**: Docker; compose with RK3588 accel and Coral.
4. **Configuration**: Handle 8+ cameras; advanced AI configs.
5. **Optimization**: Custom kernel for better perf; monitor power draw.

## License
MIT License

Brought to you by DeMoD LLC - [DeMoD.ltd](https://demod.ltd)
