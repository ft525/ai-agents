## Setup

### Copy `docker-compose.yml` and enable the services you need
```bash
cp -p ./docker-compose.yml.sample ./docker-compose.yml
```

### Copy `.env` and adjust settings
```bash
cp -p ./.env.sample ./.env
```




## Local models with Ollama (Optional)

> [!NOTE]
> Strongly recommended to run with an Nvidia GPU or AMD GPU; using CPU only may easily freeze/hang (ref: https://docs.ollama.com/docker) \
> You can also use Ollama's cloud models (registration required; ref: https://docs.ollama.com/cloud) \
> If the locally installed model stays idle for a while, it may be automatically unloaded/stopped (depending on the runtime policy) to save resources. \
> Ollama CLI help: `ollama [COMMAND] (-h | --help)` \
> Ollama API reference: https://docs.ollama.com/api/introduction

### Start Ollama
```bash
docker compose up -d ollama
```

### Download a model
Available models: https://ollama.com/library (e.g. `llama3.1`, `llama3.2`, `gemma3`, `qwen2.5`)
```bash
docker compose exec ollama ollama pull {model}
```

### Quick test (Optional)
```bash
docker compose exec ollama ollama run {model} "Say hello"
```

### Configure OpenClaw to use Ollama
Model-related settings in onboarding:

- Model/auth provider: `Ollama`
- Ollama base URL: `http://ollama:11434` (from inside Docker network)
- Ollama mode:
  - Cloud + Local (You need to register on the official website and obtain an API key)
  - Local
- Default model: ollama/{model}:{variant} (e.g. `ollama/llama3.2:latest`)




## Configure OpenClaw

> [!NOTE]
> The openclaw container image already runs as a non-root user (*node*). \
> The `docker compose run --rm ...` commands start a temporary container, apply configuration to the persisted directory, then exit. \
> The official Docker guide uses a dedicated `openclaw-cli` service name for one-off configuration tasks (e.g. onboarding and channel setup) to keep the long-running gateway service separate. In this repository we run the same CLI through the `openclaw` service instead (by overriding the command in `docker compose run ...`). Both approaches start a temporary container and write changes into the persisted OpenClaw directory. \
> Reference: https://docs.openclaw.ai/install/docker

### Run onboarding (one-off).
```bash
docker compose run --rm openclaw openclaw onboard
```

### Configure Telegram channel (Optional).
```bash
docker compose run --rm openclaw openclaw channels add --channel telegram --token "{token}"
```

> [!NOTE]
> It is recommended to use the new bot, as it will change your webhook configuration.




## Start OpenClaw

### Start the gateway service
```bash
docker compose up -d openclaw
```

Open the Control UI in your browser:\
http://127.0.0.1:18789 or http://localhost:18789

If you cannot find the token, you can retrieve it using one of the following methods:
- Run: `openclaw dashboard --no-open`
- Check config: `./openclaw.json` => key: `gateway.auth.token`

If the dashboard login shows the 'pairing required' error:
- `openclaw devices list` (To get the pending request id)
- `openclaw devices approve {request_id}`

Approve a sender (Communicating with OpenClaw via channel, e.g. Telegram):\
Reference: https://docs.openclaw.ai/channels/pairing
- Message your bot to get a pairing code (any text, but it must not be an OpenClaw command such as `/start`, `/help`, etc.)
- List pending pairing requests: `openclaw pairing list`
- Approve pairing: `openclaw pairing approve {channel} {code}`

If you only changed persisted config via CLI commands and it does not take effect, you can restart:\
`docker compose restart openclaw`




## Skills Management

### List all available skills
```bash
openclaw skills [list]
```

### Install a skill
```bash
openclaw skills install {skill}
```

#### Install & setup gog (Google Workspace CLI) skill
- Install: `openclaw skills install gog`
- Setup:
  - Go to **Google Cloud Console**
  - Create a project (e.g. `OpenClaw CLI`)
  - In the navigation menu, go to `APIs & Services` > `Library`
  - Search for APIs and services
    - Gmail API
    - Google Drive API
    - Google Sheets API
    - Google Docs API
    - Google Calendar API
    - Google People API
  - Enable the API
  - Download the credentials file
    - Go back to `APIs & Services` > `Credentials` (you must configure the OAuth consent screen first)
    - Create credentials (OAuth 2.0 Client ID)
    - Download `client_secret.json` (save it to `./secrets/google/`)
  - Configure Google auth
    - Ask OpenClaw to configure it for you (you need to provide the path to `client_secret.json`)




## Appendix

### Terminology

- Onboarding: The initial setup flow
- Gateway: The entry point that communicates with the LLM, running over WebSocket (ws://127.0.0.1:18789)
- Skill: To teach the agent how to use tools
  - gog: Google Workspace CLI for Gmail, Calendar, Drive, Contacts, Sheets, and Docs
- Channel: A communication integration (e.g. Telegram, Discord)

### Notes

- In the official docs, `openclaw`, `openclaw-cli`, and `openclaw-gateway` may use the same image; the names are used to separate roles (gateway vs one-off CLI tasks).
- How to run multiple gateways (watch resource usage):
  - Define multiple services in `docker-compose.yml` (each gateway must publish a different host port)
  - Use Docker scale: `docker compose up --scale {service}={num} -d`
- Using local models:
  - Run a local model server (e.g. Ollama) and point OpenClaw to it
