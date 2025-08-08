# Debian and Arch Linux Configurations for DIY NVR

These provide templates and Bash CLI scripts for setting up Frigate as an NVR on Debian-based (e.g., Ubuntu 24.04) and Arch Linux systems. The scripts take command-line arguments for architecture (x86 or arm) and accelerator (coral, intel, rockchip, or none), then install necessary packages and set up a basic Docker Compose file for Frigate. Assume a fresh install; run scripts as root or with sudo.

## Debian (Ubuntu) Setup

### Template Docker Compose File (docker-compose.yml)
Save this in `/opt/frigate/` and adjust manually for cameras.

```yaml
version: "3.9"
services:
  frigate:
    container_name: frigate
    image: ghcr.io/blakeblackshear/frigate:stable
    restart: unless-stopped
    privileged: true  # For hardware access
    devices:
      # INSERT_DEVICES_HERE
    volumes:
      - /opt/frigate/config.yml:/config/config.yml:ro
      - /opt/frigate/media:/media/frigate
      - type: tmpfs
        target: /tmp/cache
        tmpfs:
          size: 1000000000
    ports:
      - "5000:5000"
      - "8554:8554"  # RTSP
    environment:
      FRIGATE_RTSP_PASSWORD: "password"
```

### Bash CLI Script for Debian (setup_nvr_debian.sh)
Run with `./setup_nvr_debian.sh --arch x86 --accelerator coral`. It installs packages, sets up Docker, and modifies the compose file.

```bash
#!/bin/bash

# Usage: ./setup_nvr_debian.sh --arch <x86|arm> --accelerator <coral|intel|rockchip|none>

ARCH=""
ACCELERATOR="none"

while [[ $# -gt 0 ]]; do
  case $1 in
    --arch) ARCH="$2"; shift 2 ;;
    --accelerator) ACCELERATOR="$2"; shift 2 ;;
    *) echo "Unknown option $1"; exit 1 ;;
  esac
done

if [ -z "$ARCH" ]; then
  echo "Usage: $0 --arch <x86|arm> --accelerator <coral|intel|rockchip|none>"
  exit 1
fi

# Update and install base packages
apt update && apt upgrade -y
apt install -y docker.io docker-compose ffmpeg

# Install hardware-specific packages
case $ACCELERATOR in
  coral)
    # Add Coral repo and install
    echo "deb https://packages.cloud.google.com/apt coral-edgetpu-stable main" | tee /etc/apt/sources.list.d/coral-edgetpu.list
    curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
    apt update
    apt install -y libedgetpu1-std
    ;;
  intel)
    apt install -y intel-media-va-driver vainfo
    ;;
  rockchip)
    # For ARM Rockchip, assume custom; install mpp if available
    apt install -y rockchip-multimedia  # May need PPA or custom repo
    ;;
  none)
    echo "No accelerator selected; using CPU."
    ;;
esac

# Create directories
mkdir -p /opt/frigate

# Download template compose if not exists
if [ ! -f "/opt/frigate/docker-compose.yml" ]; then
  curl -o /opt/frigate/docker-compose.yml https://raw.githubusercontent.com/blakeblackshear/frigate/master/docker/docker-compose.yml  # Or use your template
fi

# Modify devices in compose
DEVICES=""
case $ACCELERATOR in
  coral) DEVICES="- /dev/apex_0:/dev/apex_0" ;;
  intel) DEVICES="- /dev/dri:/dev/dri" ;;
  rockchip) DEVICES="- /dev/dri:/dev/dri - /dev/mali0:/dev/mali0" ;;  # Adjust for Rockchip
esac

sed -i "s|# INSERT_DEVICES_HERE|$DEVICES|g" /opt/frigate/docker-compose.yml

# Start Frigate
cd /opt/frigate
docker-compose up -d

echo "Setup complete for $ARCH with $ACCELERATOR. Edit /opt/frigate/config.yml for cameras."
```

