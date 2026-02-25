# Autonomous AI Agent on Raspberry Pi 5 (OpenClaw + Gemini / Ollama)

This project turns a Raspberry Pi 5 (8 GB) into a production‑grade autonomous AI agent using **OpenClaw** with Google Gemini 2.5 Flash and optional local models via Ollama. It supports both microSD‑only and NVMe‑accelerated setups; NVMe is optional but recommended for 24/7 or local‑model use.[web:41][web:31]

> **Assumptions**
> - Single‑user or small private setup (Telegram direct messages only, no public groups).
> - Raspberry Pi 5 (8 GB) with active cooling, running 24/7 in a safe location.
> - You are comfortable with SSH and basic Linux administration.
>
> For multi‑tenant or internet‑exposed deployments, add extra isolation (Docker/VM, firewall, read‑only filesystems) beyond this guide.

---

## 1. Features and goals

- OpenClaw autonomous agent framework with Telegram as the primary control channel.
- Cloud mode: Gemini 2.5 Flash for low‑latency reasoning and tools. 
- Optional local mode: Ollama (e.g. `qwen2.5:3b`) running directly on the Pi.  
- Optional NVMe‑booted Raspberry Pi OS for fast I/O and reduced SD wear (microSD‑only setups still work).  
- Aggressive active cooling curve and 8 GB NVMe‑backed swap for sustained workloads on 8 GB RAM.  
- Manual, transparent setup steps, with an optional one‑line OpenClaw installer for a faster path.

---

## 2. Hardware and prerequisites

### 2.1 Tested hardware

- Raspberry Pi 5, 8 GB RAM.
- Official Raspberry Pi Active Cooler (4‑pin fan header).  
- 32 GB (or larger) microSD card – **required** and sufficient if you are just experimenting or running light cloud‑only agents.
- NVMe HAT for Pi 5 plus NVMe SSD (e.g. 256 GB+) – **optional but strongly recommended** for 24/7 agents, heavier logging, or local models.
- Official 27 W Raspberry Pi USB‑C power supply or equivalent; stable power is critical under AI load.

### 2.2 Software baseline

- Raspberry Pi OS Lite (64‑bit, Bookworm or newer).
- SSH access from your workstation.  
- Google Gemini API key and a Telegram account (for bot admin).

---

## 3. Quick start (high‑level)

1. Install Raspberry Pi OS Lite (64‑bit) on SD, enable SSH, boot, and update the system.  
2. Configure an aggressive fan curve in `config.txt` for AI workloads.  
3. **Optional but recommended:** Clone the SD to an NVMe SSD with `rpi-clone`, set NVMe boot, and enable PCIe Gen 3 on the Pi 5.  
4. Create an 8 GB swap file on the NVMe (or SD, with caveats) to avoid OOM during heavy inference.
5. Install Node.js 22 via `nvm` and build prerequisites.  
6. Install OpenClaw globally with targeted compiler flags so audio dependencies compile cleanly on ARM64.  
7. Manually create the OpenClaw agent config for Gemini + Telegram, then run the onboarding wizard in daemon mode.
8. Pair your Telegram account and start chatting with your Pi‑hosted agent.

> If you do not have NVMe hardware, you can skip the NVMe section and run entirely from a good A2‑class microSD card; expect slower I/O and more patience during installs and logging‑heavy workloads.

---

## 4. System setup and performance tuning

### 4.1 Install Raspberry Pi OS (temporary SD boot)

On your PC:

- Use **Raspberry Pi Imager**.  
- OS: `Raspberry Pi OS Lite (64‑bit)`.  
- Set hostname (e.g. `ai-agent`), enable SSH, and create your user.

Boot the Pi and SSH in:

