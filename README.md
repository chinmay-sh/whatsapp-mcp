# WhatsApp MCP Server

This is a Model Context Protocol (MCP) server for WhatsApp.

With this you can search and read your personal Whatsapp messages (including images, videos, documents, and audio messages), search your contacts and send messages to either individuals or groups. You can also send media files including images, videos, documents, and audio messages.

It connects to your **personal WhatsApp account** directly via the Whatsapp web multidevice API (using the [whatsmeow](https://github.com/tulir/whatsmeow) library). All your messages are stored locally in a SQLite database and only sent to an LLM (such as Claude) when the agent accesses them through tools (which you control).

Here's an example of what you can do when it's connected to Claude.

![WhatsApp MCP](./example-use.png)

> To get updates on this and other projects I work on [enter your email here](https://docs.google.com/forms/d/1rTF9wMBTN0vPfzWuQa2BjfGKdKIpTbyeKxhPMcEzgyI/preview)

> *Caution:* as with many MCP servers, the WhatsApp MCP is subject to [the lethal trifecta](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/). This means that project injection could lead to private data exfiltration.

## Installation

### Option A: Docker Compose (recommended)

Runs both the WhatsApp bridge and the MCP server as containers with one command — no Go, Python or FFmpeg setup needed.

1. **Clone this repository**

   ```bash
   git clone https://github.com/lharries/whatsapp-mcp.git
   cd whatsapp-mcp
   ```

2. **Start the stack**

   ```bash
   docker compose up -d --build
   ```

3. **Authenticate WhatsApp**

   The bridge prints a QR code on first start. Display it with:

   ```bash
   docker compose logs -f whatsapp-bridge
   ```

   Scan the QR code with your WhatsApp mobile app (Settings → Linked Devices → Link a Device). The session and message database persist in the `whatsapp-store` Docker volume, so you only scan once.

4. **Connect your MCP client**

   Point any Streamable HTTP-capable client at the URL entry:

   ```json
   {
     "mcpServers": {
       "whatsapp": {
         "url": "http://localhost:8000/mcp"
       }
     }
   }
   ```

Notes:

- The MCP server listens on `0.0.0.0:8000` inside the container (host port configurable under `ports` in [docker-compose.yml](docker-compose.yml)).
- Both containers mount the same `whatsapp-store` volume at `/app/store` — the server reads the bridge's SQLite DB directly and exchanged media paths resolve in both containers. This means `send_file`/`send_audio_message` media paths must reference files inside that volume (e.g. `/app/store/...`); host-local paths are not visible to the containers. To send media from your host, mount it into the server container first.
- Useful commands: `docker compose logs -f` (logs), `docker compose restart` (restart), `docker compose down` (stop).

#### Docker configuration reference

Environment variables understood by the `whatsapp-mcp-server` container (set them under `environment` in [docker-compose.yml](docker-compose.yml)):

| Variable | Default | Purpose |
| --- | --- | --- |
| `MCP_HOST` | `0.0.0.0` | Bind address of the MCP server (keep `0.0.0.0` in Docker) |
| `MCP_PORT` | `8000` | Port the MCP server listens on inside the container |
| `MCP_PATH` | `/mcp` | HTTP path serving the MCP endpoint |
| `MESSAGES_DB_PATH` | `/app/store/messages.db` | Path to the bridge's SQLite message DB |
| `WHATSAPP_API_BASE_URL` | `http://whatsapp-bridge:8080/api` | Base URL of the bridge's REST API |

The published host port is set under `ports` (default `"8000:8000"`). To serve MCP on a different host port, e.g. `9000`, change it to `"9000:8000"` and point your client at `http://localhost:9000/mcp`.

#### Sending media from the host

`send_file` and `send_audio_message` pass filesystem paths to the bridge, so the files must be visible inside the containers. Mount a host directory into the server container, e.g. add to `whatsapp-mcp-server` in [docker-compose.yml](docker-compose.yml):

```yaml
    volumes:
      - whatsapp-store:/app/store
      - ./media:/app/media
```

Then ask the agent to send `/app/media/photo.jpg`.

#### Re-authenticating

WhatsApp sessions expire after ~20 days. When that happens the bridge logs a new QR code — view it with `docker compose logs -f whatsapp-bridge` and scan it. If you want to start fresh (unlink the device and wipe all history), remove the volume:

```bash
docker compose down
docker volume rm whatsapp-mcp_whatsapp-store
docker compose up -d
```

#### Security

The MCP endpoint has no authentication. Only publish the port (`ports`) on trusted networks; if you don't need host access, remove the `ports` entry and reach the server via another container on the same Docker network, or put it behind an authenticating reverse proxy.

### Option B: Local setup

#### Prerequisites

- Go
- Python 3.6+
- Anthropic Claude Desktop app (or Cursor)
- UV (Python package manager), install with `curl -LsSf https://astral.sh/uv/install.sh | sh`
- FFmpeg (_optional_) - Only needed for audio messages. If you want to send audio files as playable WhatsApp voice messages, they must be in `.ogg` Opus format. With FFmpeg installed, the MCP server will automatically convert non-Opus audio files. Without FFmpeg, you can still send raw audio files using the `send_file` tool.

#### Steps

1. **Clone this repository**

   ```bash
   git clone https://github.com/lharries/whatsapp-mcp.git
   cd whatsapp-mcp
   ```

2. **Run the WhatsApp bridge**

   Navigate to the whatsapp-bridge directory and run the Go application:

   ```bash
   cd whatsapp-bridge
   go run main.go
   ```

   The first time you run it, you will be prompted to scan a QR code. Scan the QR code with your WhatsApp mobile app to authenticate.

   After approximately 20 days, you will might need to re-authenticate.

3. **Start the MCP server**

   The MCP server supports two modes:

   - **HTTP mode (default)** — serves Streamable HTTP at `http://127.0.0.1:8000/mcp`, so multiple clients can connect to one shared server:

     ```bash
     cd whatsapp-mcp-server
     uv run main.py
     ```

     Override the bind address/port/endpoint with environment variables (`MCP_HOST`, `MCP_PORT`, `MCP_PATH`), or pin defaults in [main.py](whatsapp-mcp-server/main.py).

   - **stdio mode** — the original behavior, one server process per client (what Claude Desktop and most local clients expect):

     ```bash
     cd whatsapp-mcp-server
     MCP_TRANSPORT=stdio uv run main.py
     ```

4. **Connect your client**

   **HTTP mode** — use a URL entry (supported by Cursor and other Streamable HTTP-capable clients):

   ```json
   {
     "mcpServers": {
       "whatsapp": {
         "url": "http://127.0.0.1:8000/mcp"
       }
     }
   }
   ```

   **stdio mode** — copy the below json with the appropriate {{PATH}} values:

   ```json
   {
     "mcpServers": {
       "server-name": {
         "command": "{{PATH_TO_UV}}", // Run `which uv` and place the output here
         "args": [
           "--directory",
           "{{PATH_TO_SRC}}/whatsapp-mcp/whatsapp-mcp-server", // cd into the repo, run `pwd` and enter the output here + "/whatsapp-mcp-server"
           "run",
           "main.py",
           "--transport",
           "stdio"
         ]
       }
     }
   }
   ```

   For **Claude**, save this as `claude_desktop_config.json` in your Claude Desktop configuration directory at:

   ```
   ~/Library/Application Support/Claude/claude_desktop_config.json
   ```

   For **Cursor**, save this as `mcp.json` in your Cursor configuration directory at:

   ```
   ~/.cursor/mcp.json
   ```

5. **Restart Claude Desktop / Cursor**

   Open Claude Desktop and you should now see WhatsApp as an available integration.

   Or restart Cursor.

### Windows Compatibility

If you're running this project on Windows, be aware that `go-sqlite3` requires **CGO to be enabled** in order to compile and work properly. By default, **CGO is disabled on Windows**, so you need to explicitly enable it and have a C compiler installed.

> **Tip:** The [Docker Compose](#option-a-docker-compose-recommended) setup avoids this entirely — the bridge builds inside a Linux container with CGO already configured.

#### Steps to get it working:

1. **Install a C compiler**  
   We recommend using [MSYS2](https://www.msys2.org/) to install a C compiler for Windows. After installing MSYS2, make sure to add the `ucrt64\bin` folder to your `PATH`.  
   → A step-by-step guide is available [here](https://code.visualstudio.com/docs/cpp/config-mingw).

2. **Enable CGO and run the app**

   ```bash
   cd whatsapp-bridge
   go env -w CGO_ENABLED=1
   go run main.go
   ```

Without this setup, you'll likely run into errors like:

> `Binary was compiled with 'CGO_ENABLED=0', go-sqlite3 requires cgo to work.`

## Architecture Overview

This application consists of two main components:

1. **Go WhatsApp Bridge** (`whatsapp-bridge/`): A Go application that connects to WhatsApp's web API, handles authentication via QR code, and stores message history in SQLite. It serves as the bridge between WhatsApp and the MCP server.

2. **Python MCP Server** (`whatsapp-mcp-server/`): A Python server implementing the Model Context Protocol (MCP), which provides standardized tools for Claude to interact with WhatsApp data and send/receive messages.

### Data Storage

- All message history is stored in a SQLite database within the `whatsapp-bridge/store/` directory
- The database maintains tables for chats and messages
- Messages are indexed for efficient searching and retrieval

## Usage

Once connected, you can interact with your WhatsApp contacts through Claude, leveraging Claude's AI capabilities in your WhatsApp conversations.

### MCP Tools

Claude can access the following tools to interact with WhatsApp:

- **search_contacts**: Search for contacts by name or phone number
- **list_messages**: Retrieve messages with optional filters and context
- **list_chats**: List available chats with metadata
- **get_chat**: Get information about a specific chat
- **get_direct_chat_by_contact**: Find a direct chat with a specific contact
- **get_contact_chats**: List all chats involving a specific contact
- **get_last_interaction**: Get the most recent message with a contact
- **get_message_context**: Retrieve context around a specific message
- **send_message**: Send a WhatsApp message to a specified phone number or group JID
- **send_file**: Send a file (image, video, raw audio, document) to a specified recipient
- **send_audio_message**: Send an audio file as a WhatsApp voice message (requires the file to be an .ogg opus file or ffmpeg must be installed)
- **download_media**: Download media from a WhatsApp message and get the local file path

### Media Handling Features

The MCP server supports both sending and receiving various media types:

#### Media Sending

You can send various media types to your WhatsApp contacts:

- **Images, Videos, Documents**: Use the `send_file` tool to share any supported media type.
- **Voice Messages**: Use the `send_audio_message` tool to send audio files as playable WhatsApp voice messages.
  - For optimal compatibility, audio files should be in `.ogg` Opus format.
  - With FFmpeg installed, the system will automatically convert other audio formats (MP3, WAV, etc.) to the required format.
  - Without FFmpeg, you can still send raw audio files using the `send_file` tool, but they won't appear as playable voice messages.

#### Media Downloading

By default, just the metadata of the media is stored in the local database. The message will indicate that media was sent. To access this media you need to use the download_media tool which takes the `message_id` and `chat_jid` (which are shown when printing messages containing the meda), this downloads the media and then returns the file path which can be then opened or passed to another tool.

## Technical Details

1. Claude sends requests to the Python MCP server
2. The MCP server queries the Go bridge for WhatsApp data or directly to the SQLite database
3. The Go accesses the WhatsApp API and keeps the SQLite database up to date
4. Data flows back through the chain to Claude
5. When sending messages, the request flows from Claude through the MCP server to the Go bridge and to WhatsApp

## Troubleshooting

- If you encounter permission issues when running uv, you may need to add it to your PATH or use the full path to the executable.
- Make sure both the Go application and the Python server are running for the integration to work properly.

### Docker

- **Bridge keeps restarting / no QR code**: check `docker compose logs whatsapp-bridge`. The bridge only starts its REST API after WhatsApp authentication succeeds; the MCP server works independently of it.
- **`client outdated (405)` in bridge logs**: your image was built from an old pinned `whatsmeow` version. Rebuild with `docker compose build --no-cache whatsapp-bridge` after pulling the latest repo.
- **MCP connection refused**: verify the stack is up with `docker compose ps` and that the host port mapping (default `8000:8000`) doesn't clash with another service.
- **Tools work but sends fail**: the server can't reach the bridge. Confirm `WHATSAPP_API_BASE_URL` points at `http://whatsapp-bridge:8080/api` and the bridge container is running and authenticated.

### Authentication Issues

- **QR Code Not Displaying**: If the QR code doesn't appear, try restarting the authentication script. If issues persist, check if your terminal supports displaying QR codes.
- **WhatsApp Already Logged In**: If your session is already active, the Go bridge will automatically reconnect without showing a QR code.
- **Device Limit Reached**: WhatsApp limits the number of linked devices. If you reach this limit, you'll need to remove an existing device from WhatsApp on your phone (Settings > Linked Devices).
- **No Messages Loading**: After initial authentication, it can take several minutes for your message history to load, especially if you have many chats.
- **WhatsApp Out of Sync**: If your WhatsApp messages get out of sync with the bridge, delete both database files (`whatsapp-bridge/store/messages.db` and `whatsapp-bridge/store/whatsapp.db`) and restart the bridge to re-authenticate.

For additional Claude Desktop integration troubleshooting, see the [MCP documentation](https://modelcontextprotocol.io/quickstart/server#claude-for-desktop-integration-issues). The documentation includes helpful tips for checking logs and resolving common issues.
