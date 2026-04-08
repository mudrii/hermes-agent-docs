# Text-to-Speech

Hermes Agent supports text-to-speech output across all platforms with five providers: Edge TTS (free, default), ElevenLabs (premium), OpenAI TTS, NeuTTS (local, free), and MiniMax TTS (high-quality with voice cloning).

TTS has been available since v0.2.0. NeuTTS was promoted from an optional skill to a first-class built-in provider in v0.4.0 ([PR #1657](https://github.com/NousResearch/hermes-agent/pull/1657), [#1664](https://github.com/NousResearch/hermes-agent/pull/1664)). OpenAI TTS `base_url` support was added in v0.4.0 ([PR #2064](https://github.com/NousResearch/hermes-agent/pull/2064)). MiniMax TTS (speech-2.8-hd) was added in v0.8.0 ([PR #4963](https://github.com/NousResearch/hermes-agent/pull/4963)).

Implemented in `tools/tts_tool.py`.

## Providers

| Provider | Quality | Cost | API Key | Notes |
|----------|---------|------|---------|-------|
| **Edge TTS** (default) | Good | Free | None needed | Microsoft Edge neural voices, 322 voices, 74 languages |
| **ElevenLabs** | Excellent | Paid | `ELEVENLABS_API_KEY` | Premium quality, streaming support |
| **OpenAI TTS** | Good | Paid | `VOICE_TOOLS_OPENAI_KEY` | Falls back to `OPENAI_API_KEY` |
| **MiniMax TTS** | High | Paid | `MINIMAX_API_KEY` | speech-2.8-hd model, voice cloning support (v0.8.0) |
| **NeuTTS** | Good | Free | None needed | Local on-device, requires `espeak-ng` |

## Platform Delivery

| Platform | Delivery | Format |
|----------|----------|--------|
| Telegram | Voice bubble (plays inline) | Opus `.ogg` |
| Discord | Voice bubble (Opus/OGG), falls back to file attachment | Opus/MP3 |
| WhatsApp | Audio file attachment | MP3 |
| CLI | Saved to `~/.hermes/audio_cache/` | MP3 |

The `text_to_speech` tool returns a `MEDIA:` path that the platform delivers as a voice message. On Telegram it plays as a voice bubble; on Discord/WhatsApp as an audio attachment; in CLI mode, saves to `~/voice-memos/`.

## Configuration

```yaml
# In ~/.hermes/config.yaml
tts:
  provider: "edge"              # "edge" | "elevenlabs" | "openai" | "minimax" | "neutts"
  edge:
    voice: "en-US-AriaNeural"   # 322 voices, 74 languages
  elevenlabs:
    voice_id: "pNInz6obpgDQGcFmaJgB"  # Adam
    model_id: "eleven_multilingual_v2"
    streaming_model_id: "eleven_flash_v2_5"  # Used for real-time streaming
  openai:
    model: "gpt-4o-mini-tts"
    voice: "alloy"              # alloy, echo, fable, onyx, nova, shimmer
    base_url: "https://api.openai.com/v1"  # Override for compatible endpoints
  minimax:
    model: "speech-2.8-hd"     # speech-2.8-hd (default) or speech-2.8-turbo
    voice_id: "English_Graceful_Lady"
    speed: 1
    vol: 1
    pitch: 0
    base_url: "https://api.minimax.io/v1/t2a_v2"
  neutts:
    ref_audio: ''
    ref_text: ''
    model: neuphonic/neutts-air-q4-gguf
    device: cpu
```

## Telegram Voice Bubbles and ffmpeg

Telegram voice bubbles require Opus/OGG audio format:

- **OpenAI and ElevenLabs** produce Opus natively -- no extra setup
- **Edge TTS** (default) outputs MP3 and needs **ffmpeg** to convert
- **MiniMax TTS** outputs MP3 and needs **ffmpeg** to convert for Telegram voice bubbles
- **NeuTTS** outputs WAV and also needs **ffmpeg** to convert for Telegram voice bubbles

```bash
# macOS
brew install ffmpeg

# Ubuntu/Debian
sudo apt install ffmpeg
```

Without ffmpeg, Edge TTS and NeuTTS audio are sent as regular audio files (playable, but shown as a rectangular player instead of a voice bubble).

## Provider Details

### Edge TTS

Default provider. Uses Microsoft Edge's neural TTS API. Free, no API key required. Supports 322 voices across 74 languages. Default voice: `en-US-AriaNeural`.

Providers are imported lazily -- Edge TTS is only imported when actually used to avoid crashing in headless environments (SSH, Docker, WSL, no PortAudio).

### ElevenLabs

Premium quality TTS. Requires `ELEVENLABS_API_KEY` in `~/.hermes/.env`.

For streaming TTS in CLI voice mode (sentence-by-sentence playback), ElevenLabs uses the `eleven_flash_v2_5` model by default (faster than `eleven_multilingual_v2`). This can be overridden via `tts.elevenlabs.streaming_model_id` in config.yaml. Streaming TTS outputs PCM audio at 24kHz.

### OpenAI TTS

Requires `VOICE_TOOLS_OPENAI_KEY` (falls back to `OPENAI_API_KEY`). Available voices: alloy, echo, fable, onyx, nova, shimmer.

v0.4.0 added `base_url` support, enabling use with self-hosted or proxy TTS endpoints.

### MiniMax TTS

High-quality cloud TTS with voice cloning support. Added in v0.8.0 ([PR #4963](https://github.com/NousResearch/hermes-agent/pull/4963)). Requires `MINIMAX_API_KEY` in `~/.hermes/.env`.

Get an API key at [platform.minimax.io](https://platform.minimax.io/).

MiniMax uses the T2A v2 HTTP API with hex-encoded audio decoding. Supports both standard (`speech-2.8-hd`) and fast (`speech-2.8-turbo`) model variants. Voice parameters (speed, volume, pitch) are configurable via `config.yaml`.

### NeuTTS

Local on-device TTS via `neutts_cli`. Free, no API key required.

**Setup:**

```bash
python -m pip install -U neutts[all]

# System dependency
# macOS
brew install espeak-ng

# Ubuntu/Debian
sudo apt install espeak-ng
```

NeuTTS runs as a subprocess from `tools/neutts_synth.py` using the `neuphonic/neutts-air-q4-gguf` model by default. Configurable `ref_audio` and `ref_text` for voice cloning.

## Voice Message Transcription (STT)

Voice messages sent on Telegram, Discord, WhatsApp, Slack, or Signal are automatically transcribed and injected as text into the conversation.

| Provider | Quality | Cost | API Key |
|----------|---------|------|---------|
| **Local Whisper** (default) | Good | Free | None needed |
| **Groq Whisper API** | Good-Best | Free tier | `GROQ_API_KEY` |
| **OpenAI Whisper API** | Good-Best | Paid | `VOICE_TOOLS_OPENAI_KEY` or `OPENAI_API_KEY` |

### STT Configuration

```yaml
stt:
  provider: "local"           # "local" | "groq" | "openai"
  local:
    model: "base"             # tiny, base, small, medium, large-v3
  openai:
    model: "whisper-1"        # whisper-1, gpt-4o-mini-transcribe, gpt-4o-transcribe
```

Local transcription works out of the box when `faster-whisper` is installed. If that's unavailable, Hermes can also use a local `whisper` CLI or a custom command via `HERMES_LOCAL_STT_COMMAND`.

### Local Whisper Model Sizes

| Model | Size | Speed | Quality |
|-------|------|-------|---------|
| `tiny` | ~75 MB | Fastest | Basic |
| `base` | ~150 MB | Fast | Good (default) |
| `small` | ~500 MB | Medium | Better |
| `medium` | ~1.5 GB | Slower | Great |
| `large-v3` | ~3 GB | Slowest | Best |

### STT Fallback Behavior

If your configured provider isn't available, Hermes automatically falls back:

- **Local faster-whisper unavailable** -- tries a local `whisper` CLI or `HERMES_LOCAL_STT_COMMAND` before cloud providers
- **Groq key not set** -- falls back to local transcription, then OpenAI
- **OpenAI key not set** -- falls back to local transcription, then Groq
- **Nothing available** -- voice messages pass through with a note to the user

## Text Cleaning

Text is automatically cleaned before TTS: markdown formatting, code blocks, URLs, and `<think>...</think>` blocks are stripped. Maximum text length per TTS call is 4000 characters.

## v0.8.0 Changes

- **MiniMax TTS provider** -- MiniMax (speech-2.8-hd model) added as a fifth TTS provider with voice cloning support. Requires `MINIMAX_API_KEY`. Configure via `tts.provider: "minimax"` in `config.yaml` ([#4963](https://github.com/NousResearch/hermes-agent/pull/4963)).
