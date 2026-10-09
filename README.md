# Automated AI YouTube Shorts Pipeline

Self-hosted platform that creates, schedules and publishes YouTube Shorts to multiple channels, using free and open-source AI that runs on your own GPU.

Two automations:

1. **AI story Shorts.** Pick a channel and a topic (or let the AI pick one), click *Create & schedule*. The system writes an original story with a hook, splits it into scenes, generates images, narrates it, adds word-highlighted captions, renders a 1080×1920 video, writes the title/description/hashtags and uploads it to the selected channel at the chosen time.
2. **AI clipping.** Upload a long video you own. The system transcribes it, asks the LLM for the best self-contained moments, cuts them, reframes them to 9:16 by tracking the speaker's face, adds captions and a hook title, and gives you drafts to schedule.

Recurring schedules (e.g. 3 Shorts a day at 09:00, 14:00, 20:00) generate videos ahead of time and publish them automatically.

## Architecture

```
┌──────────────── Laravel dashboard (any small VPS, no GPU) ────────────────┐
│ Channels (OAuth per channel) │ Queue │ Schedules │ Clips │ Logs │ AI settings │
│ Scheduler: plan schedules every 5 min, sync YouTube status every 15 min     │
│ Queue worker: PublishVideo job (resumable upload, channel check, quota)      │
│ MySQL (videos, worker_jobs, logs…)    Media disk (rendered MP4s)            │
└───────────────────────▲───────────────────────────────┬────────────────────┘
          HTTPS, worker │ pulls jobs, sends events,     │ X-Worker-Token
          uploads files │                               ▼
┌───────────────────────┴──── Python worker (GPU machine) ───────────────────┐
│ Step runner with manifest.json (retries resume at the failed step)          │
│ Providers: LLM (Ollama / any OpenAI-compatible) │ Images (ComfyUI SDXL/FLUX) │
│            TTS (Kokoro / Piper) │ STT (faster-whisper)                       │
│ Media: FFmpeg Ken Burns scenes, ASS captions via libass, OpenCV reframing   │
└──────────────────────────────────────────────────────────────────────────────┘
```

The worker **pulls** jobs over HTTPS. The GPU machine can sit at home or in the office behind a normal router, with no open ports or VPN, while the dashboard runs on a cheap VPS. Uploads to YouTube are done by the dashboard, so the worker never sees YouTube credentials.

| Folder | What |
|---|---|
| `web/` | Laravel 13 app: dashboard, OAuth, queue, scheduler, YouTube publisher, worker API |
| `worker/` | Python worker: pipelines, AI provider drivers, FFmpeg rendering, tests |
| `worker/workflows/` | ComfyUI API workflows with `{{PLACEHOLDERS}}` |
| `docker/` | Dockerfiles for both services |
| `docs/` | Google/YouTube setup and design notes |

## Story pipeline steps

`script → visuals → narration → captions → metadata → render`, then upload.

Each step saves its output in the job workspace (`manifest.json`). If a step fails, *Retry* continues from that step; *Regenerate from…* redoes a chosen step and everything after it while keeping earlier results.

## Getting the right video onto the right channel

- Every channel has its own encrypted refresh token (Laravel `encrypted` cast with `APP_KEY`). There is no "current account".
- A video stores `channel_id` from the moment it is created. The publish job loads that channel's token only.
- Before every upload the publisher calls `channels.list(mine=true)` with that token and **blocks the upload** if the returned channel ID differs from the stored one. After upload it checks `snippet.channelId` again.
- The upload job is unique per video and skips anything already uploaded, so retries never create duplicates.

## Quick start (development, no GPU)

Requirements: PHP 8.3+, Composer, MySQL 8 (or SQLite), Python 3.10+, FFmpeg.

```bash
# Dashboard
cd web
cp .env.example .env            # set DB_*, SHORTS_WORKER_TOKEN, GOOGLE_* (see docs/youtube-setup.md)
composer install
php artisan key:generate
php artisan migrate
php artisan shorts:create-admin you@example.com
php artisan serve               # http://localhost:8000
php artisan queue:work --timeout=1800    # second terminal: uploads
php artisan schedule:work                # third terminal: schedules + status sync

# Worker (CPU, placeholder images, for trying the flow)
cd ../worker
python -m venv .venv && source .venv/bin/activate
pip install -e ".[dev,stt,clip]"
cp .env.example .env            # WORKER_TOKEN = SHORTS_WORKER_TOKEN; IMAGE_DRIVER=placeholder
ollama pull qwen2.5:7b-instruct # LLM via Ollama
shorts-worker
```

