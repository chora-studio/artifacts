---
type: artifact
category: tutorials
status: active
author: Chora Studio
title: Self-Hosted Server (Part 1)
description: The first part of the self-hosted server tutorial.
created: 2026-11-22
updated: 2026-11-26
order: 1
---

# Self-Hosted Server (Part 1)

The first part of the self-hosted server tutorial.

## Prerequisites

See the [index page](./self-hosted-server) for required software, hardware, and services.

## NixOS Installation

Insert the SD card into the computer before proceeding. The computer will need to have `nix` installed (or `zstd`) in order to decompress the image. All of the commands in this section are for the terminal on the computer.

#### 1. Download the latest SD card image

Run the following:

```
curl https://hydra.nixos.org/build/333653849/download/1/nixos-image-sd-card-25.11.12484.b6018f87da91-aarch64-linux.img.zst --output nixos-image-sd-card-25.11.12484.b6018f87da91-aarch64-linux.img.zst
```

#### 2. Verify the checksum of the file

Run the following (Linux):

```
sha256sum nixos-image-sd-card-25.11.12484.b6018f87da91-aarch64-linux.img.zst
```

Run the following (MacOS):

```
shasum -a 256 nixos-image-sd-card-25.11.12484.b6018f87da91-aarch64-linux.img.zst
```

Verify the following output:

```
76c485d9455baf0c559d1ab4c330a1b3ba8318c6269b160ee47df01cdabbba3e  nixos-image-sd-card-25.11.12484.b6018f87da91-aarch64-linux.img.zst
```

#### 3. Decompress the downloaded SD card image

Run the following (using `nix`):

```
nix-shell -p zstd --run "unzstd nixos-image-sd-card-25.11.12484.b6018f87da91-aarch64-linux.img.zst"
```

Run the following (using `zstd`):

```
unzstd nixos-image-sd-card-25.11.12484.b6018f87da91-aarch64-linux.img.zst
```

#### 4. Verify the name of the target disk (the SD card)

Run the following:

```
lsblk
```

Verify the following output (Linux):

```
/dev/mmcblk0
```

Verify the following output (MacOS):

```
/dev/rdisk2
```

#### 5. Write the decompressed image to the target disk

Run the following (Linux):

```
sudo dd if=nixos-image-sd-card-25.11.12484.b6018f87da91-aarch64-linux.img of=/dev/mmcblk0 bs=10MB oflag=dsync status=progress
```

Run the following (MacOS):

```
sudo dd if=nixos-image-sd-card-25.11.12484.b6018f87da91-aarch64-linux.img of=/dev/rdisk2 bs=10m oflag=sync status=progress
```

#### 6. Unmount the disk before removing the SD card

Run the following (Linux):

```
sudo umount /dev/mmcblk0*
```

Run the following (MacOS):

```
sudo diskutil unmountDisk /dev/disk2
```

## NixOS Configuration

Insert the SD card into the ARM-64 device before proceeding. You will need a monitor and keyboard plugged into the device. All of the commands in this section are for the terminal on the ARM-64 device.

#### 1. Connect with WiFi (or ethernet)

Run the following:

```
nmtui
```

#### 2. Generate NixOS configuration files

Run the following:

```
sudo nixos-generate-config
```

#### 3. Update NixOS configuration file

Run the following:

```
sudo nano /etc/nixos/configuration.nix
```

Update the file to include the following:

```nix
{ config, lib, pkgs, ... }:

{
  imports = [ ./hardware-configuration.nix ];

  boot.loader.grub.enable = false;
  boot.loader.generic-extlinux-compatible.enable = true;

  networking = {
    hostName = "server";
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

Save and exit the file editor.

#### 4. Rebuild and reboot 

Run the following:

```
sudo nixos-rebuild switch
```

Manually turn off the device and then turn it back on.

## SSH and Password

The monitor and keyboard no longer need to be plugged into the ARM-64 device. The remainder of this tutorial will use the terminal on the computer to access the terminal on the ARM-64 device.

#### 1. Generate SSH key

Run the following:

```
ssh-keygen -t ed25519
```

#### 2. Copy SSH ID to ARM-64 device

Run the following:

```
ssh-copy-id user@server
```

Make sure the user (`user`) and hostname (`server`) match what was provided in the configuration file from the previous section.

#### 3. SSH with password authentication

Run the following:

```
ssh user@server
```

When prompted for a password, use the initial password (`password`) provided in the configuration file from the previous section.

#### 4. Disable SSH password authentication

Run the following:

```
sudo nano /etc/nixos/configuration.nix
```

Update the file to include the following:

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

Save and exit the file editor.

#### 5. Update system user password

Run the following:

```
passwd
```

Enter a more secure password.

#### 6. Rebuild and reboot

Run the following:

```
sudo nixos-rebuild switch
```

Run the following:

```
sudo reboot
```

## Next Steps

- [Part 2](./self-hosted-server-2)
