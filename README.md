---
title: First Agent Template
emoji: ⚡
colorFrom: pink
colorTo: yellow
sdk: gradio
sdk_version: 5.23.1
app_file: app.py
pinned: false
tags:
- smolagents
- agent
- smolagent
- tool
- agent-course
---

# smolagents Terminal Agent

A starter template from the [Hugging Face Agents Course](https://huggingface.co/learn/agents-course) for building AI agents with [`smolagents`](https://github.com/huggingface/smolagents). Supports both a terminal REPL and an optional Gradio web UI.

---

## Project Structure

```
smolagents-terminal-agent/
├── app.py              # Entry point — agent setup and terminal REPL
├── Gradio_UI.py        # Optional Gradio web chat interface
├── prompts.yaml        # Custom prompt templates for the agent
├── agent.json          # Agent metadata/configuration
├── requirements.txt    # Python dependencies
├── .env                # Environment variables (HF_TOKEN)
└── tools/
    ├── final_answer.py  # Mandatory tool to return the agent's final answer
    ├── web_search.py    # DuckDuckGo web search tool
    └── visit_webpage.py # Fetches and converts a URL to markdown
```

---

## How It Works

### Agent (`app.py`)

The core of the project is a [`CodeAgent`](https://huggingface.co/docs/smolagents/reference/agents#smolagents.CodeAgent) from the `smolagents` library. A `CodeAgent` solves tasks by writing and executing Python code snippets in a loop, following a **Thought → Code → Observation** cycle until it reaches a final answer.

```
You: <task>
  → Thought: reasoning about the task
  → Code: Python snippet using available tools
  → Observation: output of the code
  → ... (repeat up to max_steps)
  → final_answer(result)
Agent: <answer>
```

Key configuration in `app.py`:

| Parameter | Value | Description |
|---|---|---|
| `model_id` | `Qwen/Qwen2.5-Coder-32B-Instruct` | LLM used via HF Inference API |
| `max_tokens` | `2096` | Max tokens per model response |
| `temperature` | `0.5` | Sampling temperature |
| `max_steps` | `6` | Maximum reasoning steps per task |
| `planning_interval` | `None` | Explicit planning phase disabled |

The `HF_TOKEN` environment variable (loaded from `.env`) authenticates against the Hugging Face Inference API.

### Tools

Tools are Python callables the agent can invoke from within its generated code. They are defined either inline in `app.py` with the `@tool` decorator or as classes in `tools/`.

| Tool | Location | Description |
|---|---|---|
| `final_answer` | `tools/final_answer.py` | **Required.** Signals the end of the agent loop and returns the result. |
| `get_current_time_in_timezone` | `app.py` | Returns the current local time for a given timezone string (e.g. `America/New_York`). |
| `my_custom_tool` | `app.py` | Placeholder tool — replace with your own logic. |
| `DuckDuckGoSearchTool` | `tools/web_search.py` | Runs a DuckDuckGo text search and returns the top results as markdown. |
| `VisitWebpageTool` | `tools/visit_webpage.py` | Fetches a URL and returns its content converted to markdown (truncated at 10 000 chars). |

> **Note:** `DuckDuckGoSearchTool`, `VisitWebpageTool`, and the HF Hub `image_generation_tool` are defined/imported in `app.py` but are **not wired into the agent** by default. Add them to the `tools=[...]` list in the `CodeAgent` constructor to enable them.

### Prompt Templates (`prompts.yaml`)

All prompt templates used by the agent are stored in `prompts.yaml` and loaded at startup. This file controls:

- **`system_prompt`** — Core instructions for the agent, including the Thought/Code/Observation format, tool usage rules, and authorized Python imports.
- **`planning`** — Templates for the optional planning phase (`initial_facts`, `initial_plan`, `update_facts_*`, `update_plan_*`).
- **`managed_agent`** — Templates for multi-agent orchestration (when this agent is managed by another).
- **`final_answer`** — Fallback template used if the agent gets stuck and a recovery answer is needed.

### Gradio Web UI (`Gradio_UI.py`)

`Gradio_UI.py` provides a `GradioUI` class that wraps any `MultiStepAgent` in a Gradio chat interface. It is not invoked from `app.py` by default, but can be used instead of the terminal REPL:

```python
from Gradio_UI import GradioUI
GradioUI(agent).launch()
```

Key features:
- Streams each agent step (thought, code, observations, errors) as nested chat messages in real time.
- Tracks and displays input/output token counts and step duration.
- Optional file upload support (PDF, DOCX, TXT) — enabled by passing a `file_upload_folder` path to `GradioUI`.

The app is configured as a **Hugging Face Space** (see the YAML frontmatter above), so deploying it to the Hub will automatically launch the Gradio UI at the Space URL.

---

## Quickstart

### 1. Install dependencies

```bash
pip install -r requirements.txt
```

### 2. Configure your token

```bash
cp .env.example .env   # or create .env manually
# add: HF_TOKEN=hf_...
```

### 3. Run the terminal agent

```bash
python app.py
```

### 4. (Optional) Launch the Gradio UI locally

```python
# In app.py, replace the __main__ block with:
from Gradio_UI import GradioUI
GradioUI(agent).launch()
```

---

## Extending the Agent

- **Add a new tool:** define a function with the `@tool` decorator (or subclass `smolagents.tools.Tool`) and append it to the `tools=[...]` list in `CodeAgent`.
- **Change the model:** swap `model_id` in `InferenceClientModel` for any text-generation model available on the HF Hub, or point it to a custom endpoint.
- **Enable planning:** set `planning_interval=N` (e.g. `1`) to trigger a planning step every N agent steps.
