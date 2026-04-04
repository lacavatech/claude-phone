<p align="center">
  <img src="assets/logo.png" alt="Claude Phone" width="200">
</p>

# Claude Phone

Voice interface for Claude Code via SIP. Works with 3CX, FreePBX, Asterisk, and any standard SIP PBX. Call your AI, and your AI can call you.

## What is this?

Claude Phone gives your Claude Code installation a phone number. You can:

- **Inbound**: Call an extension and talk to Claude - run commands, check status, ask questions
- **Outbound**: Your server can call YOU with alerts, then have a conversation about what to do

## Prerequisites

| Requirement | Where to Get It | Notes |
|-------------|-----------------|-------|
| **SIP PBX** | 3CX, FreePBX, Asterisk, or any SIP-compatible PBX | See [PBX Setup](#pbx-setup) below |
| **ElevenLabs API Key** | [elevenlabs.io](https://elevenlabs.io/) | For text-to-speech |
| **OpenAI API Key** | [platform.openai.com](https://platform.openai.com/) | For Whisper speech-to-text |
| **Claude Code CLI** | [claude.ai/code](https://claude.ai/code) | Requires Claude Max subscription |

## Platform Support

| Platform | Status |
|----------|--------|
| **macOS** | Fully supported |
| **Linux** | Fully supported (including Raspberry Pi) |
| **Windows** | Not supported (may work with WSL) |

## Quick Start

### 1. Install

```bash
curl -sSL https://raw.githubusercontent.com/lacavatech/claude-phone/main/install.sh | bash
```

The installer will:
- Check for Node.js 18+, Docker, and git (offers to install if missing)
- Clone the repository to `~/.claude-phone-cli`
- Install dependencies
- Create the `claude-phone` command

### 2. Setup

```bash
claude-phone setup
```

The setup wizard asks what you're installing:

| Type | Use Case | What It Configures |
|------|----------|-------------------|
| **Voice Server** | Pi or dedicated voice box | Docker containers, connects to remote API server |
| **API Server** | Mac/Linux with Claude Code | Just the Claude API wrapper |
| **Both** | All-in-one single machine | Everything on one box |

### 3. Start

```bash
claude-phone start
```

## Deployment Modes

### All-in-One (Single Machine)

Best for: Mac or Linux server that's always on and has Claude Code installed.

```
┌─────────────────────────────────────────────────────────────┐
│  Your Phone                                                  │
│      │                                                       │
│      ↓ Call extension 9000                                  │
│  ┌─────────────┐                                            │
│  │     PBX     │  ← 3CX / FreePBX / Asterisk                │
│  └──────┬──────┘                                            │
│         │                                                    │
│         ↓                                                    │
│  ┌─────────────────────────────────────────────┐           │
│  │     Single Server (Mac/Linux)                │           │
│  │  ┌───────────┐    ┌───────────────────┐    │           │
│  │  │ voice-app │ ←→ │ claude-api-server │    │           │
│  │  │ (Docker)  │    │ (Claude Code CLI) │    │           │
│  │  └───────────┘    └───────────────────┘    │           │
│  └─────────────────────────────────────────────┘           │
└─────────────────────────────────────────────────────────────┘
```

**Setup:**
```bash
claude-phone setup    # Select "Both"
claude-phone start    # Launches Docker + API server
```

### Split Mode (Pi + API Server)

Best for: Dedicated Pi for voice services, Claude running on your main machine.

```
┌─────────────────────────────────────────────────────────────┐
│  Your Phone                                                  │
│      │                                                       │
│      ↓ Call extension 9000                                  │
│  ┌─────────────┐                                            │
│  │     PBX     │  ← 3CX / FreePBX / Asterisk                │
│  └──────┬──────┘                                            │
│         │                                                    │
│         ↓                                                    │
│  ┌─────────────┐         ┌─────────────────────┐           │
│  │ Raspberry Pi │   ←→   │ Mac/Linux with      │           │
│  │ (voice-app)  │  HTTP  │ Claude Code CLI     │           │
│  └─────────────┘         │ (claude-api-server) │           │
│                          └─────────────────────┘           │
└─────────────────────────────────────────────────────────────┘
```

**On your Pi (Voice Server):**
```bash
claude-phone setup    # Select "Voice Server", enter API server IP when prompted
claude-phone start    # Launches Docker containers
```

**On your Mac/Linux (API Server):**
```bash
claude-phone api-server    # Starts Claude API wrapper on port 3333
```

Note: On the API server machine, you don't need to run `claude-phone setup` first - the `api-server` command works standalone.

## PBX Setup

Claude Phone registers as a SIP extension on your PBX. It works with any SIP-compatible system — the main differences are in how you create the extension and what credentials it uses.

### 3CX

1. Log into your 3CX Admin panel
2. Go to **Extensions** → **Add Extension**
3. Note the **Extension number**, **Auth ID**, and **Password** from the extension settings
4. During `claude-phone setup`, enter:
   - **SIP domain**: your 3CX FQDN (e.g., `mycompany.3cx.us`)
   - **SIP registrar IP**: your 3CX server LAN IP
   - **SIP auth username**: the **Auth ID** (it differs from the extension number in 3CX)
   - **Password**: the extension password

For outbound PSTN calls, set `OUTBOUND_PSTN_PREFIX=9` in your `.env` (3CX routes external calls with a leading 9).

### FreePBX / Asterisk

1. Log into your FreePBX admin panel
2. Go to **Applications** → **Extensions** → **Add Extension** (type: **PJSIP** recommended)
3. Set an extension number and password — note them both
4. During `claude-phone setup`, enter:
   - **SIP domain**: your PBX hostname or IP (e.g., `192.168.1.100`)
   - **SIP registrar IP**: same as the PBX IP
   - **SIP auth username**: leave blank — FreePBX uses the extension number as the auth username
   - **Password**: the extension password you set

For outbound PSTN calls, leave `OUTBOUND_PSTN_PREFIX` unset — FreePBX outbound routes handle PSTN routing directly via trunks.

### Other SIP PBXes (Kamailio, Lync, etc.)

Claude Phone uses standard SIP REGISTER. You need:
- An extension/user number (used in `From`/`To`/`Contact` SIP headers)
- A password for digest authentication
- Optionally, a separate auth username if your PBX separates identity from auth credentials (set `SIP_USERNAME` in `.env`)

## CLI Commands

| Command | Description |
|---------|-------------|
| `claude-phone setup` | Interactive configuration wizard |
| `claude-phone start` | Start services based on installation type |
| `claude-phone stop` | Stop all services |
| `claude-phone status` | Show service status |
| `claude-phone doctor` | Health check for dependencies and services |
| `claude-phone api-server [--port N]` | Start API server standalone (default: 3333) |
| `claude-phone device add` | Add a new device/extension |
| `claude-phone device list` | List configured devices |
| `claude-phone device remove <name>` | Remove a device |
| `claude-phone logs [service]` | Tail logs (voice-app, drachtio, freeswitch) |
| `claude-phone config show` | Display configuration (secrets redacted) |
| `claude-phone config path` | Show config file location |
| `claude-phone config reset` | Reset configuration |
| `claude-phone backup` | Create configuration backup |
| `claude-phone restore` | Restore from backup |
| `claude-phone update` | Update Claude Phone |
| `claude-phone uninstall` | Complete removal |

## Device Personalities

Each SIP extension can have its own identity with a unique name, voice, and personality prompt:

```bash
claude-phone device add
```

Example devices:
- **Morpheus** (ext 9000) - General assistant
- **Cephanie** (ext 9002) - Storage monitoring bot

## API Endpoints

The voice-app exposes these endpoints on port 3000:

| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `/api/outbound-call` | Initiate an outbound call |
| GET | `/api/call/:callId` | Get call status |
| GET | `/api/calls` | List active calls |
| POST | `/api/query` | Query a device programmatically |
| GET | `/api/devices` | List configured devices |

See [Outbound API Reference](voice-app/README-OUTBOUND.md) for details.

## Troubleshooting

### Quick Diagnostics

```bash
claude-phone doctor    # Automated health checks
claude-phone status    # Service status
claude-phone logs      # View logs
```

### Common Issues

| Problem | Likely Cause | Solution |
|---------|--------------|----------|
| Calls connect but no audio | Wrong external IP | Re-run `claude-phone setup`, verify LAN IP |
| Extension not registering (3CX) | Wrong Auth ID | 3CX uses a separate Auth ID — set `SIP_USERNAME` to the value shown in the extension settings, not the extension number |
| Extension not registering (FreePBX) | Username mismatch | Leave `SIP_USERNAME` unset — FreePBX uses the extension number as the auth username |
| "Sorry, something went wrong" | API server unreachable | Check `claude-phone status` |
| Port conflict on startup | Another SIP process on port 5060 | Setup auto-detects this; re-run setup |
| Outbound calls fail (no audio path) | Missing `SIP_TRUNK_HOST` | Set `SIP_TRUNK_HOST` in `.env` to your PBX/trunk IP |
| Outbound calls fail with 404 | Wrong PSTN prefix | 3CX needs `OUTBOUND_PSTN_PREFIX=9`; FreePBX usually needs it empty |

See [Troubleshooting Guide](docs/TROUBLESHOOTING.md) for more.

## Configuration

Configuration is stored in `~/.claude-phone/config.json` with restricted permissions (chmod 600).

```bash
claude-phone config show    # View config (secrets redacted)
claude-phone config path    # Show file location
```

## Development

```bash
# Run tests
npm test

# Lint
npm run lint
npm run lint:fix
```

## Documentation

- [CLI Reference](cli/README.md) - Detailed CLI documentation
- [Troubleshooting](docs/TROUBLESHOOTING.md) - Common issues and solutions
- [Outbound API](voice-app/README-OUTBOUND.md) - Outbound calling API reference
- [Deployment](voice-app/DEPLOYMENT.md) - Production deployment guide
- [Claude Code Skill](docs/CLAUDE-CODE-SKILL.md) - Build a "call me" skill for Claude Code

## License

MIT
