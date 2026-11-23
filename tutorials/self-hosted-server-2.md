---
type: artifact
category: tutorials
status: active
author: Chora Studio
title: Self-Hosted Server (Part 2)
description: The second part of the self-hosted server tutorial.
created: 2026-11-22
updated: 2026-11-22
order: 2
---

# Self-Hosted Server (Part 2)

The second part of the self-hosted server tutorial. This part assumes you completed the [first part](./self-hosted-server-1) of the tutorial.

## Prerequisites

See software, hardware, and services on the [index page](./self-hosted-server).


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
    forgejo
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

## Next Steps

- [Part 3](./self-hosted-server-3)
