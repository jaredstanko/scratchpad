## Raspberry Pi 4 + **USB mic** + **raspOVOS**: simple setup instructions

(Goal: OpenVoiceOS working + microphone listening reliably, then OpenAI API answers that are short but thorough)

---

# Part 1 — raspOVOS install + USB mic working

## 1) Flash and boot raspOVOS

1. Download the **raspOVOS (Raspberry Pi 4)** image from the OVOS downloads page.
2. Flash it to a microSD (Raspberry Pi Imager or Balena Etcher).
3. Boot the Pi (Ethernet recommended for first boot), log in, and make sure it has internet.

---

## 2) Plug in the USB microphone and confirm the Pi sees it

Run:

```bash
arecord -l
```

You should see your USB mic listed as a capture device.

### Quick record/playback test (proves the mic works at the OS level)

1. Record 5 seconds:

```bash
arecord -f cd -d 5 test.wav
```

2. Play it back:

```bash
aplay test.wav
```

If you can hear your recording, the mic hardware and ALSA are good.

---

## 3) Confirm OVOS basics work (speaker output)

Run:

```bash
ovos-speak "OpenVoiceOS audio output test."
```

If you hear speech, your output is fine.

---

## 4) Make OVOS listen on the USB mic (most reliable method)

On raspOVOS, the user configuration file is typically:
`~/.config/mycroft/mycroft.conf`

The most controllable way to select a specific mic is the **SoundDevice microphone plugin**, which supports choosing a device in config.

### 4A) Install the SoundDevice mic plugin

```bash
pip install ovos-microphone-plugin-sounddevice
```

### 4B) Find the USB mic “device index”

SoundDevice sees devices slightly differently than `arecord`. Easiest approach: try indices until it hears you.

Start with `device: 0`, then `1`, then `2`, etc.

### 4C) Set the mic plugin + device in `mycroft.conf`

Edit:

```bash
nano ~/.config/mycroft/mycroft.conf
```

Add (or merge) this:

```json
{
  "listener": {
    "microphone": {
      "module": "ovos-microphone-plugin-sounddevice",
      "device": 1
    }
  }
}
```

* Change `"device": 1` to 0/2/3 if needed.

---

## 5) Restart OVOS services

On raspOVOS, service names can vary a bit by image/version, but these are the common ways to restart:

### Option A (systemd service name used on many installs)

```bash
sudo systemctl restart ovos
```

### Option B (reboot = always works)

```bash
sudo reboot
```

---

## 6) Test that OVOS is actually hearing you

Run:

```bash
ovos-listen
```

Speak after the listening cue.

If it’s not responding:

* Change the `"device"` number in `mycroft.conf`
* Restart/reboot
* Try again

(With USB mics, “wrong device index” is the #1 issue.)

---

# Part 2 — OpenAI API integration for short but thorough answers (Personas)

OVOS now prefers **Personas** for LLM responses; older ChatGPT fallback skills have been replaced by this approach.

## 1) Create and store your OpenAI API key safely

Use OpenAI’s key safety guidance (don’t paste keys into screenshots, repos, etc.).

---

## 2) Install the OVOS OpenAI persona plugin

```bash
pip install ovos-openai-plugin
```

---

## 3) Create a persona that enforces “short but thorough”

Create this file:

```bash
mkdir -p ~/.config/ovos_persona
nano ~/.config/ovos_persona/openai-short-thorough.json
```

Paste:

```json
{
  "name": "OpenAI Short Thorough",
  "solvers": ["ovos-solver-openai-plugin"],
  "ovos-solver-openai-plugin": {
    "api_url": "https://api.openai.com/v1",
    "key": "sk-YOUR_KEY_HERE",
    "model": "gpt-4.1-mini",
    "system_prompt": "You are a voice assistant. Answer in 3–6 sentences. Be brief but complete. Use plain language. If needed, ask ONE clarifying question. Avoid long lists unless requested."
  }
}
```

This JSON structure (persona + solver config) matches the ovos-openai-plugin documentation.

**Optional (recommended):** instead of putting your key in the file, set an environment variable and reference it if your setup supports it—this avoids leaving keys in plaintext. (Key safety best practice.)

---

## 4) Tell OVOS to use the persona as a fallback (so it answers anything)

Edit:

```bash
nano ~/.config/mycroft/mycroft.conf
```

Add (or merge) this persona config:

```json
{
  "intents": {
    "persona": {
      "handle_fallback": true,
      "default_persona": "OpenAI Short Thorough"
    }
  }
}
```

The OVOS Personas manual documents setting a default persona and enabling persona fallback behavior via config.

Restart OVOS (or reboot) after saving.

---

## 5) Use it

Try voice or typed testing:

### Voice

* Ask something that isn’t a built-in skill, e.g. “Explain what DNS is.”

### Typed (handy for debugging)

```bash
ovos-say-to "Explain DNS in simple terms."
```

If fallback is enabled, OVOS should answer via the persona when no skill matches.

---

## Troubleshooting quick hits (USB mic + raspOVOS)

* **`arecord` works but `ovos-listen` doesn’t:** change the SoundDevice `"device"` index.
* **No sound output:** test `ovos-speak "test"` and confirm your speaker/default output is correct.
* **Slow responses:** use a smaller/faster model and keep the system prompt strict.

---

If you want, tell me what `arecord -l` shows (just the device names, no serials) and I’ll suggest the most likely `"device"` index to try first.
