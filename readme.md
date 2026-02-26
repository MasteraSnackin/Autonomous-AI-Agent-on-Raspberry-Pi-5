# Autonomous AI Agent on Raspberry Pi 5 (OpenClaw + Gemini / Ollama)

[![Node.js Version](https://img.shields.io/badge/node-%3E%3D22.0-brightgreen.svg)](https://nodejs.org/)
[![Raspberry Pi](https://img.shields.io/badge/Raspberry%20Pi-5%20(8GB)-C51A4A.svg?logo=raspberry-pi)](https://www.raspberrypi.com/products/raspberry-pi-5/)
[![OS](https://img.shields.io/badge/OS-Raspberry%20Pi%20OS%20Lite%2064--bit-blue.svg)](https://www.raspberrypi.com/software/)
[![OpenClaw](https://img.shields.io/badge/OpenClaw-latest-orange.svg)](https://docs.openclaw.org/)
[![License](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

This project turns a Raspberry Pi 5 (8 GB) into a production‑grade autonomous AI agent using **OpenClaw**, with Google Gemini 2.5 Flash as the primary model and optional local models via Ollama. It supports both microSD‑only and NVMe‑accelerated setups (NVMe is optional but strongly recommended for 24/7 or local‑model use).

## Table of contents

- [Overview](#overview)
- [Assumptions](#assumptions)
- [Features](#features)
- [Hardware and prerequisites](#hardware-and-prerequisites)
- [Quick start](#quick-start)
- [System setup and performance tuning](#system-setup-and-performance-tuning)
- [Optional: local models with Ollama](#optional-local-models-with-ollama)
- [OpenClaw installation and configuration](#openclaw-installation-and-configuration)
- [Telegram bot and secure pairing](#telegram-bot-and-secure-pairing)
- [Modes: cloud Gemini vs local Ollama](#modes-cloud-gemini-vs-local-ollama)
- [Running in production](#running-in-production)
- [Roadmap / extensions](#roadmap--extensions)
- [License](#license)

## Overview

The goal of this repo is to provide a repeatable blueprint for running an autonomous agent on a Raspberry Pi 5 that is safe enough for home/lab use and robust enough for 24/7 operation. The guide favours explicit, manual steps so you always understand what is running on the device.

## Assumptions

- Single‑user or small private setup (Telegram direct messages only, no public groups).
- Raspberry Pi 5 (8 GB) with active cooling, intended to run 24/7 in a safe location.
- You are comfortable with SSH and basic Linux administration.

For multi‑tenant or internet‑exposed deployments, add additional isolation (Docker/VM, firewall rules, read‑only filesystems) beyond what is shown here.

## Features

- **OpenClaw** autonomous agent framework with Telegram as the primary control channel.
- Cloud mode using **Gemini 2.5 Flash** for low‑latency reasoning and tools.
- Optional local mode via **Ollama** (for example `qwen2.5:3b`) running directly on the Pi.
- Optional NVMe‑booted Raspberry Pi OS for fast I/O and reduced SD wear (microSD‑only setups still work).
- Aggressive active cooling curve and 8 GB NVMe‑backed swap for sustained workloads on 8 GB RAM.
- Manual, transparent setup steps, plus an optional one‑line OpenClaw installer for a faster path.

## Hardware and prerequisites

### Tested hardware

- Raspberry Pi 5, 8 GB RAM.
- Official Raspberry Pi Active Cooler (4‑pin fan header).
- 32 GB (or larger) microSD card – required and sufficient for light, cloud‑only agents.
- NVMe HAT for Pi 5 plus NVMe SSD (for example 256 GB+) – optional but strongly recommended for 24/7 agents, heavier logging, or local models.
- Official 27 W Raspberry Pi USB‑C power supply or equivalent; stable power is critical under AI load.

### Software baseline

- Raspberry Pi OS Lite (64‑bit, Bookworm or newer).
- SSH access from your workstation.
- Google Gemini API key and a Telegram account (for bot admin).

## Quick start

1. Install Raspberry Pi OS Lite (64‑bit) on SD, enable SSH, boot, and update the system.
2. Configure an aggressive fan curve in `config.txt` for AI workloads.
3. (Optional, recommended) Clone the SD to an NVMe SSD with `rpi-clone`, set NVMe boot, and enable PCIe Gen 3 on the Pi 5.
4. Create an 8 GB swap file (NVMe preferred) to avoid OOM during heavy inference.
5. Install Node.js 22 via `nvm` and the required build tools.
6. Install OpenClaw globally with compilation flags that make audio dependencies build cleanly on ARM64.
7. Create the OpenClaw agent config for Gemini + Telegram, then run the onboarding wizard in daemon mode.
8. Pair your Telegram account and start chatting with your Pi‑hosted agent.

If you do not have NVMe hardware, you can skip the NVMe section and run entirely from a good A2‑class microSD card; installs and logging‑heavy workloads will simply be slower.

## System setup and performance tuning

### Install Raspberry Pi OS (temporary SD boot)

- Use Raspberry Pi Imager with `Raspberry Pi OS Lite (64‑bit)`, set hostname, enable SSH and create your user.
- Boot the Pi, SSH in and run `sudo apt update && sudo apt full-upgrade -y`, then reboot.

### Aggressive thermal management

- Edit `/boot/firmware/config.txt` and add the provided aggressive fan curve for AI workloads.
- Reboot so the new cooling profile is applied, favouring sustained performance over noise.

### (Optional) Migrate from SD to NVMe

- Install `rpi-clone`, clone the SD card to the NVMe device, and set NVMe/USB as the boot target.
- Enable PCIe Gen 3 (`dtparam=pciex1_gen=3`) in `config.txt` and reboot.

### Create 8 GB swap (NVMe preferred)

- Create an 8 GB swap file on NVMe (or SD if needed), add it to `/etc/fstab`, and verify with `free -h`.
- On NVMe, 8 GB swap is safe and fast; on microSD it is workable but will increase card wear over time.

## Optional: local models with Ollama

- Install Ollama on the Pi and pull a 3B‑class model such as `qwen2.5:3b`.
- 3B‑class models are a practical upper bound on a Pi 5 for general use; larger models run more slowly and need more thermal headroom.

You can later point OpenClaw at the local Ollama endpoint to run in a hybrid cloud/local configuration.

## OpenClaw installation and configuration

### Install Node.js 22 via nvm

- Install `nvm`, source your shell profile, then install and set Node.js 22 as the default.

### Install build tools and OpenClaw

- Install build dependencies with `apt` (build‑essential, Python, `libopus-dev`, etc.).
- Install `openclaw@latest` globally using `CFLAGS`/`CXXFLAGS` to relax specific C compiler warnings so audio modules build cleanly on ARM64.

#### Alternative: one‑line OpenClaw installer

If you just want a working OpenClaw install and are happy to let a script handle most setup, you can use the one‑line installer script; the rest of this README describes the manual, more controllable path.

### Manual agent and system configuration

- Create `~/.openclaw/agents/main/agent/openclaw.json` with your agent ID, model (`gemini-2.5-flash`), system prompt, and Gemini API configuration.
- Create `~/.openclaw/openclaw.json` with a local gateway configuration (for example `localhost:18789`).
- Keep API keys, bot tokens, and user IDs out of version control.

### Run OpenClaw onboarding (daemon mode)

- Run `openclaw onboard --install-daemon`.
- Choose manual onboarding, select Google Gemini and Telegram, and keep the gateway bound to loopback.
- After onboarding, the gateway should be running as a system daemon and listening on `localhost:18789`.

### Health checks

- Use `openclaw status` to confirm the gateway and agents are healthy.
- Use `sudo systemctl status openclaw-gateway` and `journalctl -u openclaw-gateway` for deeper inspection.

## Telegram bot and secure pairing

### Create a Telegram bot

- In Telegram, talk to **@BotFather** and use `/newbot` to create a bot; copy the bot token.
- Configure the Telegram channel in OpenClaw with your bot token and your Telegram user ID, and list that ID under `admins` and `allowedUsers`.

Never commit your bot token, user IDs, or API keys to git.

### Pair your account

- Open your bot in Telegram, hit Start and take note of the pairing code (for example `A1B2-C3D4`).
- On the Pi run `openclaw pairing approve telegram`, then say "Hi" to your bot to confirm that the agent responds.

## Modes: cloud Gemini vs local Ollama

| Mode               | Where it runs      | Pros                                           | Cons                                         |
|--------------------|--------------------|-----------------------------------------------|----------------------------------------------|
| Gemini 2.5 Flash   | Cloud (Google)     | Fast, strong reasoning, low on‑Pi load        | Needs internet, API usage costs              |
| Ollama 3B model    | On the Pi (NVMe)   | Private, can run offline, no API fees         | Slower, higher CPU and RAM usage             |

For light, cloud‑only use, a good microSD card keeps hardware cost low. For local models, persistent memory, or heavy logging, NVMe makes the system far more responsive and durable.

## Running in production

### Security notes

- Run OpenClaw on a dedicated Pi or at least a dedicated Unix user.
- Keep the gateway bound to `127.0.0.1`; do not expose port 18789 directly to LAN or internet.
- If you need remote access, use a private tunnel (Tailscale, SSH port‑forwarding) rather than opening ports.
- Restrict channels via `admins` / `allowedUsers` and avoid adding the bot to public groups.
- Avoid logging secrets; rotate API keys and tokens if you suspect exposure.

### Operations, updates and backups

- Restart OpenClaw services with `systemctl` and inspect logs with `journalctl -u openclaw-gateway`.
- Use `rpi-clone` periodically to snapshot the NVMe to another disk, not just for the initial SD→NVMe migration.
- Update OpenClaw with `npm update -g openclaw`; you may need to reuse the same `CFLAGS`/`CXXFLAGS` approach if native modules rebuild.

### Troubleshooting (quick hints)

- OpenClaw install fails on `@discordjs/opus` or audio libs: re‑run `npm install` with the `CFLAGS`/`CXXFLAGS` snippet and confirm `libopus-dev` is installed.
- Gateway service not running: check `openclaw status` and `journalctl -u openclaw-gateway`.
- Local model slow or timing out: use a smaller Ollama model, reduce concurrent sessions, or move heavy tool‑use flows back to Gemini.

## Roadmap / extensions

- Add more channels (Discord, Slack, WhatsApp) via OpenClaw's multi‑channel support.
- Add tools for GPIO, cameras and sensors to turn the Pi into an embodied home or lab assistant.
- Explore hybrid policies where local Ollama handles private queries and Gemini handles heavy reasoning or tool‑rich workflows.

## License

MIT License - See [LICENSE](LICENSE) file for details.
