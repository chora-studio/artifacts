---
type: artifact
category: tutorial
status: active
author: Chora Studio
title: Self-Hosted Servers
description: A tutorial on self-hosted servers.
created: 2026-11-18
updated: 2026-11-18
---

# Self-Hosted Servers

This is a tutorial for a self-hosted server. This is an **experimental** setup that is still being tested and is **not production ready**.

## Prerequisites

#### Software

- Nix

#### Hardware

- Raspberry Pi 4 Model B
- SD Card (San Disk recommended)

#### Services

- Cloudflare (with a purchased domain)

## Installing Nixos

Insert SD card into computer and proceed with the following steps.

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

## Setting Up Nixos

Insert SD card into Raspberry Pi and proceed with the following steps.

#### 1. Connect with WiFi (or ethernet)

```
nmtui
```

#### 2. Generate Nixos configuration files

```
sudo nixos-generate-config
```

#### 3. Update Nixos configuration file

```
nano /etc/nixos/configuration.nix
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
}
```

#### 4. Rebuild and reboot 

```
nixos-rebuild switch
```

```
reboot
```

## SSH Access and Security

Start from computer and proceed with Raspberry Pi (following step 3).

#### 1. Generate SSH key (if none)

```
ssh-keygen -t ed25519
```

#### 2. Copy SSH key to Raspberry Pi

```
ssh-copy-id user@<pi-ip>
```

#### 3. Test SSH with password authentication

```
ssh user@<pi-ip>
```

#### 4. Disable SSH password authentication

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

#### 5. Change to more secure password

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

## Add Cloudflared and Caddy

Proceed with Cloudflare website and Raspberry Pi.

#### 1. Set up Cloudflare tunnel

...

#### 2. Update Nixos configuration

```nix
  imports = [
    ./hardware-configuration.nix
    ./website-configuration.nix
  ];
```

```nix
  users.groups.cloudflared = { };
  users.users.cloudflared = {
    group = "cloudflared";
    isSystemUser = true;
  };
```

```nix
  services.cloudflared = {
    enable = true;
    tunnels.example = {
      credentialsFile = "/etc/cloudflared/credentials.json";
      ingress = {
        "example.com" = "http://localhost:8000";
      };
      default = "http_status:404";
    };
  };
```

```nix
{ config, pkgs, ... }:

let
  caddyConfig = pkgs.writeText "Caddyfile" ''
    :8000 {
      root * /home/user/website/dist
      encode gzip zstd
      try_files {path} {path}/index.html {path}.html {path}/ /index.html
      file_server
    }
  '';
in
{
  systemd.services.website = {
    description = "website";
    after = [ "network.target" ];
    wants = [ "network.target" ];
    serviceConfig = {
      ExecStart = "${pkgs.caddy}/bin/caddy run --config ${caddyConfig} --adapter caddyfile";
      Restart = "always";
      RestartSec = "5";
    };
  };
}
```

#### 3. Test Cloudflare tunnel

...

## Add Forgejo

Proceed with Raspberry Pi.

#### 1. Update NixOS configuration

```nix
      ingress = {
        "example.com" = "http://localhost:8000";
        "git.example.com" = "http://localhost:3000";
      };
```

```nix
  services.forgejo = {
    enable = true;
    database.type = "sqlite3";
    settings.server.DOMAIN = "git.example.com";
    settings.server.ROOT_URL = "https://git.example.com/";
    settings.server.DISABLE_SSH = true;
    settings.service.DISABLE_REGISTRATION = true;
    settings.security.REVERSE_PROXY_TRUSTED_PROXIES = "127.0.0.1/32";
  };
```

#### 2. Create Admin User

```
sudo -u forgejo FORGEJO_WORK_DIR=/var/lib/forgejo forgejo admin user create --username user --email user@example.com --password password
```

#### 3. Test Forgejo

...

## References

- https://nixos.wiki/wiki/NixOS_on_ARM#Installation
