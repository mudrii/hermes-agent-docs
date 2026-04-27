# Voice Mode

Hermes Agent supports full voice interaction across CLI and messaging platforms, introduced as a complete feature in v0.3.0 and extended in v0.4.0. You can talk to the agent using your microphone, hear spoken replies, and hold live voice conversations in Discord voice channels.

## Voice Mode Variants

| Mode | Platform | Description |
|------|----------|-------------|
| **Interactive Voice** | CLI | Press Ctrl+B to record; agent auto-detects silence and responds |
| **Auto Voice Reply** | Telegram, Discord | Agent sends spoken audio alongside text responses |
| **Voice Channel** | Discord | Bot joins VC, listens to users speaking, speaks replies back |

## Prerequisites

Before using voice features:

1. Hermes Agent installed: `pip install hermes-agent`
2. An LLM provider configured (run `hermes model` or set credentials in `~/.hermes/.env`)
3. Working base text mode — run `hermes` and verify the agent responds to text

## Installing Voice Dependencies

### Python packages

```bash
# CLI voice mode (microphone + audio playback)
pip install hermes-agent[voice]

# Discord + Telegram messaging (includes discord.py[voice] for VC support)
pip install hermes-agent[messaging]

# Premium TTS (ElevenLabs)
pip install hermes-agent[tts-premium]

# Local TTS (NeuTTS, optional)
python -m pip install -U neutts[all]

# Everything at once
pip install hermes-agent[all]
```

| Extra | Packages | Required for |
|-------|----------|-------------|
| `voice` | `sounddevice`, `numpy` | CLI voice mode |
| `messaging` | `discord.py[voice]`, `python-telegram-bot`, `aiohttp` | Discord and Telegram bots |
| `tts-premium` | `elevenlabs` | ElevenLabs TTS provider |

`discord.py[voice]` installs PyNaCl (for voice encryption) and opus bindings automatically. This is required for Discord voice channel support.

### System dependencies

```bash
# macOS
brew install portaudio ffmpeg opus
brew install espeak-ng   # for NeuTTS

# Ubuntu/Debian
sudo apt install portaudio19-dev ffmpeg libopus0
sudo apt install espeak-ng   # for NeuTTS
```

| Dependency | Purpose | Required for |
|-----------|---------|-------------|
| **PortAudio** | Microphone input and audio playback | CLI voice mode |
| **ffmpeg** | Audio format conversion (MP3 to Opus, PCM to WAV) | All platforms |
| **Opus** | Discord voice codec | Discord voice channels |
| **espeak-ng** | Phonemizer backend | Local NeuTTS provider |

## CLI Voice Mode

### Enable voice mode

Start Hermes and enable voice mode inside the CLI:

```bash
hermes
```

Then:

```
/voice          Toggle voice mode on/off
/voice on       Enable voice mode
/voice off      Disable voice mode
/voice tts      Toggle TTS output
/voice status   Show current state
```

### Recording flow

1. Start the CLI with `hermes` and enable voice mode with `/voice on`
2. **Press Ctrl+B** — a beep plays (880Hz), recording starts
3. **Speak** — a live audio level bar shows your input: `● [▁▂▃▅▇▇▅▂] ❯`
4. **Stop speaking** — after 3 seconds of silence, recording auto-stops
5. **Two beeps** play (660Hz) confirming the recording ended
6. Audio is transcribed via Whisper STT and sent to the agent
7. If TTS is enabled, the agent's reply is spoken aloud
8. Recording **automatically restarts** — speak again without pressing any key

The loop continues until you press Ctrl+B during recording (exits continuous mode) or 3 consecutive recordings detect no speech.

The record key is configurable via `voice.record_key` in `~/.hermes/config.yaml` (default: `ctrl+b`).

### Silence detection

Two-stage algorithm detects when you've finished speaking:

1. **Speech confirmation** — waits for audio above the RMS threshold (200) for at least 0.3 seconds, tolerating brief dips between syllables
2. **End detection** — once speech is confirmed, triggers after 3.0 seconds of continuous silence

If no speech is detected at all for 15 seconds, recording stops automatically. Both `silence_threshold` and `silence_duration` are configurable in `config.yaml`.

