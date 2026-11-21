---
type: artifact
category: tutorials
status: active
author: Chora Studio
title: Self-Hosted Server
description: A tutorial for running a self-hosted server.
created: 2026-11-18
updated: 2026-11-18
---

# Self-Hosted Server

This is a tutorial for running a self-hosted server. This setup is experimental and still being tested; this setup is not recommended for production.

## Prerequisites

#### Software

- Nix

#### Hardware

- Raspberry Pi 4 Model B (or later)
- SD Card (San Disk recommended)

#### Services

- Cloudflare (with a purchased domain)

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

## Adding a Cloudflared Service

Proceed with Cloudflare website and the Raspberry Pi terminal.

#### 1. Create Cloudflare tunnel

Open Cloudflare and create a new tunnel. The name of the tunnel can be whatever you want but using the same name as the hostname might make it easier to remember since only one tunnel will be used for this device.

#### 2. Create connector token

Copy any cloudflared connector command that includes a token hash. This tutorial does not use the cloudflared binary. Paste the content somewhere safe and then cut the token hash and delete the remainder.

Run the following to decrypt the token hash:

```
echo "TOKEN_HASH" | tr '_-' '/+' | base64 -d
```

Copy the JSON output and create a new file:

```
sudo mkdir /etc/cloudflared
sudo nano /etc/cloudflared/credentials.json 
```

Paste the JSON output and update the format:

```
{
  "AccountTag": "<VALUE_A>",
  "TunnelID": "<VALUE_T>",
  "TunnelSecret": "<VALUE_S>"
}
```

#### 3. Update Nixos configuration

Open the NixOS configuration file:

```
sudo nano /etc/nixos/configuration.nix
```

Add the following group and user:

```nix
  users.groups.cloudflared = { };
  users.users.cloudflared = {
    group = "cloudflared";
    isSystemUser = true;
  };
```

Add the following service:

```nix
  services.cloudflared = {
    enable = true;
    tunnels.example = {
      credentialsFile = "/etc/cloudflared/credentials.json";
      ingress = {
        "dev.example.com" = "http://localhost:8000";
        "git.example.com" = "http://localhost:3000";
      };
      default = "http_status:404";
    };
  };
```

Note the two domains added (`dev.example.com` and `git.example.com`). These domains will be configured in the following two sections.

#### 4. Rebuild and reboot

```
sudo nixos-rebuild switch
```

```
sudo reboot
```

## Adding a Website Service

Proceed with the Raspberry Pi terminal.

#### 1. Update Nixos configuration

Update the existing configuration file:

```
sudo nano /etc/nixos/configuration.nix
```

Add the new `./website-configuration.nix` import:

```nix
  imports = [
    ./hardware-configuration.nix
    ./website-configuration.nix
  ];
```

Add the following packages to the environment:

```nix
  environment.systemPackages = with pkgs; [
    git
  ];
```

Create `/etc/nixos/website-configuration.nix` file:

```
sudo nano /etc/nixos/website-configuration.nix
```

Add the following systemd service:

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

#### 2. Rebuild system

```
sudo nixos-rebuild switch
```

#### 3. Clone and build website

```
git clone https://github.com/example/website
```

```
cd website
```

```
git submodule init
```

```
git submodule update
```

```
nix develop
```

```
bun install
```

```
bun run build
```

#### 4. Restart service and check status

```
sudo systemctl restart website
```

```
sudo systemctl status website
```

#### 5. Create CNAME record

Open Cloudflare Tunnels and copy the tunnel ID, then open Cloudflare DNS and create a CNAME record. The name must match the subdomain used for the ingress value above (`dev`) and the target must use the following format `<TUNNEL_ID>.cfargotunnel.com` (using the copied token ID).

#### 6. Create tunnel route

Open Cloudflare Tunnels and create a tunnel route. Enter the subdomain, select the domain, and enter the service URL (`http://localhost:8000`).

## Adding a Forgejo Service

Proceed with the Raspberry Pi terminal.

#### 1. Update NixOS configuration

Update the configuration file:

```
sudo nano /etc/nixos/configuration.nix
```

Add the `forgejo` package to the environment:

```nix
  environment.systemPackages = with pkgs; [
    forgejo
    git
  ];
```

Add forgejo as a service:

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

#### 2. Rebuild system

```
sudo nixos-rebuild switch
```

#### 3. Check service status

```
sudo systemctl status forgejo
```

#### 4. Create admin user

```
sudo -u forgejo FORGEJO_WORK_DIR=/var/lib/forgejo forgejo admin user create --username user --email user@example.com --password password
```

#### 5. Create CNAME record

Open Cloudflare DNS and copy the target value you used for the CNAME record above. The name must match the subdomain (`git`).

#### 6. Create tunnel route

Open Cloudflare Tunnels and create a tunnel route. Enter the subdomain, select the domain, and enter the service URL (`http://localhost:3000`).

## References

- https://nixos.wiki/wiki/NixOS_on_ARM#Installation
