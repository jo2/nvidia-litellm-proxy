# LiteLLM Proxy

A self-hosted LiteLLM AI gateway for experimenting with NVIDIA-hosted models and using them from clients such as Claude Code. 
The stack runs LiteLLM Proxy and PostgreSQL with Docker Compose.

## What this repository contains
- **LiteLLM Proxy** --- exposes API endpoints and routes requests to configured model providers.
- **PostgreSQL 18** --- persists model configuration, virtual keys, and spend logs for the Admin UI.
- **LiteLLM Admin UI** --- browser-based management for models, keys, usage, and testing prompts.

Useful upstream documentation:
- [LiteLLM Docker quickstart](https://docs.litellm.ai/docs/proxy/docker_quick_start)
- [LiteLLM configuration reference](https://docs.litellm.ai/docs/proxy/configs)
- [Claude Code integration](https://docs.litellm.ai/docs/proxy/client_setup/claude_code)
- [NVIDIA NIM provider](https://docs.litellm.ai/docs/providers/nvidia)

## Requirements
- Docker Engine and the Docker Compose plugin
- An NVIDIA API Catalog / NVIDIA NIM API key for the model you want to use
- `curl` and `openssl` are useful for setup and troubleshooting

## Quick start

### 1. Configure environment variables
Create a `.env` file based on the `.env.example` template:

``` dotenv
LITELLM_MASTER_KEY=sk-replace-with-a-long-random-secret
LITELLM_SALT_KEY=sk-replace-with-another-long-random-secret
POSTGRES_PASSWORD=replace-with-a-strong-database-password

# Optional: bind the published port to localhost only.
LITELLM_BIND=127.0.0.1
LITELLM_PORT=4000
```

Generate secure values, for example:

``` bash
openssl rand -hex 32
```

Use the generated values to populate the file.
Do not commit `.env` to version control.
If you already have a working database, **keep the existing `LITELLM_SALT_KEY`**: changing it can make provider credentials stored in the database unreadable.

`LITELLM_MASTER_KEY` is an administrator credential. Anyone who obtains it can manage the proxy.
Use a separate LiteLLM virtual key for normal client access.

### 2. Configure the NVIDIA provider
Create a `config.yaml` file with the models you want to use, for example:

``` yaml
model_list:
  - model_name: nvidia-coding
    litellm_params:
      model: nvidia_nim/deepseek-ai/deepseek-v4.1-flash
      api_key: os.environ/NVIDIA_NIM_API_KEY
```

Those models will then be available via litellm.
The corresponding model name is the name set in the parameter `model_name` in the config file, e.g. `nvidia-coding`.

### 3. Start the stack

``` bash
docker compose up -d
docker compose ps
```

Or use the shipped IntelliJ run configuration `start-litellm` to start the stack from the IDE.

### 4. Open the Admin UI

- **Admin UI:** <http://localhost:4000/ui>
- **Username:** `admin`
- **Password:** the value of `LITELLM_MASTER_KEY` in `.env`

## URLs and API endpoints

Assuming the proxy is published on port `4000` on the same machine:

| Purpose                                                      | URL                                                      |
|--------------------------------------------------------------|----------------------------------------------------------|
| Admin UI                                                     | `http://localhost:4000/ui`                               |
| API base URL                                                 | `http://localhost:4000`                                  |
| OpenAI-compatible chat completions                           | `http://localhost:4000/v1/chat/completions`              |
| List available models                                        | `http://localhost:4000/v1/models`                        |
| Anthropic Messages API (Claude Code)                         | `http://localhost:4000/v1/messages`                      |
| Health check                                                 | `http://localhost:4000/health`                           |
| OpenAPI / Swagger docs (if enabled by the installed version) | `http://localhost:4000/` or `http://localhost:4000/docs` |

Health and documentation routes can differ by LiteLLM version; verify them against the running version if a route is unavailable.
Do not expose the Admin UI or proxy to an untrusted network without suitable authentication and network controls.

## Use with Claude Code

Claude Code uses the Anthropic Messages API.
LiteLLM exposes a compatible endpoint, so point Claude Code at the proxy and use the LiteLLM model alias.

``` bash
export ANTHROPIC_BASE_URL="http://localhost:4000"
export ANTHROPIC_AUTH_TOKEN="your-litellm-virtual-key"
export ANTHROPIC_MODEL="nvidia-coding"

# Avoid accidentally using an unrelated Anthropic API key for this session.
unset ANTHROPIC_API_KEY

claude
```

`ANTHROPIC_MODEL` must match a model alias exposed by LiteLLM, such as `nvidia-coding`; it should not be set to the full provider string unless that exact string is also registered as the public alias.

To persist these settings, add them to your shell profile or use Claude Code's supported settings mechanism. Avoid putting secrets in a checked-in settings file.

Or using a `settings.local.json` file in the Claude Code project:

``` json
{
  "env": {
    "ANTHROPIC_BASE_URL": "http://localhost:4000",
    "ANTHROPIC_AUTH_TOKEN": "sk-replace-with-a-litellm-virtual-key",
    "ANTHROPIC_MODEL": "nvidia-coding"
  },
  ...
}
```

## Authentication and keys

- **Master key (`LITELLM_MASTER_KEY`):** administrator credential for the proxy and Admin UI.
- **Salt key (`LITELLM_SALT_KEY`):** used to encrypt stored provider credentials.
  Keep it stable for the lifetime of the stored data and back it up securely.
- **Virtual keys:** create these in the Admin UI for clients such as Claude Code.
  They can be scoped to models and can have budgets/rate limits depending on configuration.
- **Provider key:** the NVIDIA API key used by LiteLLM to authenticate upstream.
  Do not give the NVIDIA provider key to client applications.

Never commit `.env`, provider API keys, master keys, or virtual keys.
For remote deployments, use TLS, restrict network access, and follow LiteLLM's production deployment/security guidance.

## Configuration reference

| Variable             | Purpose                                                                                |
|----------------------|----------------------------------------------------------------------------------------|
| `LITELLM_MASTER_KEY` | Required administrator/master credential                                               |
| `LITELLM_SALT_KEY`   | Required stable encryption salt for stored credentials                                 |
| `POSTGRES_PASSWORD`  | Password for the PostgreSQL `litellm` user; must match the database URL                |
| `NVIDIA_NIM_API_KEY` | Optional environment variable for NVIDIA upstream authentication                       |
| `LITELLM_BIND`       | Optional Compose variable; set to `127.0.0.1:` to bind the published port to localhost |
| `LITELLM_PORT`       | Optional published host port; defaults to `4000`                                       |

The LiteLLM container listens on port `4000`. `LITELLM_PORT` changes the host-side published port, not the internal container port.

## License and upstream project

LiteLLM is developed by [BerriAI](https://github.com/BerriAI/litellm).
Refer to the upstream project and documentation for license terms, release notes, provider-specific settings, and production deployment guidance.
