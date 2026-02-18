---
date: 2026-02-17
type: build-guide
tags:
  - quinn-session
  - voice-assistant
  - home-assistant
  - raspberry-pi
  - esp32
  - hardware
---

# Hub-and-Spoke Voice Assistant Build Guide

> A step-by-step guide to building a voice assistant with a Raspberry Pi 4 hub and ESP32 satellite speakers. Uses Home Assistant as the platform. Written for someone who's used a terminal a few times but isn't a Linux expert.

---

## Table of Contents

1. [What You're Building](#what-youre-building)
2. [Parts List](#parts-list)
3. [Phase 1: Hub Setup (Pi 4 + Home Assistant)](#phase-1-hub-setup)
4. [Phase 2: Quick-Start Spoke (ATOM Echo, $13)](#phase-2-quick-start-spoke)
5. [Phase 3: Voice Pipeline Configuration](#phase-3-voice-pipeline)
6. [Phase 4: LLM Integration (Ollama)](#phase-4-llm-integration)
7. [Phase 5: DIY Spoke Build (XIAO ESP32S3)](#phase-5-diy-spoke)
8. [Phase 6: WiFi & Network Tuning](#phase-6-wifi-tuning)
9. [Troubleshooting](#troubleshooting)
10. [Maintenance & Updates](#maintenance-updates)

---

## What You're Building

```
                    ┌─────────────────────┐
                    │   RASPBERRY PI 4    │
                    │   (Home Assistant)  │
                    │                     │
                    │  Wake Word Engine   │
                    │  Speech-to-Text     │
                    │  Text-to-Speech     │
                    │  LLM (Ollama)       │
                    │  Smart Home Control │
                    └──────────┬──────────┘
                               │ WiFi
                ┌──────┬───────┼───────┬──────┐
                │      │       │       │      │
             [Spoke] [Spoke] [Spoke] [Spoke]
              Mic     Mic     Mic     Mic
             Speaker Speaker Speaker Speaker
```

**How it works:**
1. You say a wake word ("Hey Jarvis", "OK Nabu", etc.)
2. The spoke streams your voice over WiFi to the Pi hub
3. The hub converts speech to text (STT)
4. The hub sends the text to an AI (Ollama or OpenAI)
5. The hub converts the AI response to speech (TTS)
6. The spoke plays the audio through its speaker

**Latency:** ~800ms from when you stop talking to when it starts responding (with cloud STT). About 2-3 seconds with local-only STT.

---

## Parts List

### Hub (Raspberry Pi 4)

| # | Item | Product | Price | Where to Buy |
|---|------|---------|-------|-------------|
| 1 | Computer | Raspberry Pi 4 Model B, **4GB RAM** | ~$60 | raspberrypi.com, CanaKit, Adafruit |
| 2 | Storage | Samsung EVO Select 64GB microSD (A2 rated) | ~$10 | Amazon |
| 3 | Case | Argon ONE V2 (aluminum, built-in fan) | ~$28 | Amazon |
| 4 | Power (PoE) | UCTRONICS USB-C PoE Splitter 5V/4A | ~$22 | Amazon (B07V35DH5F) |
| 5 | Power (PoE) | 802.3at PoE+ Injector | ~$20 | Amazon |
| 6 | Network | Cat6 Ethernet cable (length you need) | ~$7 | Amazon |
| 7 | Microphone (optional) | Seeed ReSpeaker Mic Array v2.0 | ~$70 | Amazon (B07D29L3Q1) |

**Hub total: ~$147** (without hub mic) or **~$217** (with hub mic)

> **Note:** The hub microphone is optional. If you only use satellite spokes for voice input, you don't need a mic on the hub itself. Only add it if you want voice control in the same room as the Pi.

> **Alternative power:** If you don't need PoE, any USB-C power supply that does 5V/3A works. The official Raspberry Pi USB-C power supply is $12.

### Quick-Start Spokes (no soldering)

| # | Item | Price | Notes |
|---|------|-------|-------|
| 1 | M5Stack ATOM Echo | ~$13 | Tiny, has mic + speaker, USB-C. Easiest to set up. |
| 2 | ESP32-S3-BOX-3 | ~$45 | Better speaker, touchscreen, better mics |
| 3 | HA Voice Preview Edition | ~$59 | Best audio, 3.5mm out, hardware mute, LED ring |

### DIY Spokes (battery + 3.5mm, requires soldering)

| # | Item | Product | Price | Where to Buy |
|---|------|---------|-------|-------------|
| 1 | Microcontroller | Seeed XIAO ESP32S3 | ~$7.50 | seeedstudio.com |
| 2 | Microphone | INMP441 I2S MEMS Mic Breakout | ~$4 | Amazon, AliExpress |
| 3 | Amplifier | MAX98357A I2S Amp Breakout | ~$5 | Amazon, Adafruit |
| 4 | Speaker | 40mm full-range, 4 ohm, 3W | ~$4 | Amazon, AliExpress |
| 5 | Battery | 3.7V 2000mAh LiPo, JST-PH connector | ~$8 | Amazon, Adafruit |
| 6 | Audio Jack | PJ-320A 3.5mm stereo jack | ~$1 | Amazon, AliExpress |
| 7 | Enclosure | 3D printed or project box | ~$3-5 | |
| 8 | Misc | Hookup wire, solder, headers | ~$2 | |

**DIY spoke total: ~$35/unit**

### Tools You'll Need

- A computer (Mac, Windows, or Linux) with Chrome or Edge browser
- A microSD card reader (often built into laptops)
- For DIY spokes: soldering iron, solder, wire strippers, multimeter (helpful)

---

## Phase 1: Hub Setup

### Step 1.1: Download Raspberry Pi Imager

1. Go to **https://www.raspberrypi.com/software/**
2. Click **Download for** your operating system (macOS, Windows, or Linux)
3. Install it like any normal app

### Step 1.2: Flash Home Assistant OS to the SD Card

1. Insert your 64GB microSD card into your computer
2. Open **Raspberry Pi Imager**
3. Click **CHOOSE DEVICE** → Select **Raspberry Pi 4**
4. Click **CHOOSE OS** → Scroll down → Click **Other specific-purpose OS** → **Home assistants and home automation** → **Home Assistant** → **Home Assistant OS 14.x (RPi 4/400)**
5. Click **CHOOSE STORAGE** → Select your microSD card
6. Click **NEXT**
7. When asked about OS customisation, click **NO** (Home Assistant handles its own setup)
8. Click **YES** to confirm writing
9. Wait for it to finish writing and verifying (~5-10 minutes)
10. Remove the SD card

### Step 1.3: Assemble the Hub

1. **If using the Argon ONE V2 case:**
   - Follow the case instructions to mount the Pi 4 board into the case
   - Connect the fan header to GPIO pins (instructions included with case)
   - Close the case

2. **Insert the microSD card** into the Pi 4's card slot (bottom of the board, push gently until it clicks)

3. **Connect Ethernet:** Plug one end of the Cat6 cable into the Pi's Ethernet port, the other end into your PoE injector's "Data + Power Out" port. Connect the injector's "Data In" port to your router/switch.

4. **Connect PoE splitter:** Plug the PoE splitter into the Ethernet cable at the Pi end. It splits into Ethernet (plug into Pi) + USB-C power (plug into Pi's USB-C power port).

   > **If not using PoE:** Just plug a USB-C power supply directly into the Pi.

5. **If using the ReSpeaker mic:** Plug the USB cable from the ReSpeaker into one of the Pi's USB ports.

6. **Do NOT connect a monitor or keyboard.** Everything is done through the web browser from your other computer.

### Step 1.4: Boot and Initial Setup

1. Power will flow through the PoE injector (or plug in your USB-C supply). The Pi will boot automatically.

2. **Wait 5-20 minutes.** The first boot takes a while — Home Assistant is installing itself. Don't unplug it.

3. On your computer (connected to the same network), open a web browser and go to:
   ```
   http://homeassistant.local:8123
   ```
   > **If that doesn't work**, try finding the Pi's IP address in your router's admin page, then go to `http://[IP_ADDRESS]:8123`

4. You'll see a "Preparing Home Assistant" screen. This can take up to 20 minutes on first boot. Be patient.

5. Once ready, you'll see the **Welcome** screen:
   - Create your user account (pick a name and password — **remember these!**)
   - Set your home location (for weather, sunrise/sunset automations)
   - Choose which info to share (your choice, all optional)
   - Click **Finish**

6. You're now in the Home Assistant dashboard. Bookmark this page.

---

## Phase 2: Quick-Start Spoke

> Start with one ATOM Echo ($13) to verify everything works before building DIY spokes.

### Step 2.1: Flash the ATOM Echo

1. Plug the ATOM Echo into your computer via USB-C
2. Open **Chrome** (must be Chrome or Edge — Firefox won't work)
3. Go to: **https://www.home-assistant.io/voice_control/thirteen-usd-voice-remote/**
4. Scroll down to the "Connect" button and click it
5. A popup appears — select the serial port for the ATOM Echo (usually something like "USB Serial" or "CP2104")
6. Click **Install Voice Assistant**
7. Wait for the firmware to flash (~2 minutes)
8. When prompted, enter your **WiFi network name and password** (the same network your Pi is on)
9. The ATOM Echo will reboot and connect to your WiFi

### Step 2.2: Add the Spoke to Home Assistant

1. In Home Assistant, go to **Settings** → **Devices & Services**
2. You should see a notification: **"1 new device discovered"** — click it
3. Click **Configure** next to the ESPHome device
4. Click **Submit**, then **Finish**
5. The ATOM Echo is now connected to Home Assistant

> **If it doesn't auto-discover:** Go to Settings → Devices & Services → click **+ Add Integration** → search for **ESPHome** → enter the ATOM Echo's IP address (find it in your router's admin page)

---

## Phase 3: Voice Pipeline

### Step 3.1: Install Voice Add-ons

In Home Assistant:

1. Go to **Settings** → **Add-ons** → Click **Add-on Store** (bottom right)

2. **Install Whisper** (Speech-to-Text):
   - Search for "Whisper"
   - Click **Whisper**
   - Click **Install** → Wait for download
   - Toggle **Start on boot** → ON
   - Toggle **Watchdog** → ON
   - Click **Start**
   - Go to the **Configuration** tab:
     - Model: **tiny-int8** (fastest on Pi 4)
     - Language: **en**
   - Click **Save**

3. **Install Piper** (Text-to-Speech):
   - Back to Add-on Store → Search for "Piper"
   - Click **Piper**
   - Click **Install** → Wait
   - Toggle **Start on boot** → ON
   - Toggle **Watchdog** → ON
   - Click **Start**
   - Go to **Configuration** tab:
     - Voice: **en_US-lessac-medium** (good quality, runs smoothly on Pi 4)
   - Click **Save**

4. **Install OpenWakeWord** (Wake Word Detection):
   - Back to Add-on Store → Search for "openWakeWord"
   - Click **openWakeWord**
   - Click **Install** → Wait
   - Toggle **Start on boot** → ON
   - Toggle **Watchdog** → ON
   - Click **Start**

### Step 3.2: Create a Voice Pipeline

1. Go to **Settings** → **Voice assistants**
2. Click **+ Add Assistant**
3. Fill in:
   - **Name:** "Main Voice Assistant" (or whatever you want)
   - **Language:** English
   - **Conversation agent:** Home Assistant (default — we'll add Ollama later)
   - **Speech-to-text:** Whisper
   - **Text-to-speech:** Piper (choose the lessac-medium voice)
   - **Wake word:** openWakeWord → pick a wake word (e.g., "hey jarvis", "ok nabu", "alexa")
4. Click **Create**

### Step 3.3: Assign Pipeline to Your Spoke

1. Go to **Settings** → **Devices & Services** → **ESPHome**
2. Click on your ATOM Echo device
3. Under **Configuration**, find the **Voice Assistant Pipeline** dropdown
4. Select the pipeline you just created
5. Click **Save**

### Step 3.4: Test It!

1. Say your wake word toward the ATOM Echo
2. The LED should light up (it's listening)
3. Ask something: "What time is it?" or "Tell me a joke"
4. You should hear a response through the speaker

> **If it doesn't work**, check the [Troubleshooting](#troubleshooting) section below.

---

## Phase 4: LLM Integration

> By default, Home Assistant's "Assist" only understands smart home commands. Adding Ollama gives it a real AI brain.

### Step 4.1: Install Ollama

You have two options:

**Option A: Run Ollama on the Pi 4 itself** (works for small models only)

1. In Home Assistant, go to **Settings** → **Add-ons** → **Add-on Store**
2. Search for "Ollama" — if there's an add-on, install it
3. If no add-on exists, you'll need SSH access:
   - Install the **SSH & Web Terminal** add-on from the Add-on Store
   - Start it, then click **Open Web UI**
   - In the terminal, run:
     ```
     curl -fsSL https://ollama.com/install.sh | sh
     ```
   - Then pull a small model:
     ```
     ollama pull qwen2.5:3b
     ```
     > This downloads ~2GB. The Pi 4 can run 3B-parameter models. Anything bigger will be very slow.

**Option B: Run Ollama on a separate, more powerful computer** (recommended)

1. On your other computer (Mac, Windows, or Linux), go to **https://ollama.com/download**
2. Download and install Ollama
3. Open a terminal and run:
   ```
   ollama pull llama3.2
   ```
4. Ollama runs as a background service on port 11434
5. Make sure the computer is on the same network as the Pi

### Step 4.2: Connect Ollama to Home Assistant

1. In Home Assistant, go to **Settings** → **Devices & Services**
2. Click **+ Add Integration**
3. Search for **Ollama**
4. Enter the URL:
   - If Ollama is on the Pi: `http://localhost:11434`
   - If Ollama is on another computer: `http://[COMPUTER_IP]:11434`
5. Select your model (e.g., `llama3.2` or `qwen2.5:3b`)
6. Click **Submit**

### Step 4.3: Use Ollama in Your Voice Pipeline

1. Go to **Settings** → **Voice Assistants**
2. Click your voice assistant to edit it
3. Change **Conversation agent** from "Home Assistant" to **Ollama**
4. Click **Update**

Now when you ask questions, they go to the LLM instead of just Home Assistant's basic intent matcher.

> **Tip:** You can keep "Home Assistant" as the conversation agent for smart home commands (faster, works offline) and create a SECOND pipeline with Ollama for general questions. Assign different wake words to each.

### Step 4.4: Alternative — Use OpenAI Instead

If you'd rather use OpenAI's API (faster, smarter, but costs money):

1. Go to **Settings** → **Devices & Services** → **+ Add Integration**
2. Search for **OpenAI Conversation**
3. Enter your OpenAI API key
4. Select model (gpt-4o-mini is cheap and fast)
5. Assign it as the conversation agent in your voice pipeline

---

## Phase 5: DIY Spoke

> This section builds a custom satellite with battery power, 3.5mm audio jack, and USB-C charging. Requires soldering.

### Step 5.1: Wiring Diagram

```
                    XIAO ESP32S3
                   ┌─────────────┐
                   │             │
    INMP441 Mic    │             │    MAX98357A Amp
   ┌──────────┐    │             │   ┌──────────────┐
   │ VDD ─────┼──→ 3V3          │   │              │
   │ GND ─────┼──→ GND     D0 ──┼──→│ BCLK         │
   │ SCK ─────┼──→ D1           │   │              │
   │ WS  ─────┼──→ D2      D3 ──┼──→│ LRC          │
   │ SD  ─────┼──→ D4      D5 ──┼──→│ DIN          │
   │ L/R ─────┼──→ GND          │   │ VIN ─────────┼──→ 3V3
   └──────────┘    │             │   │ GND ─────────┼──→ GND
                   │             │   │              │
                   │    BAT+  ←──┼───│── LiPo +     │
                   │    BAT-  ←──┼───│── LiPo -     │   ┌─────────┐
                   │             │   │ OUT+ ────────┼──→│ Speaker +│
                   └─────────────┘   │ OUT- ────────┼──→│ Speaker -│
                                     │              │   └─────────┘
                                     │ OUT+ ──┐     │
                                     │ OUT- ──┼─────┼──→ 3.5mm Jack
                                     └────────┘     │   (Tip=OUT+,
                                                        Sleeve=OUT-)
```

### Step 5.2: Pin Connections Table

| INMP441 Mic Pin | Connect To | Wire Color (suggested) |
|-----------------|-----------|----------------------|
| VDD | XIAO 3V3 | Red |
| GND | XIAO GND | Black |
| SCK (Clock) | XIAO D1 (GPIO2) | Yellow |
| WS (Word Select) | XIAO D2 (GPIO3) | Green |
| SD (Data) | XIAO D4 (GPIO5) | Blue |
| L/R | XIAO GND (selects left channel) | Black |

| MAX98357A Pin | Connect To | Wire Color (suggested) |
|--------------|-----------|----------------------|
| BCLK | XIAO D0 (GPIO1) | Yellow |
| LRC | XIAO D3 (GPIO4) | Green |
| DIN | XIAO D5 (GPIO6) | Blue |
| VIN | XIAO 3V3 | Red |
| GND | XIAO GND | Black |
| OUT+ | Speaker + AND 3.5mm Tip | |
| OUT- | Speaker - AND 3.5mm Sleeve | |

| Battery | Connect To |
|---------|-----------|
| LiPo + (red wire) | XIAO BAT+ pad (on bottom of board) |
| LiPo - (black wire) | XIAO BAT- pad (on bottom of board) |

> **Important:** The XIAO ESP32S3 has built-in LiPo charging. When you plug in USB-C, it charges the battery automatically. No extra charging board needed.

### Step 5.3: Assembly Steps

1. **Solder headers to the XIAO** (if not pre-soldered)
   - Solder male pin headers to both rows of through-holes

2. **Wire the microphone (INMP441)**
   - Cut 5 pieces of hookup wire, ~3 inches each
   - Solder each wire to the INMP441 breakout board pads
   - Connect the other ends to the XIAO pins per the table above
   - Double-check: VDD to 3V3, GND to GND, then SCK/WS/SD to D1/D2/D4

3. **Wire the amplifier (MAX98357A)**
   - Cut 5 pieces of hookup wire, ~3 inches each
   - Solder to the MAX98357A breakout pads
   - Connect to XIAO pins per the table above
   - Double-check: VIN to 3V3, GND to GND, then BCLK/LRC/DIN to D0/D3/D5

4. **Wire the speaker**
   - Solder speaker + wire to MAX98357A OUT+
   - Solder speaker - wire to MAX98357A OUT-

5. **Wire the 3.5mm jack**
   - Solder a wire from MAX98357A OUT+ to the **Tip** terminal of the 3.5mm jack
   - Solder a wire from MAX98357A OUT- to the **Sleeve** terminal
   - (Ring terminal can be left unconnected for mono, or bridged to Tip for pseudo-stereo)

6. **Connect the battery**
   - Solder the LiPo's red wire to the BAT+ pad on the bottom of the XIAO
   - Solder the LiPo's black wire to the BAT- pad
   - **Be careful with polarity! Reversing + and - can destroy the board.**

7. **Test before enclosing**
   - Plug in USB-C to power on
   - The XIAO's LED should light up
   - Proceed to firmware flashing (next step)

### Step 5.4: Flash ESPHome Firmware

1. Plug the XIAO ESP32S3 into your computer via USB-C
2. In Home Assistant, go to **Settings** → **Add-ons** → Install **ESPHome** add-on
3. Start the ESPHome add-on → Click **Open Web UI**
4. Click **+ New Device** → Give it a name (e.g., "spoke-kitchen")
5. Select **ESP32-S3** as the board
6. Click **Skip** on the initial install (we'll use the YAML below)
7. Click the **Edit** button on the new device
8. Replace the YAML with this configuration:

```yaml
esphome:
  name: spoke-kitchen
  friendly_name: Kitchen Spoke
  platformio_options:
    board_build.flash_mode: dio

esp32:
  board: seeed_xiao_esp32s3
  variant: esp32s3
  framework:
    type: esp-idf
    version: recommended

# WiFi connection to your hub network
wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

# Fallback hotspot if WiFi fails
  ap:
    ssid: "Spoke-Kitchen-Fallback"
    password: "fallback123"

captive_portal:

logger:

api:
  encryption:
    key: !secret api_encryption_key

ota:
  - platform: esphome
    password: !secret ota_password

# Microphone (INMP441)
i2s_audio:
  - id: i2s_mic
    i2s_lrclk_pin: GPIO3    # WS on D2
    i2s_bclk_pin: GPIO2     # SCK on D1

  - id: i2s_spk
    i2s_lrclk_pin: GPIO4    # LRC on D3
    i2s_bclk_pin: GPIO1     # BCLK on D0

microphone:
  - platform: i2s_audio
    id: xiao_mic
    adc_type: external
    i2s_audio_id: i2s_mic
    i2s_din_pin: GPIO5      # SD on D4
    pdm: false
    bits_per_sample: 32bit
    channel: left

speaker:
  - platform: i2s_audio
    id: xiao_speaker
    i2s_audio_id: i2s_spk
    dac_type: external
    i2s_dout_pin: GPIO6     # DIN on D5
    mode: mono

# Voice Assistant configuration
voice_assistant:
  microphone: xiao_mic
  speaker: xiao_speaker
  use_wake_word: true
  noise_suppression_level: 2
  auto_gain: 31dBFS
  volume_multiplier: 2.0

# Optional: LED for status indication
light:
  - platform: esp32_rmt_led_strip
    id: status_led
    pin: GPIO21
    num_leds: 1
    rmt_channel: 0
    chipset: WS2812
    rgb_order: GRB
    name: "Status LED"
```

9. Click **Save**
10. Click **Install** → Choose **Plug into this computer** (for first flash)
11. Select the serial port for the XIAO
12. Wait for compilation and upload (~3-5 minutes)
13. Once done, the spoke will reboot and connect to your WiFi

### Step 5.5: Add DIY Spoke to Home Assistant

1. The spoke should auto-discover in Home Assistant
2. Go to **Settings** → **Devices & Services** → look for the new ESPHome device
3. Click **Configure** → **Submit** → **Finish**
4. Assign it to your voice pipeline (same as Phase 2, Step 2.3)

### Step 5.6: Enclose It

- If 3D printing: search Printables.com for "ESP32 voice assistant enclosure"
- If using a project box: drill a hole array for the speaker, a small hole for the mic, and holes for USB-C and 3.5mm jack
- Secure components with hot glue or double-sided tape
- Make sure the battery isn't pressed or punctured

---

## Phase 6: WiFi Tuning

> The Pi 4 uses its built-in WiFi for the satellite connections. This section optimizes it.

### Important Note on WiFi Architecture

**The default and recommended setup uses your existing WiFi router.** Both the Pi (via Ethernet) and the spokes (via WiFi) connect to your home network. The Pi does NOT need to run as a WiFi access point in this case.

**Only set up the Pi as a WiFi AP if:**
- You want an isolated network for voice devices
- You're deploying where there's no existing WiFi
- You want to control the RF environment precisely

If your existing WiFi works and spokes connect reliably, **skip this section entirely.**

### If You Need Pi as WiFi AP

> This requires SSH access. In Home Assistant, install the **SSH & Web Terminal** add-on, then open its web UI.

**This is an advanced configuration.** The Pi 4's built-in WiFi has limited range (~10-15m indoors). Your existing router is almost certainly better. Only proceed if you have a specific reason.

Setting up hostapd on HAOS is non-trivial because HAOS is a managed OS. The better approach for an isolated network is:

1. Keep HAOS on the Pi connected via Ethernet
2. Use a cheap dedicated WiFi AP (like a $20 TP-Link) for the spoke network
3. Connect that AP to the same Ethernet switch as the Pi

This gives you better range than the Pi's built-in WiFi and doesn't require modifying HAOS.

---

## Troubleshooting

### "I can't reach homeassistant.local:8123"

1. Make sure your computer and the Pi are on the same network
2. Wait 20 minutes — first boot is slow
3. Try finding the Pi's IP address:
   - Check your router's admin page (usually 192.168.1.1) for connected devices
   - Look for "homeassistant" or a device with a Raspberry Pi MAC address
   - Try `http://[IP_ADDRESS]:8123`

### "The ATOM Echo won't flash"

1. Make sure you're using **Chrome or Edge** (Firefox doesn't support Web Serial)
2. Try a different USB-C cable — some cables are charge-only and don't carry data
3. On Mac, you may need to allow the serial port in System Settings → Privacy & Security

### "Wake word isn't detected"

1. Check that OpenWakeWord add-on is running: **Settings** → **Add-ons** → **openWakeWord** → should say "Running"
2. Make sure you selected a wake word in the voice pipeline config
3. Try speaking louder and closer to the mic
4. Check the spoke's distance from the WiFi router — if it's far, audio may not stream reliably

### "It hears me but doesn't respond"

1. Check that Whisper add-on is running
2. Check that Piper add-on is running
3. Go to **Developer Tools** → **Logs** → look for errors from `whisper` or `piper`
4. Try a simpler question: "What time is it?"
5. Check the Whisper configuration — make sure Language is set to "en"

### "The response is very slow (>5 seconds)"

1. You're probably using local Whisper STT on the Pi. This is normal for larger models.
2. Switch to the **tiny-int8** model: Add-ons → Whisper → Configuration → Model: tiny-int8
3. Consider switching to cloud STT (HA Cloud at $6.50/mo gives the fastest experience)

### "No sound from the speaker"

1. For ATOM Echo: the built-in speaker is very quiet. Try holding it closer to your ear.
2. For DIY spoke: check your wiring — especially OUT+/OUT- from the MAX98357A to the speaker
3. For 3.5mm: make sure something is plugged into the jack and turned up
4. In HA, go to the device page and try playing a test sound

### "ESPHome won't compile the YAML"

1. Read the error message carefully — it usually tells you the line number
2. YAML is indentation-sensitive — use **spaces, not tabs**, and be consistent (2 spaces per indent)
3. Make sure you've set up ESPHome secrets: in ESPHome dashboard → click **Secrets** → add your wifi_ssid, wifi_password, api_encryption_key, and ota_password

### "DIY spoke connects to WiFi but no voice assistant"

1. In HA, check **Settings** → **Devices & Services** → **ESPHome** → your device should be listed
2. Make sure `voice_assistant:` section is in the YAML
3. Check that a voice pipeline is assigned to the device
4. Restart the device by unplugging/replugging USB-C

---

## Maintenance & Updates

### Updating Home Assistant

1. When an update is available, you'll see a notification in the sidebar
2. Go to **Settings** → **System** → **Updates**
3. Click **Update** → Wait for it to complete (can take 5-20 minutes)
4. **Do NOT unplug the Pi during an update**

> **Tip:** Read the release notes before updating. Once or twice a year, an update may change how voice pipelines work. The HA blog (https://www.home-assistant.io/blog/) announces breaking changes.

### Updating Add-ons (Whisper, Piper, etc.)

1. Go to **Settings** → **Add-ons**
2. If an add-on has an update available, you'll see an "Update" button
3. Click it. They usually update in under a minute.

### Updating Spoke Firmware

1. After the first flash via USB, all future updates happen **over WiFi (OTA)**
2. In ESPHome dashboard, click **Install** → **Wirelessly**
3. Wait for it to compile and upload

### Updating Ollama Models

If running Ollama, periodically check for model updates:

```bash
ollama pull llama3.2
```

This downloads only the differences, so it's usually fast.

### Backups

1. Go to **Settings** → **System** → **Backups**
2. Click **Create Backup**
3. Download the backup file to your computer
4. **Do this before every major update** and at least monthly

### If Something Breaks After an Update

1. **Don't panic.** Check the HA community forums — someone else probably hit the same issue.
2. Check **Settings** → **System** → **Logs** for error messages
3. Try restarting the add-on that's failing: **Settings** → **Add-ons** → click the add-on → **Restart**
4. If everything is broken: restore from your backup (**Settings** → **System** → **Backups** → select backup → **Restore**)

---

## Cloud STT Setup (Optional — For Faster Response)

The local Whisper STT works but is slow on Pi 4 (~2-7s). Cloud STT gets you to ~300ms.

### Option A: Home Assistant Cloud (Easiest, $6.50/month)

1. Go to **Settings** → **Home Assistant Cloud**
2. Sign up for Nabu Casa ($6.50/month)
3. Once connected, go to **Settings** → **Voice Assistants**
4. Edit your assistant → Change **Speech-to-text** to **Home Assistant Cloud**
5. That's it. Includes cloud TTS too.

### Option B: OpenAI Whisper API (Pay-per-use)

1. Get an API key from https://platform.openai.com/api-keys
2. In HA, install the **OpenAI Conversation** integration
3. Use the Whisper API endpoint for STT
4. Cost: ~$0.006/minute of audio

### Fallback Strategy

Home Assistant doesn't natively switch between cloud and local STT automatically. But you can set up two pipelines:

1. **"Fast" pipeline:** Cloud STT + Ollama/OpenAI + Piper TTS
2. **"Offline" pipeline:** Local Whisper STT + HA Assist (built-in intent matching) + Piper TTS

To switch manually: **Settings** → **Voice Assistants** → change the assigned pipeline for each spoke.

To automate switching, the community has built an `asr_proxy` tool, but it requires some technical setup beyond the scope of this guide.

---

## What's Next?

Once you have the basic system working:

- **Add smart devices** — lights, plugs, thermostats. HA supports thousands of devices. Voice control becomes much more useful when you can say "turn off the bedroom light."
- **Create automations** — "When I say goodnight, turn off all lights and lock the doors"
- **Customize wake words** — OpenWakeWord can train custom wake words using synthetic data
- **Add more spokes** — the Pi 4 handles up to 5 simultaneous satellites
- **Try different TTS voices** — Piper has 100+ voices in 30+ languages

---

## Architecture Reference

```
┌──── YOUR HOME NETWORK ────────────────────────────┐
│                                                     │
│   ┌─── Ethernet (PoE) ───┐                         │
│   │                       │                         │
│   │   RASPBERRY PI 4      │                         │
│   │   Home Assistant OS   │                         │
│   │   ┌─────────────────┐ │                         │
│   │   │ Voice Pipeline  │ │     ┌────── WiFi ─────┐│
│   │   │                 │ │     │                  ││
│   │   │ OpenWakeWord ───┤ │     │  ┌────────────┐  ││
│   │   │ Whisper STT ────┤ │ ←───┤  │ Spoke 1    │  ││
│   │   │ Piper TTS ──────┤ │     │  │ ATOM Echo  │  ││
│   │   │ Ollama LLM ─────┤ │     │  └────────────┘  ││
│   │   │                 │ │     │                  ││
│   │   │ Smart Home ─────┤ │     │  ┌────────────┐  ││
│   │   │ Control         │ │ ←───┤  │ Spoke 2    │  ││
│   │   └─────────────────┘ │     │  │ DIY XIAO   │  ││
│   │                       │     │  │ +battery   │  ││
│   └───────────────────────┘     │  │ +3.5mm     │  ││
│                                 │  └────────────┘  ││
│   ┌─── Ollama Server ────┐     │                  ││
│   │  (separate PC, opt.) │     │  ┌────────────┐  ││
│   │   llama3.2 / GPT-4o  │ ←───┤  │ Spoke 3    │  ││
│   └───────────────────────┘     │  └────────────┘  ││
│                                 │                  ││
│                                 │  ┌────────────┐  ││
│                                 │  │ Spoke 4    │  ││
│                              ←──┤  └────────────┘  ││
│                                 └──────────────────┘│
└─────────────────────────────────────────────────────┘
```

---

*Guide generated 2026-02-17. Based on Home Assistant 2025.x/2026.x, ESPHome 2025.x, Wyoming protocol. Check https://www.home-assistant.io/voice_control/ for the latest setup instructions.*