### Streaming TTS

When TTS is enabled, the agent speaks its reply sentence-by-sentence as it generates text — you don't wait for the full response:

1. Buffers text deltas into complete sentences (minimum 20 characters)
2. Strips markdown formatting and `<think>` blocks
3. Generates and plays audio per sentence in real-time

### Hallucination filter

Whisper sometimes generates phantom text from silence or background noise. The agent filters these out using a set of 26 known hallucination phrases across multiple languages, plus a regex pattern for repetitive variations.

## Gateway Voice Reply (Telegram and Discord)

Start the gateway to connect to messaging platforms:

```bash
hermes gateway
```

### Commands (Telegram and Discord)

```
/voice          Toggle voice mode on/off
/voice on       Voice replies only when you send a voice message
/voice tts      Voice replies for ALL messages
/voice off      Disable voice replies
/voice status   Show current setting
```

### Modes

| Mode | Command | Behavior |
|------|---------|----------|
| `off` | `/voice off` | Text only (default) |
| `voice_only` | `/voice on` | Speaks reply only when you send a voice message |
| `all` | `/voice tts` | Speaks reply to every message |

Voice mode setting is persisted across gateway restarts.

### Platform delivery

| Platform | Format | Notes |
|----------|--------|-------|
| **Telegram** | Voice bubble (Opus/OGG) | Plays inline in chat. ffmpeg converts MP3 to Opus if needed |
| **Discord** | Native voice bubble (Opus/OGG) | Plays inline like a user voice message. Falls back to file attachment if voice bubble API fails |

## Discord Voice Channels

The most immersive voice feature: the bot joins a Discord voice channel, listens to users speaking, transcribes their speech, processes through the agent, and speaks replies back in the voice channel.

### Discord bot permissions required

In addition to normal text bot permissions, add:

| Permission | Purpose | Required |
|-----------|---------|----------|
| **Connect** | Join voice channels | Yes |
| **Speak** | Play TTS audio in voice channels | Yes |
| **Use Voice Activity** | Detect when users are speaking | Recommended |

Updated permissions integer for text and voice: `274881432640`

Re-invite the bot with:

```
https://discord.com/oauth2/authorize?client_id=YOUR_APP_ID&scope=bot+applications.commands&permissions=274881432640
```

### Privileged Gateway Intents

Enable all three in the Developer Portal under Bot > Privileged Gateway Intents:

| Intent | Purpose |
|--------|---------|
| **Presence Intent** | Detect user online/offline status |
| **Server Members Intent** | Map voice SSRC identifiers to Discord user IDs |
| **Message Content Intent** | Read text message content in channels |

**Server Members Intent** is especially critical — without it, the bot cannot identify who is speaking.

### Voice channel commands

Use in the Discord text channel where the bot is present:

```
/voice join      Bot joins your current voice channel
/voice channel   Alias for /voice join
/voice leave     Bot disconnects from voice channel
/voice status    Show voice mode and connected channel
```

You must be in a voice channel before running `/voice join`.

### How Discord voice channel works

When the bot joins a voice channel:

1. **Listens** to each user's audio stream independently
2. **Detects silence** — 1.5 seconds of silence after at least 0.5 seconds of speech triggers processing
3. **Transcribes** the audio via Whisper STT (local, Groq, or OpenAI)
4. **Processes** through the full agent pipeline (session, tools, memory)
5. **Speaks** the reply back in the voice channel via TTS

Transcripts appear in the text channel: `[Voice] @user: what you said`. Agent responses are sent as both text in the channel and audio in the VC. The text channel used is the one where `/voice join` was issued.

Echo prevention: the bot automatically pauses its audio listener while playing TTS replies, preventing it from hearing and re-processing its own output.

Access control: only users listed in `DISCORD_ALLOWED_USERS` can interact via voice.

## Configuration Reference

### config.yaml

