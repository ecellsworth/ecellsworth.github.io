---
title: "NVIDIA Prompt Runner: Generate Text & Images from the CLI"
description: "Build a command-line tool that generates text and images using NVIDIA-hosted models like GLM, DeepSeek, Gemma, and Flux."
category: "CLI Tools"
pubDate: 2026-07-13
difficulty: "beginner"
---

This is a step-by-step tutorial for setting up a command-line tool that lets you generate **text** and **images** using NVIDIA models.

---

## Requirements

Before you begin, make sure you have:

- Python 3.10 or higher
- NVIDIA API keys for the models you want to use
- Sign up for free at [NVIDIA Build](https://build.nvidia.com/models?modal=signin)
- Generate API keys for these models:
  - [flux.2-klein-4b](https://build.nvidia.com/black-forest-labs/flux_2-klein-4b)
  - [deepseek-v4-pro](https://build.nvidia.com/deepseek-ai/deepseek-v4-pro)
  - [gemma-4-31b-it](https://build.nvidia.com/google/gemma-4-31b-it)
  - [glm-5.2](https://build.nvidia.com/z-ai/glm-5.2)

---

## Step 1: Create the Project Folder

Open your terminal and run the following commands:

```bash
mkdir nvidia-prompt-runner
cd nvidia-prompt-runner
```

---

## Step 2: Create the `run_prompt.py` Script

Inside the `nvidia-prompt-runner` folder, create a new file called `run_prompt.py` and paste the following code:

```python
from openai import OpenAI
import requests
from datetime import datetime
from pathlib import Path
import argparse
import sys
import base64

# ====================== MODEL CONFIG ======================
MODELS = {
    "flux.2-klein-4b": {
        "type": "image",
        "model": "black-forest-labs/flux.2-klein-4b",
        "api_key": "YOUR_FLUX_API_KEY",
        "endpoint": "https://ai.api.nvidia.com/v1/genai/black-forest-labs/flux.2-klein-4b"
    },
    "glm-5.2": {
        "type": "text",
        "model": "z-ai/glm-5.2",
        "api_key": "YOUR_GLM_API_KEY"
    },
    "deepseek-v4-pro": {
        "type": "text",
        "model": "deepseek-ai/deepseek-v4-pro",
        "api_key": "YOUR_DEEPSEEK_API_KEY"
    },
    "gemma-4-31b-it": {
        "type": "text",
        "model": "google/gemma-4-31b-it",
        "api_key": "YOUR_GEMMA_API_KEY"
    },
}

OUTPUT_DIR = Path("generated_output")
OUTPUT_DIR.mkdir(exist_ok=True)
# ========================================================

def get_client(model_key: str) -> OpenAI:
    config = MODELS[model_key]
    return OpenAI(
        base_url="https://integrate.api.nvidia.com/v1",
        api_key=config["api_key"]
    )

def generate_text(prompt: str, model_key: str) -> str:
    config = MODELS[model_key]
    client = get_client(model_key)
    response = client.chat.completions.create(
        model=config["model"],
        messages=[{"role": "user", "content": prompt}],
        max_tokens=4000,
        temperature=0.7
    )
    return response.choices[0].message.content

def generate_image(prompt: str, model_key: str) -> str:
    config = MODELS[model_key]
    headers = {
        "Authorization": f"Bearer {config['api_key']}",
        "Content-Type": "application/json",
        "Accept": "application/json"
    }
    payload = {
        "prompt": prompt,
        "width": 1024,
        "height": 1024,
        "seed": 0,
        "steps": 4
    }
    response = requests.post(config["endpoint"], headers=headers, json=payload)
    if response.status_code != 200:
        print(f"Error {response.status_code}: {response.text}")
        sys.exit(1)
    result = response.json()
    image_data = result.get("image") or result.get("b64_json")
    if not image_data and "artifacts" in result:
        image_data = result["artifacts"][0].get("base64")
    if not image_data:
        print("Error: Could not extract image data")
        sys.exit(1)
    img_data = base64.b64decode(image_data)

    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    filename = OUTPUT_DIR / f"{model_key}_{timestamp}.png"
    with open(filename, "wb") as f:
        f.write(img_data)
    return str(filename)

def save_text_output(content: str, model_key: str) -> str:
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    filename = OUTPUT_DIR / f"{model_key}_{timestamp}.md"
    with open(filename, "w", encoding="utf-8") as f:
        f.write(content)
    return str(filename)

def main():
    parser = argparse.ArgumentParser(description="Generate text or images using NVIDIA models.")
    parser.add_argument("prompt_file")
    parser.add_argument("--model", required=True, help="Model to use")
    args = parser.parse_args()

    prompt_path = Path(args.prompt_file)
    if not prompt_path.exists():
        print(f"Error: File '{prompt_path}' not found.")
        sys.exit(1)

    prompt = prompt_path.read_text(encoding="utf-8").strip()
    if not prompt:
        print("Error: Prompt file is empty.")
        sys.exit(1)

    model_key = args.model
    if model_key not in MODELS:
        print(f"Error: Unknown model '{model_key}'")
        print("Available models:", ", ".join(MODELS.keys()))
        sys.exit(1)

    model_type = MODELS[model_key]["type"]

    if model_type.startswith("image"):
        print(f"🖼️  Generating image using model: {model_key}")
        saved_path = generate_image(prompt, model_key)
        print(f"✅ Image saved to: {saved_path}")
    else:
        print(f"✍️  Generating text using model: {model_key}")
        result = generate_text(prompt, model_key)
        saved_path = save_text_output(result, model_key)
        print(f"✅ Response saved to: {saved_path}")
        print("\n" + "=" * 60)
        print(result)
        print("=" * 60)

if __name__ == "__main__":
    main()
```

> **Important**: Replace `YOUR_..._API_KEY` with your actual NVIDIA API keys.

---

## Step 3: Create the Output Folder

Run this command to create the folder where generated files will be saved:

```bash
mkdir generated_output
```

---

## Step 4: Create Prompt Files

Create two files in your project folder:

**`text-prompt.md`**
```markdown
Explain how a modern server rack works in simple terms.
```

**`image-prompt.md`**
```markdown
Cinematic technical exploded view of a modern enterprise server rack, components separated and floating apart vertically in an organized manner
```

---

## Step 5: Run the Script

### Generate Text

```bash
python3 run_prompt.py text-prompt.md --model glm-5.2
```

### Generate an Image

```bash
python3 run_prompt.py image-prompt.md --model flux.2-klein-4b
```

---

## Output Location

All generated content is saved inside the `generated_output/` folder:

- Text responses are saved as `.md` files
- Images are saved as `.png` files

---

## Project Files

You can download the complete files used in this tutorial from the repository below:

[View repository on GitHub](https://github.com/ecellsworth/nvidia-prompt-runner)
