---
type: artifact
category: tutorials
status: active
author: Chora Studio
title: Self-Hosted Server (Part 3)
description: The third part of the self-hosted server tutorial.
created: 2026-11-22
updated: 2026-11-26
order: 3
---

# Self-Hosted Server (Part 3)

The third part of the self-hosted server tutorial. This part assumes you completed the [second part](./self-hosted-server-2) of the tutorial.

## Prerequisites

See the [index page](./self-hosted-server) for required software, hardware, and services. This part also requires a second ARM-64 device and SD card.

## Tailscale Service

#### 1. Create authorization key

Open Tailscale settings, generate an auth key, and copy the key.

#### 2. Update NixOS Configuration

Run the following:

```
sudo nano /etc/nixos/configuration.nix
```

Update the file to include the following:

```nix
    firewall = {
      allowedTCPPorts = [ 22 ];
      trustedInterfaces = [ "tailscale0" ];
    };
```

Update the file to include the following:

```nix
  environment.systemPackages = with pkgs; [
    forgejo
    tailscale
  ];
```

Update the file to include the following:

```nix
  services.tailscale.enable = true;
```

#### 3. Rebuild the system

Run the following:

```
sudo nixos-rebuild switch
```

#### 4. Add tailscale machine

Run the following:

```
sudo tailscale up --auth-key=tskey-auth-XXXXXXX --hostname=<name>
```

#### 5. Verify tailscale status

Run the following:

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

Run the following on the computer:

```
ssh-keygen -t ed25519 -f ~/backup-server -N '' -C 'backup@server'
```

Run the following:

```
cat ~/backup-server.pub
```

Run the following:

```
cat ~/backup-server
```

#### 2. Update NixOS configuration (backup device)

Run the following on the backup device:

```
sudo nano /etc/nixos/configuration.nix
```

Update the file to include the following:

```nix
  users.users.backup = {
    isSystemUser = true;
    group = "backup";
    home = "/srv/backups";
    shell = "${pkgs.coreutils}/bin/false";
    openssh.authorizedKeys.keys = [
      "ssh-ed25519 AAAA... user@server"
    ];
  };

  users.groups.backup = {};
```

Update the file to include the following:

```nix
  system.activationScripts.backup-dirs = ''
    install -d -o backup -g backup -m 0750 \
      /srv/backups /srv/backups/raspberyypi
  '';
```

Update the file to include the following:

```nix
  services.openssh.extraConfig = ''
    Match User backup
        ForceCommand internal-sftp
        AllowTcpForwarding no
        X11Forwarding no
        PermitTunnel no
  '';
```

Run the following:

```
sudo nixos-rebuild switch
```

#### 3. Update NixOS configuration file (server device)

Run the following on the server device:

```
sudo install -d -m 0700 /etc/rustic
```

Run the following:

```
sudo nano /etc/rustic/id_ed25519
```

Run the following:

```
sudo chmod 600 /etc/rustic/id_ed25519
```

Run the following:

```
sudo nano /etc/rustic/repo-password
```

Run the following:

```
sudo chmod 600 /etc/rustic/repo-password
```

Run the following:

```
sudo nano /etc/nixos/configuration.nix
```

Update the file to include the following:

```nix
  imports = [
    ./hardware-configuration.nix
    ./runner-configuration.nix
    ./rustic-configuration.nix
    ./website-configuration.nix
  ];
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
    knownHosts = {
      backup = {
        hostNames = [ "backup" ];
        publicKey = "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...";
      };
    };
  };
```

Run the following:

```
sudo nano /etc/nixos/rustic-configuration.nix
```

Update the file to include the following:

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
      root = "/srv/backups/server"
      user = "backup"
      key = "/etc/rustic/id_ed25519"

    [backup]
      one-file-system = true
      globs = [
        "!/nix/store",
        "!/var/cache",
        "!/var/log",
        "!/var/tmp",
      ]

    [[backup.snapshots]]
      name = "root"
      sources = ["/"]

    [[backup.snapshots]]
      name = "nixos"
      sources = ["/etc/nixos"]

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

Run the following:

```
sudo nixos-rebuild switch
```

Run the following:

```
sudo rustic -P backup init
```

Run the following:

```
sudo rustic -P backup backup
```

#### 4. Clean up key files

Run the following on the computer:

```
rm backup-server backup-server.pub
```
