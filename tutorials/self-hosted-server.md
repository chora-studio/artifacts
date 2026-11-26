---
type: artifact
category: tutorials
status: active
author: Chora Studio
title: Self-Hosted Server
description: A tutorial for running a self-hosted server.
created: 2026-11-18
updated: 2026-11-26
order: 0
---

# Self-Hosted Server

This is a tutorial for running a self-hosted server.

## Index

- [Self-Hosted Server (Part 1)](./self-hosted-server-1)
- [Self-Hosted Server (Part 2)](./self-hosted-server-2)
- [Self-Hosted Server (Part 3)](./self-hosted-server-3)

## Overview

*A brief overview of the tutorial.*

### Part 1

The [first part](./self-hosted-server-1) covers the following:

- NixOS Installation
- NixOS Configuration
- SSH and Password

### Part 2

The [second part](./self-hosted-server-2) covers the following:

- Tunnel Service
- Forgejo Service
- Runner Service
- Website Service
- Forgejo Actions

### Part 3

The [third part](./self-hosted-server-3) covers the following:

- Tailscale Service
- Backup Device
- Rustic Service

## Software

*A brief overview of the required software.*

### Nix (or zstd)

Nix (or `zstd`) is only needed for the initial NixOS installation. Other tools can be used but they are currently not covered in this tutorial.

#### Requirements

- v2 (or later)

## Hardware

*A brief overview of the required hardware.*

### Keyboard

You'll need a keyboard for the initial device setup.

### Monitor

You'll need a monitor and HDMI cable for the initial device setup.

### ARM-64 Device

This device is your physical server. A Raspberry Pi 4 Model B was used for a test setup but any ARM-64 device that meets the hardware requirements listed below could also be used. Any other device that can run NixOS could also be used but they are currently not covered in this tutorial.

#### Requirements

- 4GB RAM (or more)

### SD Card

Depending on your device, a micro SD card and converter or a standard SD card. A Raspberry Pi uses a micro SD card, so the converter is needed for the laptop or computer that initially installs NixOS. Other storage devices could also be used but they are currently not covered in this tutorial.

#### Requirements

- 8GB Storage (or more)

## Services

*A brief overview of the required services.*

### Cloudflare

Cloudflare provides great services but users are required to trust Cloudflare with their identity and the ability to monitor their traffic. Alternative options are being explored but they come with heavy setup and maintenance costs and are therefore currently out of scope for this tutorial. No additional costs or subscriptions are required for this tutorial.

#### Requirements

- Cloudflare account
- Pre-purchased domain

### Tailscale

Tailscale provides great services but users are required to trust Tailscale with their identity and network topology metadata. Alternative options are being explored but they come with heavy setup and maintenance costs and are therefore currently out of scope for this tutorial. No additional costs or subscriptions are required for this tutorial.

#### Requirements

- Tailscale account

## Next Steps

- [Part 1](./self-hosted-server-1)
