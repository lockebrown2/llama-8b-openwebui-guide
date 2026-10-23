# Chat with Llama 3.1 8B (api.market) using Open WebUI

This guide shows how to run a local ChatGPT-style web interface ([Open WebUI](https://github.com/open-webui/open-webui)) on top of an OpenAI-compatible Llama 3.1 8B Instruct API sold on [api.market](https://api.market).

You will need:

- An api.market account with an active subscription to the API
- [Docker](https://docs.docker.com/get-docker/) (recommended) or Python 3.11
- About 10 minutes

---

## 1. Get your API key

1. Sign in at <https://api.market>.
2. Subscribe to the Llama 3.1 8B Instruct API (<https://api.market/store/arbit-ai-api/llama>.
3. Copy your key from <https://api.market/account/apiKeys>.

api.market authenticates requests with the `x-api-market-key` header. It is **not** a standard `Authorization: Bearer` key, which is why the setup below passes it as a custom header.

## 2. Endpoint details

| Value | What to use | Placeholder used below |
|---|---|---|
| Base URL | `https://prod.api.market/api/v1/arbit-ai-api/llama/v1` | already filled in |
| Model name | `llama-3.1-8b-instruct` | already filled in |
| Your key | Account > API keys | `YOUR_KEY` |

Replace `YOUR_KEY` in every command below with your own key.

## 3. Test the API first

Always confirm the API works before adding a UI. It makes problems much easier to diagnose.

**macOS / Linux / Git Bash**

```bash
curl -X POST "https://prod.api.market/api/v1/arbit-ai-api/llama/v1/chat/completions" \
  -H "x-api-market-key: YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"llama-3.1-8b-instruct","messages":[{"role":"user","content":"Say hello"}]}'
```

**Windows PowerShell** (use `curl.exe`, not `curl`)

```powershell
curl.exe -X POST "https://prod.api.market/api/v1/arbit-ai-api/llama/v1/chat/completions" -H "x-api-market-key: YOUR_KEY" -H "Content-Type: application/json" -d '{\"model\":\"llama-3.1-8b-instruct\",\"messages\":[{\"role\":\"user\",\"content\":\"Say hello\"}]}'
```

You should get a JSON response containing `choices[0].message.content`. If you do not, fix this before continuing (see [Troubleshooting](#troubleshooting)).

## 4. Run Open WebUI

### Option A: Docker (recommended)

```bash
docker run -d -p 3000:8080 -e ENABLE_OLLAMA_API=false -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:main
```

Open <http://localhost:3000> and create the local admin account. This account lives only on your machine.

### Option B: pip (no Docker)

```bash
pip install open-webui
open-webui serve
```

Open <http://localhost:8080> and create the local admin account.

## 5. Connect Open WebUI to the API

1. Go to **Admin Settings > Connections > OpenAI API**.
2. Click **Add Connection** (the **+** button).
3. Fill in:

| Field | Value |
|---|---|
| URL | `https://prod.api.market/api/v1/arbit-ai-api/llama/v1` |
| API key | Any placeholder, for example `unused` (the real key goes in the header) |
| Headers | `{"x-api-market-key": "YOUR_KEY"}` |
| Model IDs | `llama-3.1-8b-instruct` |

4. Click the verify/check button, then **Save**.
5. Start a new chat and select the model from the picker.

> **Note on the Headers field:** It is only available in recent Open WebUI versions. If you do not see it, update the image (`docker pull ghcr.io/open-webui/open-webui:main`, then recreate the container) or use the [proxy fallback](#fallback-add-the-header-with-a-local-proxy).

> **Note on Model IDs:** Entering the model ID manually makes the model appear even if the API does not implement `GET /models`.

---

## Fallback: add the header with a local proxy

If your Open WebUI version has no custom headers option, run [LiteLLM](https://github.com/BerriAI/litellm) as a small local proxy. It adds the `x-api-market-key` header for you and exposes a standard OpenAI-style endpoint.

**1. Install**

```bash
pip install 'litellm[proxy]'
```

**2. Create `config.yaml`**

```yaml
model_list:
  - model_name: llama-3.1-8b
    litellm_params:
      model: openai/llama-3.1-8b-instruct
      api_base: https://prod.api.market/api/v1/arbit-ai-api/llama/v1
      api_key: unused
      extra_headers:
        x-api-market-key: YOUR_KEY
```

Keep this file private since it contains your key.

**3. Start the proxy**

```bash
litellm --config config.yaml --port 4000
```

**4. Point Open WebUI at the proxy**

In **Admin Settings > Connections > OpenAI API**, add:

| Field | Value |
|---|---|
| URL | `http://host.docker.internal:4000/v1` (Docker on Windows/macOS) or `http://localhost:4000/v1` (pip install) |
| API key | `unused` |
| Model IDs | `llama-3.1-8b` |

On Linux with Docker, add `--add-host=host.docker.internal:host-gateway` to the `docker run` command in step 4.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `401` / `403` in the curl test | Wrong or missing key, or not subscribed | Re-copy the key, confirm the subscription is active, and check the header name is exactly `x-api-market-key` |
| `404` in the curl test | Wrong base URL or path | Check the URL against the API's page on api.market; the full request URL should be `https://prod.api.market/api/v1/arbit-ai-api/llama/v1/chat/completions` |
| curl works but Open WebUI shows no models | API has no `GET /models` | Enter the model ID manually under **Model IDs** |
| Connection verifies but chats fail with `401` | Header not being sent | Check the Headers JSON is valid, or use the proxy fallback |
| `Connection refused` to the proxy from Docker | Container cannot reach host | Use `host.docker.internal`, or add the `--add-host` flag on Linux |
| First request is slow, then fast | Cold start on the provider's GPU workers | Wait and retry; later requests are faster |
| `429` errors | Rate limit or quota reached | Check your plan limits on api.market |

## Security tips

- Do not commit your key or `config.yaml` to a public repository.
- Open WebUI stores your key in its local database. Do not expose port 3000 to the internet without authentication and HTTPS.
- Rotate your key at <https://api.market/account/apiKeys> if you think it leaked.

## Links

- api.market: <https://api.market>
- Open WebUI: <https://github.com/open-webui/open-webui>
- LiteLLM proxy: <https://github.com/BerriAI/litellm>
