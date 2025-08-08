Fully Fleshed Out NixOS Configuration for DIY NVR
This document provides a complete, standalone NixOS configuration.nix template for setting up Frigate as an NVR. It includes basic system setup, hardware configurations with placeholders for x86-64 or ARM64, and support for accelerators like Coral TPU, Intel Quick Sync, or Rockchip NPU. The config assumes a server-like setup (no desktop environment) and uses the built-in services.frigate module available in NixOS 23.11+.
Apply this to /etc/nixos/configuration.nix and run nixos-rebuild switch. For ARM64 (e.g., Raspberry Pi), ensure the system is built for aarch64-linux.
configuration.nix
# /etc/nixos/configuration.nix
# Edit this configuration file to define what should be installed on your system.

{ config, pkgs, lib, ... }:

{
  imports = [
    # Include hardware configuration (generate with nixos-generate-config)
    ./hardware-configuration.nix
  ];

  # Boot loader configuration
  boot.loader = {
    # For x86-64 systems
    # systemd-boot.enable = true;
    # efi.canTouchEfiVariables = true;

    # For ARM64 (e.g., Raspberry Pi)
    # generic-extlinux-compatible.enable = true;
    # raspberryPi.enable = true;  # If using official Raspberry Pi support
  };

  # Kernel and modules (adjust for ARM)
  boot.kernelPackages = pkgs.linuxPackages_latest;  # Or linuxPackages_rpi4 for Pi
  boot.initrd.availableKernelModules = [ "usbhid" "usb_storage" ];  # Add for USB devices

  # Networking
  networking = {
    hostName = "nvr-server";  # Set your hostname
    useDHCP = lib.mkDefault true;  # Or configure static IP
    firewall = {
      enable = true;
      allowedTCPPorts = [ 5000 8554 ];  # Frigate web UI and RTSP
    };
  };

  # Time zone and locale
  time.timeZone = "UTC";
  i18n.defaultLocale = "en_US.UTF-8";

  # Users
  users.users.root = {
    initialPassword = "change-me";  # Change immediately
  };
  users.users.nvruser = {
    isNormalUser = true;
    extraGroups = [ "wheel" "video" "docker" ];  # For hardware access
    initialPassword = "password";  # Change this
  };

  # System packages
  environment.systemPackages = with pkgs; [
    vim  # Or your editor
    git
    htop
    ffmpeg-full
    libedgetpu  # For Coral
  ];

  # Enable Docker if not using native Frigate service
  # virtualisation.docker.enable = true;

  # Hardware-specific configurations
  # INSERT_HARDWARE_HERE
  hardware.opengl.enable = true;
  hardware.opengl.extraPackages = with pkgs; [
    # For Intel: intel-media-driver vaapiIntel
    # For ARM/Rockchip: mesa
  ];

  # Services
  services.openssh.enable = true;  # For remote access

  # Frigate NVR service
  services.frigate = {
    enable = true;
    hostname = "frigate";
    settings = {
      mqtt = {
        enabled = false;  # Enable if using Home Assistant
      };
      cameras = {
        # Example camera; add your own
        front_door = {
          ffmpeg = {
            inputs = [
              { path = "rtsp://user:pass@camera-ip:554/stream"; }
            ];
          };
          detect = {
            enabled = true;
          };
        };
      };
      detectors = {
        # INSERT_DETECTOR_HERE
      };
      record = {
        enabled = true;
        retain = {
          days = 7;
        };
      };
      # Add storage paths, etc.
    };
  };

  # System version (do not edit)
  system.stateVersion = "24.05";  # Adjust to your NixOS version
}

Bash Script to Adjust Config (adjust_nvr_config.sh)
This script modifies the configuration.nix based on architecture (x86 or arm) and accelerator (coral, intel, rockchip, or none). It also handles basic boot loader adjustments for ARM vs x86. Run as ./adjust_nvr_config.sh --config /etc/nixos/configuration.nix --arch x86 --accelerator coral --board rpi4 (board optional for ARM specifics).
#!/bin/bash

# Usage: ./adjust_nvr_config.sh --config <path> --arch <x86|arm> --accelerator <coral|intel|rockchip|none> [--board <rpi4|generic>]

CONFIG_FILE=""
ARCH=""
ACCELERATOR="none"
BOARD="generic"

while [[ $# -gt 0 ]]; do
  case $1 in
    --config) CONFIG_FILE="$2"; shift 2 ;;
    --arch) ARCH="$2"; shift 2 ;;
    --accelerator) ACCELERATOR="$2"; shift 2 ;;
    --board) BOARD="$2"; shift 2 ;;
    *) echo "Unknown option $1"; exit 1 ;;
  esac
