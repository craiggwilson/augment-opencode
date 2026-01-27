# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Auggie Wrapper is an OpenAI-compatible API proxy that routes requests to Claude models via the Augment Code SDK. It enables [OpenCode](https://opencode.ai) and other OpenAI-compatible clients to use Claude models through Augment Code's infrastructure.

## Architecture

```
OpenCode/Client → HTTP Request → server.js → Auggie SDK → Augment Code API → Claude Model
```

### Key Components

- **server.js**: Single-file Node.js HTTP server (ES modules)
  - Exposes OpenAI-compatible endpoints (`/v1/chat/completions`, `/v1/models`)
  - Maintains per-model Auggie SDK client cache
  - Supports streaming via Server-Sent Events
  - Uses authentication from `~/.augment/session.json`

- **setup.sh**: Automated setup script for dependencies and OpenCode configuration

## Available Models

| OpenCode Model ID | Auggie Model ID | Description |
|-------------------|-----------------|-------------|
| `claude-opus-4.5` | `opus4.5` | Default, most capable |
| `claude-sonnet-4.5` | `sonnet4.5` | Balanced performance |
| `claude-sonnet-4` | `sonnet4` | Previous generation |
| `claude-haiku-4.5` | `haiku4.5` | Fastest, lightweight |

## Common Commands

```bash
# Install dependencies
npm install

# Start server (production)
npm start

# Start server (development with auto-reload)
npm run dev

# Run setup script
./setup.sh

# Test API
curl http://localhost:8765/v1/models
curl -X POST http://localhost:8765/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "claude-opus-4.5", "messages": [{"role": "user", "content": "Hello"}]}'
```

## Code Patterns

### Model Mapping

Models are mapped in `MODEL_MAP` constant:
```javascript
const MODEL_MAP = {
  'claude-opus-4.5': { auggie: 'opus4.5', name: 'Claude Opus 4.5', context: 200000, output: 32000 },
  // ...
};
```

### SDK Client Caching

Clients are cached per Auggie model ID in `auggieClients` object to avoid re-initialization:
```javascript
async function getAuggieClient(modelId) {
  const modelConfig = MODEL_MAP[modelId] || MODEL_MAP[DEFAULT_MODEL];
  const auggieModel = modelConfig.auggie;
  if (auggieClients[auggieModel]) return auggieClients[auggieModel];
  // ... create and cache new client
}
```

### Authentication

Session credentials are loaded from `~/.augment/session.json`:
- `accessToken`: API authentication token
- `tenantURL`: Augment API endpoint

## Configuration

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `PORT` | `8765` | Server port |

### OpenCode Configuration

OpenCode config location: `~/.config/opencode/opencode.json`

Provider configuration uses `@ai-sdk/openai-compatible` npm package with `baseURL` pointing to local server.

## Dependencies

- `@augmentcode/auggie-sdk`: Augment Code SDK for API access
- Node.js 22+ required (ES modules, native fetch)

## File Structure

```
auggie-wrapper/
├── server.js          # Main proxy server
├── setup.sh           # Automated setup script
├── package.json       # Node.js configuration
├── README.md          # User documentation
└── CLAUDE.md          # This file
```

## Testing Changes

After modifying server.js:
1. Restart the server: `npm start`
2. Test models endpoint: `curl http://localhost:8765/v1/models`
3. Test chat completion with each model to verify routing

## Known Limitations

- Streaming is simulated (response is fetched completely, then chunked)
- Token usage is not tracked (returns 0 for all token counts)
- No support for function calling or tool use

