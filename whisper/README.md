# whisper

Self-hosted speech-to-text ([Speaches](https://github.com/speaches-ai/speaches), faster-whisper) with an OpenAI-compatible API. Used by LibreChat's mic button, since Firefox has no built-in speech recognition.

## Allocations

- Port: `10030` -> container `8000` (next free slot after librechat's `10020`)
- UID/GID: `10006:1000` (dedicated stack UID; `10005` librechat, `10006` = this). GID `1000` is the image's own group, not `nogroup`: the image's Python lives in a `750` home dir, so other groups cannot start it
- Image (pinned): `ghcr.io/speaches-ai/speaches:0.8.3-cuda`
- Limits: `3 cpus / 8g` (inference runs on the GPU, ~0.5s for a 15s clip with `small` on an RTX 4070 using ~900 MiB VRAM)
- Data: `./data` (Hugging Face model cache), must be owned by `10006:1000`

## First-time setup

0. Needs an NVIDIA GPU with `nvidia-container-toolkit` and a CDI spec (`/var/run/cdi/nvidia.yaml`). The compose file uses `devices: nvidia.com/gpu=all`.
1. `cp .env.example .env` (no secrets).
2. `mkdir -p data/huggingface/hub && sudo chown -R 10006:1000 data`
3. Deploy via Arcane and wait for the healthcheck to pass.
4. Download the model once (it persists in `./data`): `curl -X POST http://localhost:10030/v1/models/Systran/faster-whisper-small`
5. Test: `curl http://localhost:10030/v1/audio/transcriptions -F file=@clip.wav -F model=Systran/faster-whisper-small`

## LibreChat

`stacks/librechat/librechat.yaml` has a `speech:` section pointing at `http://host.docker.internal:10030/v1/audio/transcriptions`. Redeploy librechat after changing it.

## Notes

- No auth: the API is open to anything that can reach port `10030`. Keep it LAN-only.
- Microphone access in the browser needs `localhost` or HTTPS; plain `http://<lan-ip>` will be blocked.

## Deviations

- GID `1000` instead of `nogroup` (see Allocations).
- The model is downloaded by a one-time API call, not at startup (`PRELOAD_MODELS` is not honored in this version).
