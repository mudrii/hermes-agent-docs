
# Voice & TTS

Hermes Agent supports both text-to-speech output and voice message transcription across all messaging platforms.

If you have a paid Nous Portal subscription, OpenAI TTS can also be routed through the [Nous Tool Gateway](tool-gateway.md) without a separate `VOICE_TOOLS_OPENAI_KEY`.

## Text-to-Speech

Convert text to speech with nine providers:

| Provider | Quality | Cost | API Key |
|----------|---------|------|---------|
| **Edge TTS** (default) | Good | Free | None needed |
| **ElevenLabs** | Excellent | Paid | `ELEVENLABS_API_KEY` |
| **OpenAI TTS** | Good | Paid | `VOICE_TOOLS_OPENAI_KEY` |
| **xAI TTS** | Excellent | Paid | `XAI_API_KEY` |
| **MiniMax TTS** | Excellent | Paid | `MINIMAX_API_KEY` |
| **Mistral (Voxtral TTS)** | Excellent | Paid | `MISTRAL_API_KEY` |
| **Google Gemini TTS** | Excellent | Paid | `GEMINI_API_KEY` (or `GOOGLE_API_KEY`) |
| **NeuTTS** | Good | Free (local) | None needed |
| **KittenTTS** | Good | Free (local, ~25 MB) | None needed |

### Platform Delivery

| Platform | Delivery | Format |
|----------|----------|--------|
| Telegram | Voice bubble (plays inline) | Opus `.ogg` |
| Discord | Voice bubble (Opus/OGG), falls back to file attachment | Opus/MP3 |
| WhatsApp | Audio file attachment | MP3 |
| CLI | Saved to `~/.hermes/audio_cache/` | MP3 |

### Configuration

```yaml
# In ~/.hermes/config.yaml
tts:
  provider: "edge"              # "edge" | "elevenlabs" | "openai" | "xai" | "minimax" | "mistral" | "gemini" | "neutts" | "kittentts"
  speed: 1.0                    # Global speed multiplier (provider-specific settings override this)
  edge:
    voice: "en-US-AriaNeural"   # 322 voices, 74 languages
    speed: 1.0                  # Converted to rate percentage (+/-%)
  elevenlabs:
    voice_id: "pNInz6obpgDQGcFmaJgB"  # Adam
    model_id: "eleven_multilingual_v2"
  openai:
    model: "gpt-4o-mini-tts"
    voice: "alloy"              # alloy, echo, fable, onyx, nova, shimmer
    base_url: "https://api.openai.com/v1"  # Override for OpenAI-compatible TTS endpoints
    speed: 1.0                  # 0.25 - 4.0
  xai:
    voice_id: "eve"             # See https://docs.x.ai/docs/models#text-to-speech for available voices
    language: "en"
    sample_rate: 24000
    bit_rate: 128000
  minimax:
    model: "speech-2.8-hd"     # speech-2.8-hd (default), speech-2.8-turbo
    voice_id: "English_Graceful_Lady"  # See https://platform.minimax.io/faq/system-voice-id
    speed: 1                    # 0.5 - 2.0
    vol: 1                      # 0 - 10
    pitch: 0                    # -12 - 12
  mistral:
    model: "voxtral-mini-tts-2603"
    voice_id: "c69964a6-ab8b-4f8a-9465-ec0925096ec8"  # Paul - Neutral (default)
  gemini:
    model: "gemini-2.5-flash-preview-tts"
    voice: "Kore"               # Prebuilt voice name (e.g. Kore, Puck, Charon, ...)
  neutts:
    ref_audio: ''
    ref_text: ''
    model: neuphonic/neutts-air-q4-gguf
    device: cpu
  kittentts:
    model: "KittenML/kitten-tts-nano-0.8-int8"  # ~25 MB local model
```

:::tip Per-provider character caps
Each TTS provider also accepts an optional `max_text_length:` override for the per-request input-character cap. Defaults: OpenAI 4096, xAI 15000, MiniMax 10000, ElevenLabs 5k–40k (model-aware), Gemini 5000, Edge 5000, Mistral 4000, NeuTTS/KittenTTS 2000.
:::

**Speed control**: The global `tts.speed` value applies to all providers by default. Each provider can override it with its own `speed` setting (e.g., `tts.openai.speed: 1.5`). Provider-specific speed takes precedence over the global value. Default is `1.0` (normal speed).

### Telegram Voice Bubbles & ffmpeg

Telegram voice bubbles require Opus/OGG audio format:

- **OpenAI, ElevenLabs, and Mistral** produce Opus natively — no extra setup
- **Edge TTS** (default) outputs MP3 and needs **ffmpeg** to convert:
- **xAI TTS** outputs MP3 (or WAV) and needs **ffmpeg** to convert for Telegram voice bubbles
- **MiniMax TTS** outputs MP3 and needs **ffmpeg** to convert for Telegram voice bubbles
- **Google Gemini TTS** returns raw PCM (wrapped as WAV) and needs **ffmpeg** to convert
- **NeuTTS** outputs WAV and also needs **ffmpeg** to convert for Telegram voice bubbles
- **KittenTTS** outputs WAV (local 25 MB model) and also needs **ffmpeg** to convert

```bash
# Ubuntu/Debian
sudo apt install ffmpeg

# macOS
brew install ffmpeg

# Fedora
sudo dnf install ffmpeg
```

Without ffmpeg, Edge TTS, xAI TTS, MiniMax TTS, Gemini TTS, NeuTTS, and KittenTTS audio are sent as regular audio files (playable, but shown as a rectangular player instead of a voice bubble).

