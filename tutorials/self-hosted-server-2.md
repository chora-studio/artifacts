---
type: artifact
category: tutorials
status: active
author: Chora Studio
title: Self-Hosted Server (Part 2)
description: The second part of the self-hosted server tutorial.
created: 2026-11-22
updated: 2026-11-26
order: 2
---

# Self-Hosted Server (Part 2)

The second part of the self-hosted server tutorial. This part assumes you completed the [first part](./self-hosted-server-1) of the tutorial.

## Prerequisites

See the [index page](./self-hosted-server) for required software, hardware, and services.

## Tunnel Service

Proceed with the Cloudflare website and the terminal of your server.

#### 1. Create Cloudflare tunnel

Open Cloudflare and create a new tunnel. The name of the tunnel can be whatever you want but using the same name as the hostname might make it easier to remember since only one tunnel will be used for this device.

#### 2. Create connector token

Copy the token hash from one of the cloudflared connector commands on the tunnels page. This tutorial does not use the cloudflared binary.

Run the following to decrypt the token hash:

```
echo "TOKEN_HASH" | tr '_-' '/+' | base64 -d
```

Copy the JSON output and then create a new directory and file:

```
sudo mkdir /etc/cloudflared
```

```
sudo nano /etc/cloudflared/credentials.json 
```

Paste the JSON output into the file and update the field names:

```
{
  "AccountTag": "<VALUE_A>",
  "TunnelID": "<VALUE_T>",
  "TunnelSecret": "<VALUE_S>"
}
```

Save and exit the file editor.

#### 3. Update Nixos configuration

Run the following:

```
sudo nano /etc/nixos/configuration.nix
```

Update the file to include the following:

```nix
  users.groups.cloudflared = { };
  users.users.cloudflared = {
    group = "cloudflared";
    isSystemUser = true;
  };
```

Update the file to include the following:

```nix
  services.cloudflared = {
    enable = true;
    tunnels.default = {
      credentialsFile = "/etc/cloudflared/credentials.json";
      ingress = {
        "example.com" = "http://localhost:8000";
        "dev.example.com" = "http://localhost:8080";
        "git.example.com" = "http://localhost:3000";
        "www.example.com" = "http://localhost:8000";
      };
      default = "http_status:404";
    };
  };
```

Save and exit the file editor.

#### 4. Rebuild the system

Run the following:

```
sudo nixos-rebuild switch
```

#### 5. Check service status

Run the following:

```
sudo systemctl status cloudflared-tunnel-default
```

## Forgejo Service

Proceed with the terminal of your server.

#### 1. Update NixOS configuration

Run the following:

```
sudo nano /etc/nixos/configuration.nix
```

Update the file to include the following:

```nix
  environment.systemPackages = with pkgs; [
    forgejo
  ];
```

Update the file to include the following:

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

Save and exit the file editor.

#### 2. Rebuild the system

Run the following:

```
sudo nixos-rebuild switch
```

#### 3. Check service status

Run the following:

```
sudo systemctl status forgejo
```

#### 4. Create admin user

Run the following:

```
sudo -u forgejo FORGEJO_WORK_DIR=/var/lib/forgejo forgejo admin user create --username user --email user@example.com --password password
```

#### 5. Create CNAME record

Open Cloudflare Tunnels and copy the tunnel ID, then open Cloudflare DNS and create a CNAME record for the `git` subdomain. The target must use the following format `<TUNNEL_ID>.cfargotunnel.com`.

#### 6. Create tunnel route

Open Cloudflare Tunnels and create a tunnel route. Enter the subdomain, select the domain, and enter the service URL (`http://localhost:3000`).

## Runner Service

Run the following:

```
sudo nano /etc/nixos/configuration.nix
```

Update the file to include the following:

```nix
  imports = [
    ./hardware-configuration.nix
    ./runner-configuration.nix
  ];
```

Update the file to include the following:

```nix
  services.forgejo = {
    enable = true;
    database.type = "sqlite3";
    settings.server.DOMAIN = "git.example.com";
    settings.server.ROOT_URL = "https://git.example.com/";
    settings.server.DISABLE_SSH = true;
    settings.service.DISABLE_REGISTRATION = true;
    settings.security.REVERSE_PROXY_TRUSTED_PROXIES = "127.0.0.1/32";
    settings.actions.ENABLED = true;
    settings.actions.DEFAULT_ACTIONS_URL = "github";
  };
```

Run the following:

```
sudo nano /etc/nixos/runner-configuration.nix
```

Update the file to include the following:

```nix
{ config, lib, pkgs, ... }:

{
  systemd.services."gitea-runner-default".serviceConfig.DynamicUser = lib.mkForce false;

  users.users.gitea-runner = {
    isSystemUser = true;
    group = "gitea-runner";
    home = "/var/lib/gitea-runner";
    createHome = true;
    shell = pkgs.bash;
  };

  users.groups.gitea-runner = { };

  systemd.tmpfiles.rules = [
    "d /var/lib/gitea-runner 0700 gitea-runner gitea-runner -"
    "d /srv/dev 0755 gitea-runner gitea-runner -"
    "d /srv/www 0755 gitea-runner gitea-runner -"
  ];

  services.gitea-actions-runner = {
    package = pkgs.forgejo-runner;
    instances.default = {
      enable = true;
      name = config.networking.hostName;
      url = "http://localhost:3000";
      token = "TODO";
      labels = [ "native:host" ];
      hostPackages = with pkgs; [
        bash
        bun
        git
        nodejs
      ];
      settings = {
        log.level = "info";
        runner.capacity = 1;
        cache.dir = "/var/lib/gitea-runner/.cache/actcache";
        host.workdir_parent = "/var/lib/gitea-runner/.cache/act";
      };
    };
  };

  systemd.services."gitea-runner-default".serviceConfig.ReadWritePaths =
    lib.mkAfter [ "/srv/dev" "/srv/www" ];
}
```

Open forgejo in a web browser and then navigate to the settings of the admin user or organization that will host your website. Go to the settings for runners and copy the runner token and replace the `TODO` above with the runner token. After updating the token, save and exit the file editor.

Run the following:

```
sudo nixos-rebuild switch
```

## Website Service

Proceed with the terminal of your server.

#### 1. Update Nixos configuration

Run the following:

```
sudo nano /etc/nixos/configuration.nix
```

Update the file to include the following:

```nix
  imports = [
    ./hardware-configuration.nix
    ./runner-configuration.nix
    ./website-configuration.nix
  ];
```

Run the following:

```
sudo nano /etc/nixos/website-configuration.nix
```

Update the file to include the following:

```nix
{ config, pkgs, ... }:

let
  caddyConfig = pkgs.writeText "Caddyfile" ''
    :8000 {
      root * /srv/www
      encode gzip zstd
      try_files {path} {path}/index.html {path}.html {path}/ /index.html
      file_server
    }
    :8080 {
      root * /srv/dev
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

#### 2. Rebuild the system

```
sudo nixos-rebuild switch
```

#### 3. Check service status

```
sudo systemctl status website
```

#### 4. Create CNAME record

Open Cloudflare DNS and copy the target used above (same target as the CNAME record for the `git` subdomain), then create CNAME records for the root domain and the `dev` and `www` subdomains using the same target.

#### 5. Create tunnel route

Open Cloudflare Tunnels and create tunnel routes for the root domain and the `dev` and `www` subdomains. Enter root or the subdomain name, select the domain, and enter the service URL (`http://localhost:8000` for the root and the `www` subdomain and `http://localhost:8080` for the `dev` subdomain).

## Forgejo Actions

#### 1. Add dev workflow

Run the following inside your website repository:

```
mkdir -p .forgejo/workflows
```

```
nano .forgejo/workflows/dev.yml
```

Update the file to include the following:

```yml
on:
  push:
    branches:
      - 'dev'

jobs:
  build:
    runs-on: native
    steps:
      - name: checkout
        uses: actions/checkout@v4
      - name: submod
        run: |
            git submodule init
            git submodule update
      - name: install
        run: bun install
      - name: build
        run: bun run build
      - name: deploy
        run: |
            rm -rf /srv/dev/*
            cp -r dist/. /srv/dev/
```

Save and exit the file editor.

#### 2. Add www workflow

Run the following inside your website repository:

```
nano .forgejo/workflows/www.yml
```

Update the file to include the following:

```yml
on:
  push:
    branches:
      - 'www'

jobs:
  build:
    runs-on: native
    steps:
      - name: checkout
        uses: actions/checkout@v4
      - name: submod
        run: |
            git submodule init
            git submodule update
      - name: install
        run: bun install
      - name: build
        run: bun run build
      - name: deploy
        run: |
            rm -rf /srv/www/*
            cp -r dist/. /srv/www/
```

#### 3. Commit and push changes

Commit and push changes to `main` and then `dev` on your forgejo instance.

## Next Steps

- [Part 3](./self-hosted-server-3)
