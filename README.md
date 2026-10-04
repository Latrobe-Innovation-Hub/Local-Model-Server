# Multi-GPU LLM Inference Platform

Docker-based multi-GPU LLM inference platform using vllm and OpenWebUI.

## System Configuration

This system is pre-configured with the following:

| Component | Configuration |
|-----------|---------------|
| OS | Arch Linux (VM) |
| GPU | 8x NVIDIA L40S (46GB each) |
| Driver | NVIDIA 580.159.04 |
| Docker | 29.5.1 |
| NVIDIA Container Toolkit | Installed and configured |
| Total VRAM | 368GB |

## Architecture

```
+------------------+
|    OpenWebUI     |  Port 3000
+------------------+
        |
        +---> vllm (OpenAI-compatible API)  Port 8000
        |
        +---> Ollama (when enabled)         Port 11434
                |
                v
        +------------------+
        |  8x NVIDIA GPUs  |
        +------------------+
```

## Quick Start

### Start Default Stack (vllm + OpenWebUI)

```bash
docker compose up -d
```

### Stop All Services

```bash
docker compose down
```

### Access OpenWebUI

Open http://localhost:3000 in your browser.

## Model Configuration

### vllm Models

Models are configured in `docker-compose.yml` by commenting/uncommenting the `command` block in the vllm service. Only one command block should be active at a time.

**Current default model:** `nvidia/Qwen3.5-397B-A17B-NVFP4`

### Switching vllm Models

1. Open `docker-compose.yml`
2. Comment out the current `command` block
3. Uncomment the desired model's `command` block
4. Restart:

```bash
docker compose down
docker compose up -d
```

The model will download automatically on first startup. Large models may take considerable time to download.

### Available vllm Model Configurations

| Model | Status | Notes |
|-------|--------|-------|
| nvidia/Qwen3.5-397B-A17B-NVFP4 | **Default** | Recommended, NF4 quantization |
| Qwen/Qwen3.5-397B-A17B-FP8 | Available | FP8 quantization with CPU offload |
| MiniMaxAI/MiniMax-M2.7 | Available | Requires `--trust-remote-code` |
| lukealonso/GLM-5.1-NVFP4 | Special | Requires GLM image (see below) |
| deepseek-ai/DeepSeek-V4-Flash | **Broken** | vllm compatibility issue |

### GLM Model Setup

GLM models require a specific vllm image:

1. Edit `docker-compose.yml` line 29-31:
   ```yaml
   image: vllm/vllm-openai:glm51-cu130
   ```

2. Uncomment the GLM command block (lines 113-125)
3. Uncomment `VLLM_USE_DEEP_GEMM=0` in environment variables (line 45)
4. Restart:
   ```bash
   docker compose down
   docker compose up -d
   ```

### Nightly Build

For latest model compatibility, use the nightly vllm image:

1. Edit `docker-compose.yml` line 29-31:
   ```yaml
   image: vllm/vllm-openai:nightly
   ```

2. Restart:
   ```bash
   docker compose down
   docker compose up -d
   ```

## Ollama Configuration

Ollama is available as an alternative inference engine for downloading models from the Ollama model library. It is commented out by default.

### Enable Ollama (Disable vllm)

To download models from the Ollama library, vllm must be disabled:

1. Edit `docker-compose.yml`:
   - Comment out the entire `vllm` service (lines 28-133)
   - Uncomment the `ollama` service (lines 2-26)

2. Update OpenWebUI environment (lines 142-143):
   ```yaml
   - OLLAMA_BASE_URL=http://ollama:11434
   # - OPENAI_API_BASE_URL=http://vllm:8000/v1
   ```

3. Start:
   ```bash
   docker compose up -d ollama open-webui
   ```

### Download Models via OpenWebUI

When Ollama is enabled, models can be downloaded directly from the OpenWebUI interface:

1. Open http://localhost:3000
2. Go to Admin Panel > Settings > Models (older versions: Settings (gear icon) > Models)
3. Click "Pull a model"
5. Enter the model name (e.g., `llama3.2`, `mistral`, `gemma2`)
6. Click Pull

### Recommended Ollama Models for This Hardware

With 368GB of VRAM across 8x L40S, these are the largest strong models that run locally in Ollama. Both fit in GPU memory at the same time.

| Model | Size (Q4) | Notes |
|-------|-----------|-------|
| `qwen3.5:122b` | ~81GB | Largest Qwen3.5 in the Ollama library; text and image input, 256K context |
| `gpt-oss:120b` | ~65GB | OpenAI open-weights model, 128K context |

The vllm default (`Qwen3.5-397B-A17B`) is larger, but Qwen3.5 GGUF files don't currently run in Ollama (separate vision projector files), and the Ollama library offers the 397B model as a cloud tag only. Use vllm for the 397B model.

Pull from the command line (faster to monitor than the web UI):

```bash
docker compose exec ollama ollama pull qwen3.5:122b
docker compose exec ollama ollama pull gpt-oss:120b
docker compose exec ollama ollama list
```

### Settings for a Workshop (several users at once)

In the `ollama` service environment:

```yaml
- OLLAMA_NUM_PARALLEL=4        # requests served at once per model
- OLLAMA_MAX_LOADED_MODELS=2   # keep both models in GPU memory
- OLLAMA_KEEP_ALIVE=4h         # stop models unloading between sessions
```

Create workshop logins in OpenWebUI under **Admin Panel > Users > Add User**, and turn off new sign-ups under **Admin Panel > Settings** while the workshop runs. Remove the accounts afterwards.

### Switch Back to vllm

1. Edit `docker-compose.yml`:
   - Comment out the `ollama` service (lines 2-26)
   - Uncomment the `vllm` service (lines 28-133)

2. Restore OpenWebUI environment (lines 142-143):
   ```yaml
   - OLLAMA_BASE_URL=http://ollama:11434
   - OPENAI_API_BASE_URL=http://vllm:8000/v1
   ```

3. Restart:
   ```bash
   docker compose up -d vllm open-webui
   ```

## Environment Configuration

### HuggingFace Token

1. Copy the sample environment file:
   ```bash
   cp .env.sample .env
   ```

2. Edit `.env` and add your HuggingFace token:
   ```
   HF_TOKEN=hf_your_token_here
   ```

3. Get your token from https://huggingface.co/settings/tokens

## Volume Management

### Directory Structure

| Directory | Purpose |
|-----------|---------|
| `hf_cache/` | vllm HuggingFace model cache |
| `ollama_models/` | Ollama model storage |
| `ollama-data/` | Ollama runtime data |
| `openwebui_data/` | OpenWebUI database and user data |

## Known Issues

### DeepSeek Models

DeepSeek models are currently not runnable due to a vllm bug. The DeepSeek-V4-Flash configuration is provided for reference but will fail to start.

## Recommendations

### Recommended Model: Qwen3.5 NF4

The `nvidia/Qwen3.5-397B-A17B-NVFP4` model is recommended:

- NF4 (Normal Float 4) quantization for optimal memory usage
- Good balance of quality and performance
- Native support in vllm without special flags
- 262K context window support