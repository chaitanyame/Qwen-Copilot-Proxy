# Qwen-Copilot-Proxy

![Python 3.8+](https://img.shields.io/badge/python-3.8%2B-blue)
![FastAPI](https://img.shields.io/badge/FastAPI-0.68%2B-009688)
![License](https://img.shields.io/badge/license-MIT-green)
![Version](https://img.shields.io/badge/version-1.1.0-orange)
[![GitHub](https://img.shields.io/badge/GitHub-chaitanyame/Qwen--Copilot--Proxy-181717?logo=github)](https://github.com/chaitanyame/Qwen-Copilot-Proxy)

![Qwen Copilot Proxy Architecture](assets/qwen-copilot-proxy.png)

A robust proxy server that enables Qwen Code models to work with GitHub Copilot Chat by mimicking the Ollama API interface.

## Why This Exists

GitHub Copilot Chat in VS Code supports connecting to model providers through Ollama's API. Qwen's code models (qwen3-coder-plus, vision-model) are accessible via their own API but not natively through Ollama. This proxy bridges the gap — it sits between Copilot Chat and Qwen's API, translating Ollama-format requests into Qwen API calls while handling OAuth authentication, token refresh, and streaming transparently.

```
GitHub Copilot Chat ↔️ localhost:11434 (Ollama endpoint)
                              ↓
                    Qwen-Copilot-Proxy Server
                              ↓
                      Qwen API (OAuth2)
```

## Supported Models

| Model | ID | Capabilities | Best For |
|-------|-----|-------------|----------|
| **qwen3-coder-plus** | `qwen3-coder-plus-2025-09-23` | Tools, Code generation | Pure coding tasks, refactoring, debugging |
| **vision-model** | `qwen3-vl-plus-2025-09-23` | Tools, Vision, Code generation | Multimodal tasks: screenshots → code, UI analysis, visual coding |

> 💡 Use `qwen3-coder-plus` for everyday coding and switch to `vision-model` when you need to work with images or screenshots.

## Quick Start

### Prerequisites

1. **Qwen Account**: Install the [Qwen-code CLI](https://github.com/QwenLM/qwen-code) and authenticate with OAuth to obtain credentials.
2. **Qwen OAuth Credentials**: Stored at `~/.qwen/oauth_creds.json`
3. **Python 3.8+**: Required to run the proxy server
4. **GitHub Copilot Chat Extension**: Required for VS Code integration

### Installation

```bash
# Create and activate a virtual environment (recommended)
python -m venv qwen-proxy-venv
source qwen-proxy-venv/bin/activate  # On Windows: qwen-proxy-venv\Scripts\activate

# Install Python dependencies
pip install -r requirements.txt
```

### Running the Proxy

```bash
python proxy_server.py
```

The server starts on **`http://localhost:11434`** (the same port Ollama uses — make sure Ollama is not already running on that port).

### Configure GitHub Copilot Chat

1. Open VS Code → GitHub Copilot Chat
2. Click the current model name → **Manage Models...**
3. Choose **ollama** as the provider
4. Select your preferred model:
   - **`qwen3-coder-plus`** for code generation
   - **`vision-model`** for code + vision tasks

## Features

### 🔧 Robust OAuth Authentication
- Automatic token refresh with 30-second buffer
- Comprehensive error handling for credential issues
- Secure credential storage and validation

### 🔄 Retry Logic & Error Recovery
- Automatic retries for network failures (up to 3 attempts)
- Intelligent token refresh on 401 errors (auto-retries with refreshed credentials)
- Graceful degradation and detailed error reporting

### 📊 Enhanced Monitoring
- Health check endpoint at `/health`
- Detailed status reporting
- Real-time authentication status

### 🎯 Model Selection
- Both `qwen3-coder-plus` and `vision-model` available
- Proper capability reporting (tools, vision)
- Model-specific context window handling (32K tokens)

## API Endpoints

### Ollama-Compatible Endpoints
- `GET /api/tags` — Returns list of available Qwen models in Ollama format
- `GET /api/list` — Alias for `/api/tags`
- `POST /api/show` — Retrieves detailed model information including capabilities
- `POST /v1/chat/completions` — OpenAI-compatible chat completions (streaming + non-streaming)

### Monitoring Endpoints
- `GET /` — Server status and available models
- `GET /health` — Health check with authentication status
- `GET /api/version` — Returns Ollama version 0.6.4 for compatibility
- `GET /version` — Returns proxy version and supported models

## Running with Docker

Containerize the proxy for a clean, isolated environment:

```dockerfile
# Dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY proxy_server.py VERSION .
EXPOSE 11434
CMD ["uvicorn", "proxy_server:app", "--host", "0.0.0.0", "--port", "11434"]
```

**Build and run:**

```bash
docker build -t qwen-copilot-proxy .
docker run -d \
  --name qwen-proxy \
  -p 11434:11434 \
  -v ~/.qwen:/root/.qwen:ro \
  qwen-copilot-proxy
```

> ⚠️ The `-v` mount makes your Qwen OAuth credentials available inside the container. Ensure the credentials file exists at `~/.qwen/oauth_creds.json` before starting.

## Running as a Systemd Service

For a persistent setup that auto-starts on boot:

```ini
# /etc/systemd/system/qwen-copilot-proxy.service
[Unit]
Description=Qwen Copilot Proxy
After=network.target

[Service]
Type=simple
User=your-username
WorkingDirectory=/path/to/Qwen-Copilot-Proxy
ExecStart=/path/to/qwen-proxy-venv/bin/uvicorn proxy_server:app --host localhost --port 11434
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now qwen-copilot-proxy
sudo systemctl status qwen-copilot-proxy
```

## Custom Port

To run the proxy on a different port (e.g., if Ollama is already using 11434):

```bash
uvicorn proxy_server:app --host localhost --port 11435 --reload
```

Then configure Copilot Chat's Ollama provider to point to `http://localhost:11435`.

## Troubleshooting

### Common Issues

| Symptom | Likely Cause | Solution |
|---------|-------------|----------|
| "Failed to authenticate with Qwen" | Missing or invalid credentials | Verify `~/.qwen/oauth_creds.json` exists and contains `access_token`, `refresh_token`, `token_type`, `expiry_date` |
| "Unsupported model" error | Model name mismatch | Use exactly `qwen3-coder-plus` or `vision-model` |
| Shows actual Ollama models instead of Qwen | Another service on port 11434 | Stop Ollama: `sudo systemctl stop ollama` or use a custom port |
| Network errors / timeouts | Internet connectivity | The proxy auto-retries up to 3 times; check your connection |
| "Port 11434 already in use" | Conflicting service | `lsof -i :11434` to find the culprit, then stop it or change port |

### Health Check

```bash
curl http://localhost:11434/health
```

Expected response:
```json
{
  "status": "healthy",
  "authenticated": true,
  "models_supported": ["qwen3-coder-plus", "vision-model"]
}
```

### Debug Mode

Console output shows authentication status, available models, and any startup errors. Run in the foreground for full logs:

```bash
uvicorn proxy_server:app --host localhost --port 11434 --log-level debug
```

## Performance Tips

1. **Network Stability**: The proxy handles network issues gracefully, but a stable connection provides the best experience
2. **Token Management**: Tokens auto-refresh; restart the proxy occasionally for a fresh authentication cycle
3. **Model Selection**: Use `qwen3-coder-plus` for pure coding and `vision-model` only when you need vision capabilities — the coder model is faster and more responsive for code tasks
4. **Session Persistence**: Run as a systemd service (see above) so the proxy stays available across logins

## Version

**Current Version**: 1.1.0 (2025-11-28)

- Check version: `curl http://localhost:11434/version`
- View changelog: [CHANGELOG.md](CHANGELOG.md)
- Report issues: [GitHub Issues](https://github.com/chaitanyame/Qwen-Copilot-Proxy/issues)

## License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.