```yaml
# Voice recording (CLI)
voice:
  record_key: "ctrl+b"            # Key to start/stop recording
  max_recording_seconds: 120       # Maximum recording length
  auto_tts: false                  # Auto-enable TTS when voice mode starts
  beep_enabled: true               # Play start/stop beeps (set false to mute) — v0.11.0
  silence_threshold: 200           # RMS level (0-32767) below which counts as silence
  silence_duration: 3.0            # Seconds of silence before auto-stop

# Speech-to-Text
stt:
  enabled: true                      # Set to false to disable STT entirely
  provider: "local"                  # "local" (free) | "local_command" | "groq" | "openai" | "mistral" | "xai"
  local:
    model: "base"                    # tiny, base, small, medium, large-v3
    language: ""                     # Force language code (e.g. "en", "es", "fr"); empty = auto-detect
  mistral:
    model: "voxtral-mini-latest"     # Voxtral Transcribe; needs MISTRAL_API_KEY
  xai:
    language: ""                     # auto-detect; or "en", "es", "fr", ... — needs XAI_API_KEY
    format: true                     # Inverse Text Normalization (punctuation/casing/numerals)
    diarize: false                   # Speaker diarization

# Text-to-Speech
tts:
  provider: "edge"                 # "edge" (free) | "elevenlabs" | "openai" | "xai" | "minimax" | "mistral" | "gemini" | "neutts" | "kittentts"
  edge:
    voice: "en-US-AriaNeural"      # 322 voices, 74 languages
  elevenlabs:
    voice_id: "pNInz6obpgDQGcFmaJgB"    # Adam
    model_id: "eleven_multilingual_v2"   # Used for file generation
    streaming_model_id: "eleven_flash_v2_5"  # Used for real-time streaming TTS
  openai:
    model: "gpt-4o-mini-tts"
    voice: "alloy"                 # alloy, echo, fable, onyx, nova, shimmer
  xai:
    voice_id: "eve"                # See https://docs.x.ai/docs/models#text-to-speech
    language: "en"
    sample_rate: 24000
    bit_rate: 128000
  minimax:
    model: "speech-2.8-hd"        # speech-2.8-hd (default) or speech-2.8-turbo
    voice_id: "English_Graceful_Lady"
    speed: 1
    vol: 1
    pitch: 0
  mistral:
    model: "voxtral-mini-tts-2603"
    voice_id: "c69964a6-ab8b-4f8a-9465-ec0925096ec8"  # Paul - Neutral
  gemini:
    model: "gemini-2.5-flash-preview-tts"
    voice: "Kore"                  # Prebuilt voice (Kore, Puck, Charon, ...)
  neutts:
    ref_audio: ''
    ref_text: ''
    model: neuphonic/neutts-air-q4-gguf
    device: cpu
  kittentts:
    model: "KittenML/kitten-tts-nano-0.8-int8"  # ~25 MB local model
```

### API keys (.env)

```bash
# Speech-to-Text providers (local needs no key)
GROQ_API_KEY=...                    # Groq Whisper (fast, free tier)
VOICE_TOOLS_OPENAI_KEY=...         # OpenAI Whisper (paid)
# Falls back to OPENAI_API_KEY if VOICE_TOOLS_OPENAI_KEY is not set

# STT model overrides (optional)
STT_GROQ_MODEL=whisper-large-v3-turbo
STT_OPENAI_MODEL=whisper-1

# Custom local STT command template (optional, for local_command provider)
# Uses {input_path}, {output_dir}, {model}, {language} placeholders
HERMES_LOCAL_STT_COMMAND=
HERMES_LOCAL_STT_LANGUAGE=en        # Fallback language for local STT (prefer stt.local.language in config.yaml)

# Text-to-Speech providers (Edge TTS and NeuTTS need no key)
ELEVENLABS_API_KEY=***             # ElevenLabs (premium quality)
# VOICE_TOOLS_OPENAI_KEY above also enables OpenAI TTS

# Discord voice channel
DISCORD_BOT_TOKEN=...
DISCORD_ALLOWED_USERS=...
```

## STT Provider Comparison

