FROM ghcr.io/astral-sh/uv:python3.11-bookworm-slim

# ffmpeg is needed by send_audio_message (opus/ogg conversion)
RUN apt-get update \
    && apt-get install -y --no-install-recommends ffmpeg \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Install dependencies first for better layer caching
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

COPY main.py whatsapp.py audio.py ./

# Bind to all interfaces inside the container; override via MCP_* env vars
ENV MCP_HOST=0.0.0.0 \
    MCP_PORT=8000 \
    MCP_PATH=/mcp \
    MESSAGES_DB_PATH=/app/store/messages.db \
    WHATSAPP_API_BASE_URL=http://whatsapp-bridge:8080/api

EXPOSE 8000

CMD [".venv/bin/python", "main.py"]
