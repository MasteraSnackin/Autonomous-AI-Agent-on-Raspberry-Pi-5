# Autonomous AI Agent on Raspberry Pi 5 (OpenClaw + Gemini / Ollama)

[![Node.js Version](https://img.shields.io/badge/node-%3E%3D22.0-brightgreen.svg)](https://nodejs.org/)
[![Raspberry Pi](https://img.shields.io/badge/Raspberry%20Pi-5%20(8GB)-C51A4A.svg?logo=raspberry-pi)](https://www.raspberrypi.com/products/raspberry-pi-5/)
[![OS](https://img.shields.io/badge/OS-Raspberry%20Pi%20OS%20Lite%2064--bit-blue.svg)](https://www.raspberrypi.com/software/)
[![OpenClaw](https://img.shields.io/badge/OpenClaw-latest-orange.svg)](https://docs.openclaw.ai/start/getting-started)
[![License](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

This project turns a Raspberry Pi 5 (8 GB) into a production‑grade autonomous AI agent using **OpenClaw**, with Google Gemini 2.5 Flash as the primary model and optional local models via Ollama. It supports both microSD‑only and NVMe‑accelerated setups (NVMe is optional but strongly recommended for 24/7 or local‑model use).

## Table of contents

- [Overview](#overview)
- [Assumptions](#assumptions)
- [Features](#features)
- [Hardware and prerequisites](#hardware-and-prerequisites)
- [Quick start](#quick-start)
- [System setup and performance tuning](#system-setup-and-performance-tuning)
- [Optional: storage migration SD → NVMe](#optional-storage-migration-sd--nvme)
- [System optimisation: 8 GB swap](#system-optimisation-8-gb-swap)
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

### Install Raspberry Pi OS and first boot

Write Raspberry Pi OS Lite (64‑bit) to the microSD card using Raspberry Pi Imager, set the hostname (for example `ai-agent`), enable SSH, and create your user.

SSH into the Pi:

```bash
ssh pi@ai-agent.local

On first login, fully update the OS and reboot:
sudo apt update && sudo apt full-upgrade -y
sudo reboot

Aggressive thermal management
Lower the fan thresholds to prevent heat soaking under sustained AI workloads.

Open the boot configuration file:
sudo nano /boot/firmware/config.txt
Paste these lines at the very bottom:
# Aggressive Cooling Curve for AI Workloads
dtparam=fan_temp0=40000
dtparam=fan_temp0_hyst=5000
dtparam=fan_temp0_speed=75

dtparam=fan_temp1=55000
dtparam=fan_temp1_hyst=5000
dtparam=fan_temp1_speed=150

dtparam=fan_temp2=65000
dtparam=fan_temp2_hyst=5000
dtparam=fan_temp2_speed=255
Save (Ctrl+O, Enter) and exit (Ctrl+X), then reboot:
sudo reboot

Optional: storage migration SD → NVMe
Objective: copy the OS from the slow SD card to the fast NVMe SSD using rpi-clone, then boot directly from NVMe.

Install Git and rpi-clone

sudo apt install git -y
git clone https://github.com/geerlingguy/rpi-clone.git
cd rpi-clone
sudo cp rpi-clone rpi-clone-setup /usr/local/sbin

Execute the clone
Identify your NVMe drive (usually nvme0n1):
lsblk
Run the clone command:
sudo rpi-clone nvme0n1
Prompts:

Initialise destination? type yes.

Label partitions? press Enter (leave blank).

Unmount when finished? type yes.

Configure NVMe boot order
sudo raspi-config

Navigate to:

6 Advanced Options

A4 Boot Order

B2 NVMe/USB Boot

Select Yes, then Finish.

Shut down the Pi:
sudo shutdown -h now
sudo shutdown -h now
Crucial step: physically remove the microSD card and turn the power back on.

Enable PCIe Gen 3 (post‑boot optimisation)
After booting from NVMe:
sudo nano /boot/firmware/config.txt
dtparam=pciex1
dtparam=pciex1_gen=3

Reboot:

bash
sudo reboot
System optimisation: 8 GB swap
Objective: create virtual memory to prevent crashes when loading or running AI models.

Create an 8 GB empty file on NVMe:

bash
sudo fallocate -l 8G /swapfile
Set secure permissions:

bash
sudo chmod 600 /swapfile
Format and activate swap:

bash
sudo mkswap /swapfile
sudo swapon /swapfile
Make it permanent:

bash
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
Verify swap is active (should show roughly 8.0 GiB):

bash
free -h
Optional: local models with Ollama
Objective: although this guide focuses on the cloud model Gemini 2.5 Flash, installing Ollama now lets you switch to local models later.

Install Ollama
bash
curl -fsSL https://ollama.com/install.sh | sh
Pull AI models
bash
ollama pull qwen2.5:3b
3B‑class models are a practical upper bound on a Pi 5 for general use; larger models run more slowly and need more thermal headroom.

You can later point OpenClaw at the local Ollama endpoint to run in a hybrid cloud/local configuration.

OpenClaw installation and configuration
Install Node.js 22 via nvm
bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install 22
nvm use 22
nvm alias default 22
Install build tools and OpenClaw (compiler flags)
OpenClaw relies on native audio processing that can fail to compile on modern Pi systems, so inject C compiler flags before install.

Ensure OS‑level build dependencies exist:

bash
sudo apt-get update
sudo apt-get install -y build-essential python3 make gcc g++ libopus-dev
Inject compiler flags and install OpenClaw globally:

bash
export CFLAGS="-Wno-error=implicit-function-declaration -Wno-implicit-function-declaration"
export CXXFLAGS="-Wno-error=implicit-function-declaration -Wno-implicit-function-declaration"

npm install -g openclaw@latest
This may take a few minutes as node-gyp compiles @discordjs/opus and other native modules.

Manual agent and system configuration (Gemini 2.5 Flash)
Create the agent directory:

bash
mkdir -p ~/.openclaw/agents/main/agent
Create the agent config (replace placeholders):

bash
cat > ~/.openclaw/agents/main/agent/openclaw.json <<EOF
{
  "id": "main",
  "name": "Clawbot",
  "description": "Gemini Powered Assistant",
  "model": "gemini-2.5-flash",
  "systemPrompt": "You are a helpful assistant running on a Raspberry Pi. Be concise.",
  "llm": {
    "provider": "google",
    "apiKey": "YOUR_API_KEY_HERE",
    "model": "gemini-2.5-flash"
  },
  "embedding": {
    "provider": "google",
    "apiKey": "YOUR_API_KEY_HERE",
    "model": "text-embedding-004"
  },
  "admins": ["YOUR_TELEGRAM_ID"],
  "allowedUsers": ["YOUR_TELEGRAM_ID"]
}
EOF
Then create the global system configuration:

bash
cat > ~/.openclaw/openclaw.json <<EOF
{
  "gateway": {
    "mode": "local",
    "port": 18789
  }
}
EOF
Keep API keys, bot tokens, and user IDs out of version control.

Run OpenClaw onboarding (daemon mode)
Install the gateway as a system daemon and complete manual onboarding:

bash
openclaw onboard --install-daemon
In the onboarding TUI:

Security warning: select Yes.

Onboarding mode: select Manual Mode.

Gateway config & workspace: press Enter to accept defaults.

Model/Auth provider & clients/channels: select Google Gemini and Telegram.

Runtime: select Node as the gateway runtime and let it install the gateway.

Finish: you can add skills now or skip for a minimal setup.

After onboarding, the gateway should be running as a system daemon and listening on localhost:18789.

Health checks
Check OpenClaw status:

bash
openclaw status
Inspect the gateway service:

bash
sudo systemctl status openclaw-gateway
journalctl -u openclaw-gateway
Telegram bot and secure pairing
Create a Telegram bot
In Telegram, talk to @BotFather and use /newbot to create a bot; copy the bot token.

Configure the Telegram channel in OpenClaw with your bot token and your Telegram user ID, and list that ID under admins and allowedUsers.

Never commit your bot token, user IDs, or API keys to git.

Secure pairing
By default, OpenClaw will not talk to strangers; you must pair your Telegram account.

Open your bot inside the Telegram app and click Start; the bot will reply with a pairing code (for example A1B2-C3D4).

SSH into the Pi and run:

bash
openclaw pairing approve telegram <YOUR_PAIRING_CODE_HERE>
Go back to Telegram and type “Hi” to start chatting with your AI agent.

Modes: cloud Gemini vs local Ollama
Mode	Where it runs	Pros	Cons
Gemini 2.5 Flash	Cloud (Google)	Fast, strong reasoning, low on‑Pi load	Needs internet, API usage costs
Ollama 3B model	On the Pi (NVMe)	Private, can run offline, no API fees	Slower, higher CPU and RAM usage
For light, cloud‑only use, a good microSD card keeps hardware cost low; for local models, persistent memory, or heavy logging, NVMe makes the system far more responsive and durable.

Running in production
Security notes
Run OpenClaw on a dedicated Pi or at least a dedicated Unix user.

Keep the gateway bound to 127.0.0.1; do not expose port 18789 directly to LAN or internet.

For remote access, prefer private tunnels (Tailscale, SSH port forwarding) rather than opening ports.

Restrict channels via admins / allowedUsers and avoid adding the bot to public groups.

Avoid logging secrets; rotate API keys and tokens if you suspect exposure.

Operations, updates and backups
Restart OpenClaw services and inspect logs:

bash
sudo systemctl restart openclaw-gateway
journalctl -u openclaw-gateway
Use rpi-clone periodically to snapshot the NVMe to another disk, not just for the initial SD→NVMe migration.

Update OpenClaw:

bash
npm update -g openclaw
If native modules rebuild, reuse the same CFLAGS/CXXFLAGS approach as during installation.

Troubleshooting (quick hints)
Install fails on @discordjs/opus or audio libs: re‑run npm install with the CFLAGS/CXXFLAGS snippet and confirm libopus-dev is installed.

Gateway service not running: check openclaw status and journalctl -u openclaw-gateway.

Local model slow or timing out: use a smaller Ollama model, reduce concurrent sessions, or move heavy tool‑use flows back to Gemini.

Roadmap / extensions
Add more channels (Discord, Slack, WhatsApp) via OpenClaw’s multi‑channel support.

Add tools for GPIO, cameras and sensors to turn the Pi into an embodied home or lab assistant.

Explore hybrid policies where local Ollama handles private queries and Gemini handles heavy reasoning or tool‑rich workflows.

License
MIT License – see the LICENSE file for details.

text

***

To “download” it, save that as `README.md` locally (or in your repo) and you are done.