| Provider | Model | Speed | Quality | Cost | API Key |
|----------|-------|-------|---------|------|---------|
| **Local** (faster-whisper) | `base` | Fast (CPU/GPU) | Good | Free | No |
| **Local** (faster-whisper) | `small` | Medium | Better | Free | No |
| **Local** (faster-whisper) | `large-v3` | Slow | Best | Free | No |
| **Local command** | system `whisper` CLI | Varies | Varies | Free | No |
| **Groq** | `whisper-large-v3-turbo` | Very fast (~0.5s) | Good | Free tier | Yes |
| **Groq** | `whisper-large-v3` | Fast (~1s) | Better | Free tier | Yes |
| **Groq** | `distil-whisper-large-v3-en` | Fast | Good (English) | Free tier | Yes |
| **OpenAI** | `whisper-1` | Fast (~1s) | Good | Paid | Yes |
| **OpenAI** | `gpt-4o-mini-transcribe` | Fast | Good | Paid | Yes |
| **OpenAI** | `gpt-4o-transcribe` | Medium (~2s) | Best | Paid | Yes |

Provider auto-detection priority (when `stt.provider` is not explicitly set): **local** (faster-whisper) > **local_command** (system whisper CLI) > **groq** > **openai** > **mistral** > **xai**

When `stt.provider` is explicitly set in config.yaml, that choice is respected -- no silent cloud fallback occurs. If the chosen provider is unavailable, STT is disabled rather than falling back.

If `faster-whisper` is installed, voice mode works with zero API keys for STT. The model (~150 MB for `base`) downloads automatically on first use.

## TTS Provider Comparison

| Provider | Quality | Cost | Latency | Key Required |
|----------|---------|------|---------|-------------|
| **Edge TTS** | Good | Free | ~1s | No |
| **ElevenLabs** | Excellent | Paid | ~2s | Yes |
| **OpenAI TTS** | Good | Paid | ~1.5s | Yes |
| **xAI TTS** | Excellent | Paid | ~1.5s | Yes (`XAI_API_KEY`) |
| **MiniMax TTS** | High | Paid | ~1.5s | Yes (`MINIMAX_API_KEY`) |
| **Mistral (Voxtral) TTS** | Excellent | Paid | ~1.5s | Yes (`MISTRAL_API_KEY`) |
| **Google Gemini TTS** | Excellent | Paid | ~1.5s | Yes (`GEMINI_API_KEY`) |
| **NeuTTS** | Good | Free | Depends on CPU/GPU | No |
| **KittenTTS** | Good | Free | Depends on CPU (~25 MB local) | No |