## Arch Linux Setup

### Template Docker Compose File (docker-compose.yml)
Same as Debian; save in `/opt/frigate/`.

```yaml
version: "3.9"
services:
  frigate:
    container_name: frigate
    image: ghcr.io/blakeblackshear/frigate:stable
    restart: unless-stopped
    privileged: true
    devices:
      # INSERT_DEVICES_HERE
    volumes:
      - /opt/frigate/config.yml:/config/config.yml:ro
      - /opt/frigate/media:/media/frigate
      - type: tmpfs
        target: /tmp/cache
        tmpfs:
          size: 1000000000
    ports:
      - "5000:5000"
      - "8554:8554"
    environment:
      FRIGATE_RTSP_PASSWORD: "password"
```

### Bash CLI Script for Arch (setup_nvr_arch.sh)
Run with `./setup_nvr_arch.sh --arch x86 --accelerator coral`. Assumes yay for AUR; install yay first if needed. Run as user with sudo.

```bash
#!/bin/bash

# Usage: ./setup_nvr_arch.sh --arch <x86|arm> --accelerator <coral|intel|rockchip|none>

ARCH=""
ACCELERATOR="none"

while [[ $# -gt 0 ]]; do
  case $1 in
    --arch) ARCH="$2"; shift 2 ;;
    --accelerator) ACCELERATOR="$2"; shift 2 ;;
    *) echo "Unknown option $1"; exit 1 ;;
  esac
done

if [ -z "$ARCH" ]; then
  echo "Usage: $0 --arch <x86|arm> --accelerator <coral|intel|rockchip|none>"
  exit 1
fi

# Update and install base packages
sudo pacman -Syu --noconfirm
sudo pacman -S --noconfirm docker docker-compose ffmpeg

# Install hardware-specific packages (use AUR via yay if needed)
case $ACCELERATOR in
  coral)
    # Assume yay installed; install libedgetpu from AUR
    yay -S --noconfirm libedgetpu
    ;;
  intel)
    sudo pacman -S --noconfirm intel-media-driver
    ;;
  rockchip)
    # For Rockchip NPU, may need AUR package
    yay -S --noconfirm rockchip-mpp  # Adjust if package name differs
    ;;
  none)
    echo "No accelerator selected; using CPU."
    ;;
esac

# Create directories
mkdir -p /opt/frigate

# Download template compose if not exists
if [ ! -f "/opt/frigate/docker-compose.yml" ]; then
  curl -o /opt/frigate/docker-compose.yml https://raw.githubusercontent.com/blakeblackshear/frigate/master/docker/docker-compose.yml  # Or use your template
fi

# Modify devices in compose
DEVICES=""
case $ACCELERATOR in
  coral) DEVICES="- /dev/apex_0:/dev/apex_0" ;;
  intel) DEVICES="- /dev/dri:/dev/dri" ;;
  rockchip) DEVICES="- /dev/dri:/dev/dri - /dev/mali0:/dev/mali0" ;;
esac

sed -i "s|# INSERT_DEVICES_HERE|$DEVICES|g" /opt/frigate/docker-compose.yml

# Enable and start Docker
sudo systemctl enable --now docker

# Start Frigate
cd /opt/frigate
docker-compose up -d

echo "Setup complete for $ARCH with $ACCELERATOR. Edit /opt/frigate/config.yml for cameras."
```

## Notes
- **Customization**: After running the script, create/edit `/opt/frigate/config.yml` with Frigate settings (cameras, detectors). For accelerators, add corresponding detector configs (e.g., `detectors: coral: type: edgetpu`).
- **ARM64**: Ensure the system is aarch64; scripts assume compatible packages.
- **Dependencies**: For Arch, install `yay` manually if not present (`git clone https://aur.archlinux.org/yay.git && cd yay && makepkg -si`).
- **Testing**: Verify hardware access inside Docker with `docker exec -it frigate ls /dev`.