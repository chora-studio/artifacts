---
type: artifact
category: tutorials
status: active
author: Chora Studio
title: Self-Hosted Server (Part 3)
description: The third part of the self-hosted server tutorial.
created: 2026-11-22
updated: 2026-11-22
order: 3
---

# Self-Hosted Server (Part 3)

The third part of the self-hosted server tutorial. This part assumes you completed the [second part](./self-hosted-server-2) of the tutorial.

## Prerequisites

See software, hardware, and services on the [index page](./self-hosted-server). This tutorial also requires a second ARM-64 device and SD card for managing backups.

## Adding a Tailscale Service

#### 1. Create auth key

Open Tailscale settings, generate an auth key, and copy the key.

#### 2. Update NixOS Configuration

```nix
    firewall = {
      allowedTCPPorts = [ 22 ];
      trustedInterfaces = [ "tailscale0" ];
    };
```

```nix
  environment.systemPackages = with pkgs; [
    git
    forgejo
    tailscale
  ];
```

```nix
  services.tailscale.enable = true;
```

#### 3. Rebuild

```
sudo nixos-rebuild switch
```

#### 4. Add Machine

```
sudo tailscale up --auth-key=tskey-auth-XXXXXXX --hostname=<name>
```

#### 5. Verify Status

```
tailscale status
```

## Adding a Backup Device

With a second device, repeat [part 1](./self-hosted-server-1) of the tutorial series.

After repeating part 1 of the tutorial series with the second device, repeat the above section with the second device as well.

When the above steps are complete, the second device should have NixOS installed with OpenSSH and Tailscale services running.

## Adding a Rustic Service

Proceed with computer, then backup device, then server device.

#### 1. Generate SSH key

Proceed with computer.

```
ssh-keygen -t ed25519 -f ~/backup-raspberrypi -N '' -C 'backup@raspberrypi'
```

```
cat ~/backup-raspberrypi.pub
```

```
cat ~/backup-raspberrypi
```

#### 2. Update NixOS configuration (backup device)

Proceed with the backup device.

```
sudo nano /etc/nixos/configuration.nix
```

```nix
  users.users.backup = {
    isSystemUser = true;
    group = "backup";
    home = "/srv/backups";
    shell = "${pkgs.coreutils}/bin/false";
    openssh.authorizedKeys.keys = [
      "ssh-ed25519 AAAA... user@raspberrypi"
    ];
  };

  users.groups.backup = {};
```

```nix
  system.activationScripts.backup-dirs = ''
    install -d -o backup -g backup -m 0750 \
      /srv/backups /srv/backups/raspberyypi
  '';
```

```nix
  services.openssh.extraConfig = ''
    Match User backup
        ForceCommand internal-sftp
        AllowTcpForwarding no
        X11Forwarding no
        PermitTunnel no
  '';
```

```
sudo nixos-rebuild switch
```

#### 3. Update NixOS configuration file (server device)

Proceed with the server device.

```
sudo install -d -m 0700 /etc/rustic
```

```
sudo nano /etc/rustic/id_ed25519
```

```
sudo chmod 600 /etc/rustic/id_ed25519
```

```
sudo nano /etc/rustic/repo-password
```

```
sudo chmod 600 /etc/rustic/repo-password
```

```
sudo nano /etc/nixos/configuration.nix
```

```nix
  imports = [
    ./hardware-configuration.nix
    ./rustic-configuration.nix
    ./website-configuration.nix
  ];
```

```nix
  services.openssh = {
    enable = true;
    openFirewall = true;
    settings = {
      PermitRootLogin = "no";
      PasswordAuthentication = false;
    };
    knownHosts = {
      backup = {
        hostNames = [ "backup" ];
        publicKey = "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...";
      };
    };
  };
```

```
sudo nano /etc/nixos/rustic-configuration.nix
```

```nix
{ config, lib, pkgs, ... }:

{
  environment.systemPackages = with pkgs; [
    rustic
  ];

  environment.etc."rustic/backup.toml".text = ''
    [repository]
      repository = "opendal:sftp"
      password-file = "/etc/rustic/repo-password"

    [repository.options]
      endpoint = "ssh://backup"
      root = "/srv/backups/raspberrypi"
      user = "backup"
      key = "/etc/rustic/id_ed25519"

    [backup]
      one-file-system = true
      globs = [
        "!/nix/store",
        "!/var/tmp",
      ]

    [[backup.snapshots]]
      name = "root"
      sources = ["/"]

    [forget]
      keep-daily = 7
      keep-weekly = 4
      keep-monthly = 1
  '';

  environment.etc."ssh/ssh_config.d/rustic.conf".text = ''
    Host backup
      IdentityFile /etc/rustic/id_ed25519
      IdentitiesOnly yes
    HostKeyAlias backup
  '';

  systemd.services.rustic-backup = {
    after = [ "network-online.target" ];
    wants = [ "network-online.target" ];
    serviceConfig = {
      Type = "oneshot";
      User = "root";
      Nice = 10;
      IOSchedulingClass = "best-effort";
      IOSchedulingPriority = 7;
    };
    path = [ pkgs.rustic pkgs.openssh ];
    script = ''
      rustic -P backup backup
      rustic -P backup forget
      rustic -P backup prune
    '';
  };

  systemd.timers.rustic-backup = {
    wantedBy = [ "timers.target" ];
    timerConfig = {
      OnCalendar = "daily";
      Persistent = true;
    };
  };
}
```

```
sudo nixos-rebuild switch
```

```
sudo rustic -P backup init
```

```
sudo rustic -P backup backup
```

#### 4. Clean up key files

Proceed with computer.

```
rm backup-raspberrypi backup-raspberrypi.pub
```