done

if [ -z "$CONFIG_FILE" ] || [ -z "$ARCH" ]; then
  echo "Usage: $0 --config <path> --arch <x86|arm> --accelerator <coral|intel|rockchip|none> [--board <rpi4|generic>]"
  exit 1
fi

if [ ! -f "$CONFIG_FILE" ]; then
  echo "Config file not found: $CONFIG_FILE"
  exit 1
fi

# Backup original
cp "$CONFIG_FILE" "$CONFIG_FILE.bak"

# Modify boot loader based on arch and board
if [ "$ARCH" = "x86" ]; then
  sed -i 's/# For x86-64 systems/systemd-boot.enable = true;\n    efi.canTouchEfiVariables = true;/g' "$CONFIG_FILE"
  sed -i 's/# For ARM64 (e.g., Raspberry Pi)//g' "$CONFIG_FILE"
else
  sed -i 's/# For x86-64 systems//g' "$CONFIG_FILE"
  if [ "$BOARD" = "rpi4" ]; then
    sed -i 's/# For ARM64 (e.g., Raspberry Pi)/generic-extlinux-compatible.enable = true;\n    raspberryPi.enable = true;\n    raspberryPi.version = 4;/g' "$CONFIG_FILE"
  else
    sed -i 's/# For ARM64 (e.g., Raspberry Pi)/generic-extlinux-compatible.enable = true;/g' "$CONFIG_FILE"
  fi
  # Adjust kernel for ARM
  sed -i 's/boot.kernelPackages = pkgs.linuxPackages_latest;/boot.kernelPackages = pkgs.linuxPackages_rpi4;  # Or appropriate for your board/g' "$CONFIG_FILE"
fi

# Modify detectors
DETECTOR=""
case $ACCELERATOR in
  coral)
    DETECTOR="coral = { type = \"edgetpu\"; device = \"usb\"; };"
    ;;
  intel)
    DETECTOR="cpu = { type = \"cpu\"; };  # Use VAAPI in Frigate ffmpeg args"
    ;;
  rockchip)
    DETECTOR="rknpu = { type = \"rknpu\"; };  # Assuming custom support"
    ;;
  none)
    DETECTOR="cpu = { type = \"cpu\"; };"
    ;;
esac
sed -i "s/# INSERT_DETECTOR_HERE/$DETECTOR/g" "$CONFIG_FILE"

# Modify hardware
HARDWARE=""
if [ "$ARCH" = "x86" ]; then
  HARDWARE="hardware.opengl.extraPackages = with pkgs; [ intel-media-driver vaapiIntel libvdpau-va-gl ];"
  if [ "$ACCELERATOR" = "coral" ]; then
    HARDWARE="$HARDWARE\n  services.udev.extraRules = \"SUBSYSTEM==\\\"apex\\\", MODE=\\\"0660\\\", GROUP=\\\"users\\\"\";"
  fi
elif [ "$ARCH" = "arm" ]; then
  HARDWARE="hardware.opengl.extraPackages = with pkgs; [ mesa ];"
  HARDWARE="$HARDWARE\n  boot.kernelModules = [ \"videodev\" \"v4l2loopback\" ];  # For video"
  if [ "$ACCELERATOR" = "rockchip" ]; then
    HARDWARE="$HARDWARE\n  boot.kernelModules = boot.kernelModules ++ [ \"rknpu\" \"mali\" ];"
  fi
  if [ "$ACCELERATOR" = "coral" ]; then
    HARDWARE="$HARDWARE\n  environment.systemPackages = config.environment.systemPackages ++ [ pkgs.libedgetpu ];"
  fi
fi
sed -i "s/# INSERT_HARDWARE_HERE/$HARDWARE/g" "$CONFIG_FILE"

echo "Configuration adjusted for $ARCH ($BOARD) with $ACCELERATOR. Backup: $CONFIG_FILE.bak"
echo "Run 'nixos-rebuild switch' to apply. For ARM, ensure aarch64 build."

Notes

Hardware Configuration: Generate ./hardware-configuration.nix with nixos-generate-config after initial install.
Frigate Customization: Edit the services.frigate.settings section for your cameras, storage, etc. Refer to Frigate docs.
ARM64 Specifics: For Raspberry Pi 4, use --board rpi4. For other ARM boards (e.g., Rockchip), adjust kernel/modules manually.
Python Alternative: If complex logic is needed (e.g., interactive questions), a Python script could replace bash, but bash suffices here.
Testing: Verify on a VM first. For Coral, ensure USB passthrough.
