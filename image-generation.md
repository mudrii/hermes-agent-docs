# Image Generation

Hermes Agent generates images through the `image_gen` tool. As of **v0.11.0**, `image_gen` is a pluggable surface — Hermes ships several backends out of the box and plugins can register their own. The selected backend is the only thing that changes; the tool name, schema, and call site are stable.

Available since v0.2.0 (FAL FLUX). Backend plugin architecture introduced in v0.11.0.

## Backends

v0.11.0 ships these built-in backends. All but `fal` are new in v0.11.0; `openai`, `xai`, and `openai-codex` are bundled provider plugins under `plugins/image_gen/`. FAL is the legacy default served directly by `tools/image_generation_tool.py` and exposes many model variants (FLUX 2 Pro, Recraft V4 Pro, Nano Banana Pro, etc.) selectable per call.

| Backend | Provider | Best for | Credentials |
|---------|----------|----------|-------------|
| `fal` (default) | FAL.ai | General-purpose photoreal / illustration; multi-model — FLUX 2 Pro, Recraft V4 Pro, Nano Banana Pro, etc. all selectable per call | `FAL_KEY` or Nous Tool Gateway |
| `openai` | OpenAI gpt-image-2 (Responses `image_generation` tool) | Strong instruction following, text rendering | `OPENAI_API_KEY` |
| `xai` | xAI `grok-imagine-image` | Stylised generations via xAI | `XAI_API_KEY` |
| `openai-codex` | OpenAI gpt-image-2 routed over Codex OAuth | Use a ChatGPT subscription instead of a separate API key | Codex OAuth login |

A plugin can add additional backends by calling `ctx.register_image_gen_provider(name, handler)` in its `register()` function — see [Plugins](./plugins.md). Only one backend is active at a time; selection is configured under `image_gen.provider` in `~/.hermes/config.yaml` or interactively via `hermes tools` and `hermes plugins`.

## Selecting a Backend

```yaml
# ~/.hermes/config.yaml
image_gen:
  provider: fal          # fal | openai | xai | openai-codex | <plugin name>
  use_gateway: false     # fal-only: route through Nous Tool Gateway instead of a direct key
```

Or interactively:

```bash
hermes tools     # toggle the image_gen tool and pick a backend
hermes plugins   # if you want to use a third-party image_gen backend plugin
```

## Setup per Backend

### FAL (default)

```bash
# Either: direct key
echo "FAL_KEY=your-fal-api-key-here" >> ~/.hermes/.env
pip install fal-client

# Or: paid Nous Portal subscription (no FAL key needed)
hermes model     # pick Nous Portal, follow tool prompts
# This sets image_gen.use_gateway: true
```

The FAL backend is the only one that uses the Nous Tool Gateway path. All other backends require a direct API key.

### Recraft V4 Pro

```bash
echo "RECRAFT_API_KEY=..." >> ~/.hermes/.env
```

### Nano Banana Pro

```bash
echo "NANO_BANANA_API_KEY=..." >> ~/.hermes/.env
```

### GPT Image 2

```bash
echo "OPENAI_API_KEY=..." >> ~/.hermes/.env
```

### xAI grok-imagine

```bash
echo "XAI_API_KEY=..." >> ~/.hermes/.env
```

### Codex OAuth (GPT-5.5 image)

```bash
hermes login codex      # browser OAuth flow against ChatGPT account
```

No API key is required — generations are billed against the ChatGPT subscription. See [Provider Authentication](./provider-runtime.md) for OAuth details.

## Usage

The tool name and surface are identical across backends. The active backend handles the call.

```text
Generate an image of a serene mountain landscape with cherry blossoms
```

```text
Create a portrait of a wise old owl perched on an ancient tree branch
```

## Parameters

| Parameter | Default | Notes |
|-----------|---------|-------|
| `prompt` | *(required)* | Text description |
| `aspect_ratio` | `landscape` | `landscape`, `square`, `portrait` (FAL also accepts raw size presets) |
| `num_images` | 1 | 1-4 |
| `output_format` | `png` | `png`, `jpeg` |
| `seed` | *(random)* | Reproducible runs |

Backends accept additional backend-specific parameters; the tool schema documents what each backend exposes. Unknown parameters are passed through to the backend's API.

### FAL-specific parameters

`num_inference_steps` (1-100, default 50), `guidance_scale` (0.1-20.0, default 4.5). FAL also supports raw size presets `square_hd`, `square`, `portrait_4_3`, `portrait_16_9`, `landscape_4_3`, `landscape_16_9`, and custom sizes up to 2048x2048.

## Automatic Upscaling (FAL only)

The FAL backend automatically 2x-upscales every result with the Clarity Upscaler. Other backends return the model's native resolution and do not chain a second pass — pick a backend whose default resolution already meets your need, or upscale separately.

| Setting | Value |
|---------|-------|
| Upscale Factor | 2x |
| Creativity | 0.35 |
| Resemblance | 0.6 |
| Guidance Scale | 4 |
| Inference Steps | 18 |
| Positive Prompt | `"masterpiece, best quality, highres"` + your original prompt |
| Negative Prompt | `"(worst quality, low quality, normal quality:2)"` |

If upscaling fails (network issue, rate limit), the original resolution image is returned automatically.

## Debugging

```bash
export IMAGE_TOOLS_DEBUG=true
```

Debug logs are saved to `./logs/image_tools_debug_<session_id>.json` with the active backend name, request parameters, timing, and any errors.

## Safety Settings

For backends that expose a safety toggle (currently FAL via `safety_tolerance` and `ENABLE_SAFETY_CHECKER`), Hermes runs with the most permissive setting. Other backends use their provider's default safety filtering. Backend-specific safety knobs are documented at the call site in source.

## Nous Tool Gateway (FAL only)

Paid [Nous Portal](https://portal.nousresearch.com) subscribers can use the FAL backend without a `FAL_KEY`:

```yaml
image_gen:
  provider: fal
  use_gateway: true
```

The gateway routes generation through FAL using your Nous subscription credentials stored in `~/.hermes/auth.json`. When `use_gateway: true`, the gateway is used even if `FAL_KEY` is also set. For setup, run `hermes model` → Nous Portal → follow tool prompts. See [Nous Tool Gateway](/docs/nous-tool-gateway) for the full guide.

The gateway path is only available for the FAL backend. Other backends always go directly to their provider.

## Limitations

- **Backend selection is single-active** — only one image_gen backend is enabled at a time.
- **No image editing** — `image_gen` is text-to-image; inpainting and img2img are backend-specific extensions if exposed at all.
- **URL-based delivery** — most backends return temporary URLs rather than saving files locally.
- **Backend feature parity is not guaranteed** — parameters supported by FAL (e.g. `num_inference_steps`, exact `guidance_scale` ranges) may not exist on other backends.

## Adding a Backend (v0.11.0)

A plugin can register a new image_gen backend. Minimal example:

```python
# ~/.hermes/plugins/my-image-backend/__init__.py
def register(ctx):
    def handler(prompt, aspect_ratio="landscape", num_images=1, **kwargs):
        # call your model, return a list of dicts: [{"url": "...", "width": ..., "height": ...}]
        return [{"url": "https://example.com/img.png", "width": 1024, "height": 1024}]

    ctx.register_image_gen_provider("my-backend", handler)
```

The user then selects it via:

```bash
hermes plugins        # enable my-image-backend
# then in config.yaml:
# image_gen:
#   provider: my-backend
```

See [Plugins](./plugins.md) and the [plugin guide](/docs/guides/build-a-hermes-plugin) for the full plugin contract.
