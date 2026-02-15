# Building a Voice Assistant 

- Part 1: Create assistant w/Raspberry Pi 4 + USB mic + raspOVOS
- Part 2: Add LLM with `openai/gpt-4o-mini` for additional features

_**Instructions:**_
[https://github.com/jaredstanko/scratchpad/](https://github.com/jaredstanko/scratchpad/)

---

## Part 1 — raspOVOS + USB microphone 

### 1) Boot raspOVOS

1. Download the **raspOVOS** image and flash it to microSD.
2. Boot the Pi, connect to network, log in.

On raspOVOS, your user config is typically:

* `~/.config/mycroft/mycroft.conf` ([GitHub][1])

---

### 2) Verify the USB mic is detected (OS-level)

Plug in the USB mic, then run:

```bash
arecord -l
```

You should see your USB mic listed as a capture device.

Quick record/play test (often `hw:1,0` for the USB mic, but check your `arecord -l` output):

```bash
arecord -D plughw:1,0 -f cd -t wav -d 5 test.wav
aplay test.wav
```

If you hear your recording back, the mic works at the OS level.

---

### 3) Make OVOS listen to the correct mic (recommended method)

The most reliable way on Pi is to use the SoundDevice microphone plugin so you can explicitly pick a device. ([GitHub][1])

Install it:

```bash
pip install ovos-microphone-plugin-sounddevice
```

Edit your OVOS config:

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

* Start with `"device": 1`.
* If it doesn’t hear you, try `0`, then `2`, etc. (device numbering depends on what audio devices you have plugged in).

---

### 4) Smoke test OVOS audio + listening

Run:

```bash
ovos-speak "Microphone setup test."
ovos-listen
```

* `ovos-speak` confirms your speaker output is working.
* `ovos-listen` should “wake up” and actually capture speech.

If OVOS speaks but doesn’t hear you: it’s almost always the wrong `device` index in the config.

---

# Part 2 — LLM integration for short, thorough answers

This uses OVOS “Personas” + the OVOS OpenAI solver plugin, pointed at OpenRouter’s OpenAI-compatible API.

### 1) Get an OpenRouter API key

Create an API key in OpenRouter. Authentication is via:

* `Authorization: Bearer <key>` ([OpenRouter][2])

OpenRouter’s OpenAI-compatible base URL is:

* `https://openrouter.ai/api/v1` ([OpenRouter][2])

---

### 2) Install the OVOS OpenAI solver plugin

On the Pi:

```bash
pip install ovos-openai-plugin
```

(That plugin provides the solver used by personas.) ([GitHub][1])

---

### 3) Create a Persona that calls OpenRouter + `openai/gpt-4o-mini`

Create a persona file:

```bash
mkdir -p ~/.config/ovos_persona
nano ~/.config/ovos_persona/openrouter-gpt4o-mini-short.json
```

Paste this:

```json
{
  "name": "OpenRouter GPT-4o-mini",
  "solvers": ["ovos-solver-openai-plugin"],
  "ovos-solver-openai-plugin": {
    "api_url": "https://openrouter.ai/api/v1",
    "key": "OR_YOUR_OPENROUTER_KEY_HERE",
    "model": "openai/gpt-4o-mini",
    "system_prompt": "You are a voice assistant. Answer in 3–6 sentences. Be concise but thorough. Use plain language. If the request is ambiguous, ask ONE short clarifying question. No filler."
  }
}
```

Why these fields:

* OpenRouter base URL: `https://openrouter.ai/api/v1` ([OpenRouter][2])
* Model name on OpenRouter: `openai/gpt-4o-mini` ([OpenRouter][3])

---

### 4) Enable Personas as fallback (so it answers when no skill matches)

Edit:

```bash
nano ~/.config/mycroft/mycroft.conf
```

Add (or merge) something like:

```json
{
  "intents": {
    "persona": {
      "handle_fallback": true,
      "default_persona": "OpenRouter GPT-4o-mini"
    }
  }
}
```

This tells OVOS to use your persona for fallback answers (great for “general questions”).

---

### 5) Restart OVOS services

If your raspOVOS image uses systemd services, a safe generic approach is reboot:

```bash
sudo reboot
```

---

### 6) Test it

After reboot:

```bash
ovos-say-to "Explain what Petrichor is."
```

Then try voice:

1. `ovos-listen`
2. Ask a general question (that won’t match an installed skill), like:

   * “What’s the difference between RAM and storage?”

If configured correctly, the response should follow your “short but thorough” prompt style.

---

## Notes that save headaches

* OpenRouter is OpenAI-chat compatible and normalizes responses to OpenAI’s chat schema. ([OpenRouter][4])
* If you ever change USB devices, your mic device index may change—re-check and adjust `"device"`.


[1]: https://github.com/OpenVoiceOS/ovos-openai-plugin "OpenVoiceOS/ovos-openai-plugin - GitHub"
[2]: https://openrouter.ai/docs/api/reference/authentication "API Authentication | OpenRouter OAuth and API Keys | Documentation"
[3]: https://openrouter.ai/openai/gpt-4o-mini "GPT-4o-mini - API, Providers, Stats - OpenRouter"
[4]: https://openrouter.ai/docs/api/reference/overview "OpenRouter API Reference | Complete API Documentation"
