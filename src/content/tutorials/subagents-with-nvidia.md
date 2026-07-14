---
title: "Setting Up Subagents in Claude Desktop with NVIDIA Models via LiteLLM + MCP"
description: "Connect Claude Desktop to NVIDIA-hosted models (GLM-5.2, DeepSeek, Gemma, Flux) as subagents using LiteLLM and a custom MCP server."
category: "MCP & Agents"
pubDate: 2026-07-12
difficulty: "intermediate"
---

This guide shows you how to connect **Claude Desktop** to multiple NVIDIA-hosted models (GLM-5.2, DeepSeek, Gemma, Flux, and Paligemma) so you can use them as **subagents**.

---

## Prerequisites

- Python 3.10 or higher
- Claude Desktop installed
- Terminal access
- NVIDIA API keys for the models you want to use:
- Sign up for free at https://build.nvidia.com/models?modal=signin
    - Generate APIs for:
        - [flux_2-klein-4b](https://build.nvidia.com/black-forest-labs/flux_2-klein-4b)
        - [deepseek-v4-pro](https://build.nvidia.com/deepseek-ai/deepseek-v4-pro)
        - [gemma-4-31b-it](https://build.nvidia.com/google/gemma-4-31b-it)
        - [glm-5.2](https://build.nvidia.com/z-ai/glm-5.2)

---

## Step 1: Install LiteLLM

Open your terminal and run:

```bash
pip install litellm
pip install 'litellm[proxy]'
```

---

## Step 2: Create `litellm_config.yaml`

Create a file named `litellm_config.yaml` with the following content (replace the API keys with your own):

```yaml
model_list:
  - model_name: flux.2-klein-4b
    litellm_params:
      model: openai/black-forest-labs/flux.2-klein-4b
      api_base: https://integrate.api.nvidia.com/v1
      api_key: "YOUR_FLUX_API_KEY"
      custom_llm_provider: openai

  - model_name: glm-5.2
    litellm_params:
      model: openai/z-ai/glm-5.2
      api_base: https://integrate.api.nvidia.com/v1
      api_key: "YOUR_GLM_API_KEY"
      custom_llm_provider: openai

  - model_name: deepseek-v4-pro
    litellm_params:
      model: openai/deepseek-ai/deepseek-v4-pro
      api_base: https://integrate.api.nvidia.com/v1
      api_key: "YOUR_DEEPSEEK_API_KEY"
      custom_llm_provider: openai

  - model_name: gemma-4-31b-it
    litellm_params:
      model: openai/google/gemma-4-31b-it
      api_base: https://integrate.api.nvidia.com/v1
      api_key: "YOUR_GEMMA_API_KEY"
      custom_llm_provider: openai

```

---

## Step 3: Start LiteLLM

Navigate to the folder where you saved `litellm_config.yaml` and run:

```bash
litellm --config litellm_config.yaml --port 4000
```

You should see output similar to:

```
LiteLLM: Proxy initialized successfully
Server running on http://0.0.0.0:4000
```

**Leave this terminal open.**

Test that LiteLLM is working:

```bash
curl http://localhost:4000/v1/models
```

You should see your models listed.

---

## Step 4: Create the MCP Server (`server.py`)

### 4.1 Create a folder

```bash
mkdir mcp-litellm-python
cd mcp-litellm-python
```

### 4.2 Create `server.py`

Create a file called `server.py` with this content:

```python
from mcp.server.fastmcp import FastMCP
from openai import OpenAI

# Initialize FastMCP server
mcp = FastMCP("litellm-proxy")

# Connect to your running LiteLLM instance
client = OpenAI(
    base_url="http://localhost:4000/v1",
    api_key="dummy"  # LiteLLM handles the real NVIDIA keys
)

@mcp.tool()
def call_llm(model: str, prompt: str) -> str:
    """
    Call any model through your LiteLLM proxy.
    Example models: glm-5.2, deepseek-v4-pro, gemma-4-31b-it, flux.2-klein-4b
    """
    response = client.chat.completions.create(
        model=model,
        messages=[{"role": "user", "content": prompt}]
    )
    return response.choices[0].message.content or "No response"

if __name__ == "__main__":
    mcp.run()
```

### 4.3 Install dependencies

```bash
pip install mcp openai
```

---

## Step 5: Start the MCP Server

In the `mcp-litellm-python` folder, run:

```bash
python server.py
```

**Leave this terminal open** as well.

---

## Step 6: Configure Claude Desktop

### 6.1 Open the config file

1. Open **Claude Desktop**
2. Go to **Developer → Local MCP servers**
3. Click **Edit Config**

### 6.2 Add the MCP server

Add the following inside your `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "litellm": {
      "command": "python",
      "args": ["/absolute/path/to/mcp-litellm-python/server.py"],
      "env": {}
    }
  }
}
```

> Replace `/absolute/path/to/mcp-litellm-python/server.py` with the actual path on your machine.

### 6.3 Restart Claude Desktop

Fully quit and reopen Claude Desktop.

You should now see `litellm` under **Local MCP servers**.

---

## Step 7: Correct Startup Order

You **must** start the services in this order:

| Order | Service           | Command                                              | Keep Running? |
|-------|-------------------|------------------------------------------------------|---------------|
| 1     | **LiteLLM**       | `litellm --config litellm_config.yaml --port 4000`   | Yes           |
| 2     | **MCP Server**    | `python server.py` (inside `mcp-litellm-python/`)    | Yes           |
| 3     | **Claude Desktop**| Open the app                                         | —             |

If anything stops working, restart in this same order.

---

## Step 8: How to Use the Models as Subagents in Claude

Once everything is running, you can ask Claude to use specific models using the `call_llm` tool.

### Examples

**Use GLM-5.2 for reasoning:**
```
Use call_llm with model "glm-5.2" to explain how transformers work in simple terms.
```

**Use Flux for image generation:**
```
Use call_llm with model "flux.2-klein-4b" to generate an image of a cyberpunk samurai.
```

**Use DeepSeek for coding:**
```
Use call_llm with model "deepseek-v4-pro" to write a FastAPI endpoint with authentication.
```

Claude will route the request to the correct model through LiteLLM.

---

## Troubleshooting

| Problem                          | Possible Cause                          | Solution |
|----------------------------------|-----------------------------------------|----------|
| "Server disconnected"            | MCP server not running                  | Run `python server.py` |
| Model returns 404 or not found   | LiteLLM not running or wrong model name | Restart LiteLLM and verify model name |
| Claude doesn't see `call_llm`    | Claude Desktop not restarted            | Fully quit and reopen Claude Desktop |
| Image generation fails           | Wrong model or endpoint issue           | Use `flux.2-klein-4b` for images |

---

## Summary: Full Startup Sequence

```bash
# Terminal 1 - LiteLLM
cd /path/to/your/litellm-config
litellm --config litellm_config.yaml --port 4000

# Terminal 2 - MCP Server
cd /path/to/mcp-litellm-python
python server.py

# Then open Claude Desktop
```

Once both services are running, you can use Claude as a router to multiple high-quality NVIDIA models.

---

## Project Files

You can download the complete files used in this tutorial from the repository below:

<a class="btn-github" href="https://github.com/ecellsworth/nvidia-subagents-mcp">View repository on GitHub</a>