```bash
ssh pi@ai-agent.local
sudo apt update && sudo apt full-upgrade -y
sudo reboot

4.2 Aggressive thermal management
Edit the firmware boot config:

sudo nano /boot/firmware/config.txt

Append:
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

Save and reboot. These values favour sustained performance over noise and are appropriate for enclosed cases and long AI runs.

4.3 (Optional) Migrate from SD to NVMe
Optional but recommended
You can run this entire stack from a fast A2‑class microSD card. However, an NVMe SSD will give you much faster installs, log writes, and model downloads, and is generally more durable under constant writes on a Pi 5.

Install rpi-clone and clone the SD to NVMe:

sudo apt install -y git
git clone https://github.com/geerlingguy/rpi-clone.git
cd rpi-clone
sudo cp rpi-clone rpi-clone-setup /usr/local/sbin

lsblk  # identify NVMe (usually nvme0n1)
sudo rpi-clone nvme0n1
Answer the prompts:

Initialise destination: yes

Label partitions: press Enter

Unmount when finished: yes

Set boot order:

bash
sudo raspi-config
# 6 Advanced Options -> A4 Boot Order -> B2 NVMe/USB Boot
Shut down, physically remove the microSD card, and power the Pi back on.

Enable PCIe Gen 3 for maximum throughput:[web:7][web:47]

bash
sudo nano /boot/firmware/config.txt
Append:

text
dtparam=pciex1
dtparam=pciex1_gen=3
Reboot:

bash
sudo reboot
4.4 Create 8 GB swap (NVMe preferred)
Swap location guidance
On NVMe, 8 GB swap is safe and fast for this workload. On microSD it will be slower and will wear the card more quickly, so consider reducing swap size or moving to NVMe for long‑term or 24/7 use.[web:7][web:41]

Create a standard Linux swap file on the root filesystem (NVMe, or SD if you are not using NVMe):

bash
sudo fallocate -l 8G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
free -h
You should now see roughly 8 GiB of swap available. This gives the OpenClaw gateway and Node processes headroom when memory usage spikes.[web:34]

5. Optional: local models with Ollama
This is optional but lets you run fully local models when you do not want to use cloud APIs.[web:8][web:11]

bash
curl -fsSL https://ollama.com/install.sh | sh
ollama pull qwen2.5:3b
On a Pi 5, 3B‑class models are a practical upper bound for general use; larger models will run noticeably slower and demand more thermal headroom.[web:62][web:60] You can later point OpenClaw at this local Ollama endpoint for a hybrid cloud/local setup.

6. OpenClaw installation and configuration
6.1 Install Node.js 22 via nvm
bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install 22
nvm use 22
nvm alias default 22
6.2 Install build tools and OpenClaw (with compilation fix)
Install build dependencies:

bash
sudo apt-get update
sudo apt-get install -y build-essential python3 make gcc g++ libopus-dev
Install OpenClaw globally, relaxing specific C compiler warnings just for this command so native audio modules build cleanly on ARM64:[web:3][web:34]

bash
CFLAGS="-Wno-error=implicit-function-declaration -Wno-implicit-function-declaration" \
CXXFLAGS="-Wno-error=implicit-function-declaration -Wno-implicit-function-declaration" \
npm install -g openclaw@latest
This scopes the flags to the install command only and avoids weakening your compiler globally.

Alternative: one‑line OpenClaw installer
If you just want a working OpenClaw install and are happy to let a script handle most setup, there is also a one‑line installer:

bash
curl -sSL https://get.moltbot.org/install-pi.sh | bash
This script sets up OpenClaw, dependencies and systemd services for you.[web:53][web:27] The rest of this guide shows the manual, more controllable path.

6.3 Manual agent and system configuration
You will bypass the interactive onboarding initially and seed a known‑good Gemini + Telegram config. You will plug in your own API keys and IDs; do not commit these files to git.[web:9][web:6]

Create the agent directory:

bash
mkdir -p ~/.openclaw/agents/main/agent
Create ~/.openclaw/agents/main/agent/openclaw.json:

bash
cat > ~/.openclaw/agents/main/agent/openclaw.json <<'EOF'
{
  "id": "main",
  "name": "Clawbot",
  "description": "Gemini Powered Assistant",
  "model": "gemini-2.5-flash",
  "systemPrompt": "You are a helpful assistant running on a Raspberry Pi. Be concise.",
  "llm": {
    "provider": "google",
    "apiKey": "<GEMINI_API_KEY>",
    "model": "gemini-2.5-flash"
  },
  "embedding": {
    "provider": "google",
    "apiKey": "<GEMINI_API_KEY>",
    "model": "text-embedding-004"
  },
  "admins": ["<YOUR_TELEGRAM_USER_ID>"],
  "allowedUsers": ["<YOUR_TELEGRAM_USER_ID>"]
}
EOF
Note: Google are rolling out newer Gemini embedding models; text-embedding-004 remains valid in many setups but check current model IDs and deprecation timelines and adjust accordingly.[web:22][web:28]

Create the global system config ~/.openclaw/openclaw.json:

bash
cat > ~/.openclaw/openclaw.json <<'EOF'
{
  "gateway": {
    "mode": "local",
    "port": 18789
  }
}
EOF
The default gateway bind is loopback‑only, which is what you want for a safe local deployment.[web:58][web:67]

6.4 Run OpenClaw onboarding (daemon mode)
Now run the onboarding wizard and install OpenClaw as a system daemon:[web:9][web:27]

bash
openclaw onboard --install-daemon
When prompted:

Security warning: choose Yes to install as a daemon.

Onboarding mode: select Manual mode (you already seeded the config).

Gateway config & workspace: accept defaults.

Model/auth provider & clients/channels: select Google Gemini and Telegram.

Gateway runtime: choose Node when asked and proceed with gateway install.

After onboarding completes, you should see the gateway connected in the TUI. At this point your Pi agent is running as a background service.[web:24][web:52]

6.5 Health checks
Check OpenClaw status:

bash
openclaw status
If a systemd service was created, you can also inspect it:

bash
sudo systemctl status openclaw-gateway
The gateway should be listening on localhost:18789 as configured.[web:58][web:55]

7. Telegram bot and secure pairing
7.1 Create a Telegram bot
In Telegram, start a chat with @BotFather.[web:6][web:15]

Use /newbot to create a bot and follow the prompts.

Copy the bot token and add it in your OpenClaw Telegram channel configuration (via TUI/web UI or config files).[web:53][web:65]

Find your Telegram user ID (for example using a “user ID” bot) and ensure it is listed under admins and allowedUsers in your agent config.[web:6][web:9]

Never commit your bot token, user IDs, or API keys to git.

7.2 Pair your account
Open your bot in the Telegram app and click Start. The bot will respond with a pairing code such as A1B2-C3D4.[web:53]

On the Pi:

bash
openclaw pairing approve telegram <YOUR_PAIRING_CODE>
Return to Telegram and say “Hi” to your bot. You should now have a fully autonomous Gemini‑powered agent running on your Raspberry Pi.[web:52][web:24]

8. Modes: cloud Gemini vs local Ollama
This setup supports two primary inference modes:

Mode	Where it runs	Pros	Cons
Gemini 2.5 Flash	Cloud (Google)	Fast, strong reasoning, low on‑Pi load[web:10][web:6]	Needs internet, API usage costs
Ollama 3B model	On the Pi (NVMe)	Private, can run offline, no API fees[web:8][web:11]	Slower, heavier CPU and RAM usage
For light, cloud‑only use (Gemini only, no heavy local models), a good microSD card is acceptable and keeps hardware cost low.[web:41][web:27] For local models, persistent memory, or heavy logging, an NVMe SSD is the preferred storage backend on the Pi 5 and makes the system feel much more responsive.[web:7][web:47]

OpenClaw’s security guidance recommends using a modern, strong model for any agent that can run tools or touch files and networks; Gemini 2.5 Flash fits that role well.[web:58][web:25] If you use smaller or local models, limit tool access and sandbox the filesystem accordingly.[web:58][web:67]

By default this guide configures Gemini for best responsiveness, while the NVMe + swap + cooling optimisations ensure the Pi can also handle modest local models if you choose to configure OpenClaw with Ollama later.[web:7][web:10]

9. Running in production
9.1 Security notes
Run OpenClaw on a dedicated Pi or at least a dedicated Unix user, not on a machine that holds unrelated secrets.

Keep the gateway bound to loopback (127.0.0.1), which is the default; do not expose port 18789 directly to LAN or the internet.

If you need remote access, use a private tunnel (e.g. Tailscale, SSH port‑forwarding) rather than opening ports.

Restrict channels with admins / allowedUsers and avoid adding the bot to public groups.

Avoid logging secrets; rotate API keys and Telegram tokens if you ever suspect exposure.

9.2 Operations, updates and backups
24/7 running

Restart services: sudo systemctl restart openclaw-gateway (or the service name created on your system).

Logs: use journalctl -u openclaw-gateway and OpenClaw’s own log directory.

Backups

Use rpi-clone periodically to snapshot the NVMe to another disk, not just for the initial SD→NVMe migration.

Updating OpenClaw

npm update -g openclaw to pull new releases.

Major updates may re‑trigger native compilation; if that happens, reuse the same CFLAGS/CXXFLAGS technique used for first install.

9.3 Troubleshooting (quick hints)
OpenClaw install fails on @discordjs/opus or audio libs

Re‑run the npm install with the CFLAGS/CXXFLAGS snippet and check that libopus-dev is installed.

Gateway service not running

Run openclaw status and inspect logs with journalctl -u openclaw-gateway.[web:27][web:52]

Local model is very slow or times out

Try a smaller Ollama model, reduce concurrent sessions, or switch heavy tool‑use flows back to Gemini.

10. Roadmap / extensions
Add more channels (Discord, Slack, WhatsApp) via OpenClaw’s multi‑channel support.

Add tools for local hardware control (GPIO, cameras, sensors) to turn the Pi into a fully embodied home or lab assistant.

Explore hybrid policies where local Ollama handles private queries and Gemini handles heavy reasoning or tool‑rich workflows.

text