For narration on CPU without the Kokoro install, set `TTS_DRIVER=piper` and point `PIPER_MODEL` at a Piper `.onnx` voice.

## Production

```bash
cp .env.example .env && cp web/.env.example web/.env && cp worker/.env.example worker/.env
# fill in the three files, then on the VPS:
docker compose up -d
docker compose exec web php artisan key:generate --force   # once, then restart
docker compose exec web php artisan shorts:create-admin you@example.com

# on the GPU machine (NVIDIA driver + NVIDIA Container Toolkit installed):
docker compose --profile gpu up -d ollama worker
docker compose exec ollama ollama pull qwen2.5:7b-instruct
```

Put the dashboard behind HTTPS (Caddy, Nginx or Cloudflare Tunnel). Google OAuth requires an HTTPS redirect URI outside localhost.

### ComfyUI (images)

ComfyUI runs next to the worker, outside Docker:

```bash
git clone https://github.com/comfyanonymous/ComfyUI && cd ComfyUI
pip install -r requirements.txt
# put sd_xl_base_1.0.safetensors in models/checkpoints/
python main.py --listen 127.0.0.1 --port 8188
```

To use another model (FLUX.1-schnell, an SDXL fine-tune, a LoRA), build the workflow in ComfyUI, export it with *Save (API Format)*, replace the prompt/size/seed values with `"{{PROMPT}}"`, `"{{NEGATIVE}}"`, `"{{WIDTH}}"`, `"{{HEIGHT}}"`, `"{{SEED}}"`, `"{{STEPS}}"`, `"{{CFG}}"`, `"{{CHECKPOINT}}"`, and point `COMFYUI_WORKFLOW` at the file.

## Swapping AI providers

Workers read defaults from `worker/.env`. The dashboard's **AI settings** page overrides them per role for new jobs; options are stored encrypted, so API keys for paid services can go there.

- **LLM:** any OpenAI-compatible endpoint, e.g. Ollama, llama.cpp server, vLLM, LM Studio, Groq, OpenRouter or OpenAI. Change `base_url`, `model`, `api_key`.
- **Images:** `comfyui` or `placeholder`.
- **TTS:** `kokoro` (Apache-2.0) or `piper` (MIT).
- **STT:** `faster_whisper` or `none`.

To add a provider, implement the interface in `worker/shorts_worker/providers/base.py` and register it in `providers/registry.py`. Pipelines do not change.

## Hardware

| | Minimum | Recommended |
|---|---|---|
| GPU | RTX 3060 12 GB | RTX 3090 / 4090 24 GB |
| RAM | 32 GB | 64 GB |
| Disk | 500 GB NVMe | 1–2 TB NVMe |
| Dashboard VPS | 2 vCPU, 4 GB RAM | same |

On a 12 GB card, one job runs at a time and models load per step. Expect a few minutes per 40-second story Short; schedules create videos hours ahead (`lead_hours`) so this never delays publishing.

## YouTube API limits (read before going live)

See [docs/youtube-setup.md](docs/youtube-setup.md). In short:

- Default quota is 10,000 units/day per Google Cloud project and an upload costs about 1,600, so about 6 uploads a day across all channels. `YOUTUBE_DAILY_UPLOAD_LIMIT` makes the queue wait for the next quota day instead of failing. Request more quota for more volume.
- Until the API project passes YouTube's audit, uploads are forced to **private**. The dashboard detects this and marks such videos "Private on YouTube".
- While the OAuth app is in *Testing* mode, Google expires refresh tokens after 7 days. Publish the app (and complete verification for the YouTube scopes) for permanent logins.

## Tests

```bash
cd worker && pytest          # pipelines end-to-end with fake providers + real FFmpeg rendering
cd web && php artisan test   # worker API, publishing hand-off, schedule planner
```

## Content and policy notes

- Clipping is only for videos you own or have permission to reuse; the form requires confirmation. URL download (yt-dlp) is off by default because downloading from YouTube conflicts with its Terms of Service. Upload the file instead.
- YouTube does not monetize mass-produced, repetitive content. Vary topics, keep a consistent channel identity, and use *Hold for review* on channels where quality matters.
- *Mark uploads as altered or synthetic content* is on by default per channel and sets YouTube's AI-disclosure flag when the API client supports it.