Edge TTS default voice is `en-US-AriaNeural` (322 voices, 74 languages available). NeuTTS is a local on-device provider run via subprocess from `tools/neutts_synth.py` using the `neuphonic/neutts-air-q4-gguf` model by default. In v0.4.0 NeuTTS was promoted from an optional skill to a first-class built-in TTS provider with a setup flow ([#1657](https://github.com/NousResearch/hermes-agent/pull/1657), [#1664](https://github.com/NousResearch/hermes-agent/pull/1664)). The `neutts` provider is now available in `tts.provider` without installing a separate skill. In v0.8.0, MiniMax TTS was added as a fifth provider with voice cloning support ([#4963](https://github.com/NousResearch/hermes-agent/pull/4963)).

For **streaming TTS** in CLI voice mode (sentence-by-sentence), ElevenLabs uses the `eleven_flash_v2_5` model by default (faster than `eleven_multilingual_v2`). This can be overridden via `tts.elevenlabs.streaming_model_id` in config.yaml. Streaming TTS outputs PCM audio at 24kHz and plays through sounddevice. If sounddevice is unavailable, it falls back to a temporary WAV file played through system audio players.

Text is automatically cleaned before TTS: markdown formatting, code blocks, URLs, and `<think>...</think>` blocks are stripped. Maximum text length per TTS call is 4000 characters.

## Recommended Setup by Use Case

### Best quality

- STT: local `large-v3` or Groq `whisper-large-v3`
- TTS: ElevenLabs

### Best speed and convenience

- STT: local `base` or Groq
- TTS: Edge TTS

### Zero cost

- STT: local (install `faster-whisper`)
- TTS: Edge TTS (no key required)

## v0.11.0 Changes

- **Google Gemini TTS provider** -- New built-in `gemini` provider using Google's `gemini-2.5-flash-preview-tts` model with prebuilt voices (Kore, Puck, Charon, ...). Requires `GEMINI_API_KEY` (or `GOOGLE_API_KEY`) ([#11229](https://github.com/NousResearch/hermes-agent/pull/11229)).
- **xAI TTS provider** -- New built-in `xai` provider using xAI's `/v1/tts` endpoint (default voice `eve`). Requires `XAI_API_KEY` ([#10783](https://github.com/NousResearch/hermes-agent/pull/10783)).
- **KittenTTS local provider** -- New built-in `kittentts` provider using the ~25 MB `KittenML/kitten-tts-nano-0.8-int8` model. Local, free, no API key. Install with `pip install https://github.com/KittenML/KittenTTS/releases/download/0.8.1/kittentts-0.8.1-py3-none-any.whl` ([#13395](https://github.com/NousResearch/hermes-agent/pull/13395)).
- **xAI Grok STT provider** -- New built-in `xai` STT provider using `POST /v1/stt` (`grok-stt` model). Supports 21 languages, Inverse Text Normalization (`format: true`), and speaker diarization (`diarize: true`). Requires `XAI_API_KEY` ([#14473](https://github.com/NousResearch/hermes-agent/pull/14473)).
- **CLI record beep toggle** -- New `voice.beep_enabled` config flag (default `true`) lets you mute the CLI start/stop beeps without disabling voice mode ([#13247](https://github.com/NousResearch/hermes-agent/pull/13247)).

## v0.8.0 Changes

- **MiniMax TTS provider** -- MiniMax (speech-2.8-hd model) added as a fifth TTS provider with voice cloning support. Requires `MINIMAX_API_KEY` ([#4963](https://github.com/NousResearch/hermes-agent/pull/4963)).
- **STT language moved to config.yaml** -- `stt.local.language` can now be set in `config.yaml` (empty = auto-detect). The `HERMES_LOCAL_STT_LANGUAGE` env var still works as a fallback for backward compatibility. Both local faster-whisper and `local_command` STT paths read config.yaml first.

## v0.4.0 Changes

- **NeuTTS promoted to first-class provider** -- NeuTTS (local on-device TTS) is now a built-in provider with an interactive setup wizard. Install with `python -m pip install neutts[all]`; no separate skill required ([#1657](https://github.com/NousResearch/hermes-agent/pull/1657), [#1664](https://github.com/NousResearch/hermes-agent/pull/1664)).
- **STT tool added** -- A `stt` tool is now available in the tool registry, letting the agent invoke Whisper transcription as an explicit tool call alongside the automatic voice-mode transcription ([#2072](https://github.com/NousResearch/hermes-agent/pull/2072)). Uses the same provider stack (`tools/transcription_tools.py`) as the voice pipeline.
- **OpenAI TTS `base_url` support** -- The `openai` TTS provider now accepts a custom `base_url`, enabling use with self-hosted or proxy TTS endpoints ([#2064](https://github.com/NousResearch/hermes-agent/pull/2064)).
- **Voice channel TTS fix with streaming** -- Discord voice channel TTS output was silent when streaming mode was enabled; this is fixed ([#2322](https://github.com/NousResearch/hermes-agent/pull/2322)).

---

## Troubleshooting

**"No audio device found" (CLI):** Install PortAudio: `brew install portaudio` (macOS) or `sudo apt install portaudio19-dev` (Ubuntu).

**Bot doesn't respond in Discord server channels:** The bot requires an @mention by default in server channels. Use DMs instead (no mention needed), or set `DISCORD_REQUIRE_MENTION=false` in `~/.hermes/.env`.

**Bot joins VC but doesn't hear you:** Check your Discord user ID is in `DISCORD_ALLOWED_USERS`, you are not muted in Discord, and privileged intents are enabled. The bot needs a SPEAKING event from Discord before it can map your audio.

**Bot responds in text but not in voice channel:** Check TTS provider configuration and API key quota. Edge TTS (free, no key) is the default fallback.

**Whisper returns garbage text:** The hallucination filter catches most cases. For persistent issues: use a quieter environment, increase `silence_threshold` in config, or try a different STT provider or model.