:::tip
If you want voice bubbles without installing ffmpeg, switch to the OpenAI, ElevenLabs, or Mistral provider.
:::

## Voice Message Transcription (STT)

Voice messages sent on Telegram, Discord, WhatsApp, Slack, or Signal are automatically transcribed and injected as text into the conversation. The agent sees the transcript as normal text.

| Provider | Quality | Cost | API Key |
|----------|---------|------|---------| 
| **Local Whisper** (default) | Good | Free | None needed |
| **Groq Whisper API** | Good–Best | Free tier | `GROQ_API_KEY` |
| **OpenAI Whisper API** | Good–Best | Paid | `VOICE_TOOLS_OPENAI_KEY` or `OPENAI_API_KEY` |
| **Mistral Voxtral Transcribe** | Excellent | Paid | `MISTRAL_API_KEY` |
| **xAI Grok STT** | Excellent | Paid | `XAI_API_KEY` |

:::info Zero Config
Local transcription works out of the box when `faster-whisper` is installed. If that's unavailable, Hermes can also use a local `whisper` CLI from common install locations (like `/opt/homebrew/bin`) or a custom command via `HERMES_LOCAL_STT_COMMAND`.
:::

### Configuration

```yaml
# In ~/.hermes/config.yaml
stt:
  provider: "local"           # "local" | "groq" | "openai" | "mistral" | "xai"
  local:
    model: "base"             # tiny, base, small, medium, large-v3
  openai:
    model: "whisper-1"        # whisper-1, gpt-4o-mini-transcribe, gpt-4o-transcribe
  mistral:
    model: "voxtral-mini-latest"  # voxtral-mini-latest, voxtral-mini-2602
  xai:
    language: ""              # auto-detect by default; set to "en", "es", "fr", etc. to force
    format: true              # Inverse Text Normalization (punctuation, casing, numerals)
    diarize: false            # Speaker diarization (per-segment speaker labels)
```

### Provider Details

**Local (faster-whisper)** — Runs Whisper locally via [faster-whisper](https://github.com/SYSTRAN/faster-whisper). Uses CPU by default, GPU if available. Model sizes:

| Model | Size | Speed | Quality |
|-------|------|-------|---------|
| `tiny` | ~75 MB | Fastest | Basic |
| `base` | ~150 MB | Fast | Good (default) |
| `small` | ~500 MB | Medium | Better |
| `medium` | ~1.5 GB | Slower | Great |
| `large-v3` | ~3 GB | Slowest | Best |

**Groq API** — Requires `GROQ_API_KEY`. Good cloud fallback when you want a free hosted STT option.

**OpenAI API** — Accepts `VOICE_TOOLS_OPENAI_KEY` first and falls back to `OPENAI_API_KEY`. Supports `whisper-1`, `gpt-4o-mini-transcribe`, and `gpt-4o-transcribe`.

**Mistral API (Voxtral Transcribe)** — Requires `MISTRAL_API_KEY`. Uses Mistral's [Voxtral Transcribe](https://docs.mistral.ai/capabilities/audio/speech_to_text/) models. Supports 13 languages, speaker diarization, and word-level timestamps. Install with `pip install hermes-agent[mistral]`.

**xAI API (Grok STT)** — Requires `XAI_API_KEY`. Uses the xAI `POST /v1/stt` endpoint (`grok-stt` model) with multipart/form-data uploads. Supports 21 languages, Inverse Text Normalization (punctuation/casing/numerals via `format: true`), speaker diarization (`diarize: true`), and word-level timestamps. Override the endpoint via `stt.xai.base_url` or the `XAI_STT_BASE_URL` environment variable.

**Custom local CLI fallback** — Set `HERMES_LOCAL_STT_COMMAND` if you want Hermes to call a local transcription command directly. The command template supports `{input_path}`, `{output_dir}`, `{language}`, and `{model}` placeholders.

### Fallback Behavior

If your configured provider isn't available, Hermes automatically falls back:
- **Local faster-whisper unavailable** → Tries a local `whisper` CLI or `HERMES_LOCAL_STT_COMMAND` before cloud providers
- **Groq key not set** → Falls back to local transcription, then OpenAI
- **OpenAI key not set** → Falls back to local transcription, then Groq
- **Mistral key/SDK not set** → Skipped in auto-detect; falls through to next available provider
- **xAI key not set** → Skipped in auto-detect; falls through to next available provider
- **Nothing available** → Voice messages pass through with an accurate note to the user

## CLI Record Beep Toggle (v0.11.0)

The CLI plays short start/stop beeps when you trigger push-to-talk recording. If you find the beeps distracting (or you're recording over a microphone that picks them up), disable them with `voice.beep_enabled` in `~/.hermes/config.yaml`:

```yaml
voice:
  record_key: "ctrl+b"
  max_recording_seconds: 120
  beep_enabled: false   # Mute the CLI record-start/stop beeps (default: true)
  silence_threshold: 200
  silence_duration: 3.0
```

This flag only affects the local CLI voice loop — it does not alter audio sent to or received from any messaging platform.

## Alternative: Nous Tool Gateway

Paid [Nous Portal](https://portal.nousresearch.com) subscribers (non-free tier) can use OpenAI TTS via the Nous gateway with no `OPENAI_API_KEY`. Set both `provider` and `use_gateway` in `~/.hermes/config.yaml`:

```yaml
tts:
  provider: openai
  use_gateway: true
```

When `use_gateway: true`, both the `text_to_speech` tool and voice mode route through the gateway. A direct `OPENAI_API_KEY` is not required. For setup, run `hermes model` → Nous Portal → follow tool prompts.

See [Nous Tool Gateway](/docs/nous-tool-gateway) for the full guide.
