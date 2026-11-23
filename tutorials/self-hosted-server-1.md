---
type: artifact
category: tutorials
status: active
author: Chora Studio
title: Self-Hosted Server (Part 1)
description: The first part of the self-hosted server tutorial.
created: 2026-11-22
updated: 2026-11-22
order: 1
---

# Self-Hosted Server (Part 1)

The first part of the self-hosted server tutorial.

## Prerequisites

See software, hardware, and services on the [index page](./self-hosted-server).

## Installing NixOS

Insert the SD card into the computer before proceeding. You will need a browser with internet access and `nix` installed on the computer.

#### 1. Download the latest SD card image

```
https://hydra.nixos.org/job/nixos/release-25.11/nixos.sd_image.aarch64-linux
```

#### 2. Decompress the downloaded SD card image

```
nix-shell -p zstd --run "unzstd <img-name>.img.zst"
```

#### 3. Write the downloaded image to the SD card

```
sudo dd if=<img-name>.img of=/dev/mmcblk0 bs=10MB oflag=dsync status=progress
```

## Setting Up NixOS

Insert the SD card into the Raspberry Pi before proceeding. You will need a monitor and keyboard plugged into the Raspberry Pi.

#### 1. Connect with WiFi (or ethernet)

```
nmtui
```

#### 2. Generate NixOS configuration files

```
sudo nixos-generate-config
```

#### 3. Update NixOS configuration file

```
sudo nano /etc/nixos/configuration.nix
```

```nix
{ config, lib, pkgs, ... }:

{
  imports = [ ./hardware-configuration.nix ];

  boot.loader.grub.enable = false;
  boot.loader.generic-extlinux-compatible.enable = true;

  networking = {
    hostName = "raspberrypi";
    firewall = {
      allowedTCPPorts = [ 22 ];
    };
    networkmanager = {
      enable = true;
      wifi.powersave = false;
    };
  };

  users.users.user = {
    isNormalUser = true;
    extraGroups = [ "wheel" "networkmanager" ];
    initialPassword = "password";
  };

  services.openssh = {
    enable = true;
    openFirewall = true;
  };

  system.stateVersion = "25.11";
  
  nix.settings.experimental-features = [ "nix-command" "flakes" ];
}
```

#### 4. Rebuild and reboot 

```
sudo nixos-rebuild switch
```

Manually turn off Raspberry Pi and then turn back on.

## SSH Access and Security

Start from the computer and then proceed with the Raspberry Pi terminal (following step 3). The monitor and keyboard used with the Raspberry Pi are no longer needed now that SSH is enabled on the device.

#### 1. Generate SSH key

```
ssh-keygen -t ed25519
```

#### 2. Copy SSH key to Raspberry Pi

```
ssh-copy-id user@raspberrypi
```

#### 3. SSH with password authentication

```
ssh user@raspberrypi
```

#### 4. Disable SSH password authentication

```
sudo nano /etc/nixos/configuration.nix
```

```nix
  services.openssh = {
    enable = true;
    openFirewall = true;
    settings = {
      PermitRootLogin = "no";
      PasswordAuthentication = false;
    };
  };
```

#### 5. Update system user password

```
passwd
```

#### 6. Rebuild and reboot

```
sudo nixos-rebuild switch
```

```
sudo reboot
```

## Next Steps

- [Part 2](./self-hosted-server-2)
