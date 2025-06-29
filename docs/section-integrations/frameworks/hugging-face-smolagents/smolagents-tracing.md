---
description: How to use the SmolagentsInstrumentor to trace smolagents by Hugging Face
---

# smolagents Tracing

smolagents is a minimalist AI agent framework developed by Hugging Face, designed to simplify the creation and deployment of powerful agents with just a few lines of code. It focuses on simplicity and efficiency, making it easy for developers to leverage large language models (LLMs) for various applications.

Phoenix provides auto-instrumentation, allowing you to track and visualize every step and call made by your agent.

{% embed url="https://colab.research.google.com/github/Arize-ai/phoenix/blob/main/tutorials/tracing/smolagents_tracing_tutorial.ipynb" %}

## Launch Phoenix

We have several code samples below on different ways to integrate with smolagents, based on how you want to use Phoenix.

{% tabs %}
{% tab title="Phoenix Cloud" %}
**Sign up for Phoenix:**

Sign up for an Arize Phoenix account at [https://app.phoenix.arize.com/login](https://app.phoenix.arize.com/login)

**Install packages:**

```bash
pip install arize-phoenix-otel
```

**Set your Phoenix endpoint and API Key:**

```python
import os

# Add Phoenix API Key for tracing
PHOENIX_API_KEY = "ADD YOUR API KEY"
os.environ["PHOENIX_CLIENT_HEADERS"] = f"api_key={PHOENIX_API_KEY}"
os.environ["PHOENIX_COLLECTOR_ENDPOINT"] = "https://app.phoenix.arize.com"
```

Your **Phoenix API key** can be found on the Keys section of your [dashboard](https://app.phoenix.arize.com).
{% endtab %}

{% tab title="Command Line" %}
**Launch your local Phoenix instance:**

```bash
pip install arize-phoenix
phoenix serve
```

For details on customizing a local terminal deployment, see [Terminal Setup](https://arize.com/docs/phoenix/setup/environments#terminal).

**Install packages:**

```bash
pip install arize-phoenix-otel
```

**Set your Phoenix endpoint:**

```python
import os

os.environ["PHOENIX_COLLECTOR_ENDPOINT"] = "http://localhost:6006"
```

See Terminal for more details
{% endtab %}

{% tab title="Docker" %}
**Pull latest Phoenix image from** [**Docker Hub**](https://hub.docker.com/r/arizephoenix/phoenix)**:**

```bash
docker pull arizephoenix/phoenix:latest
```

**Run your containerized instance:**

```bash
docker run -p 6006:6006 arizephoenix/phoenix:latest
```

This will expose the Phoenix on `localhost:6006`

**Install packages:**

```bash
pip install arize-phoenix-otel
```

**Set your Phoenix endpoint:**

```python
import os

os.environ["PHOENIX_COLLECTOR_ENDPOINT"] = "http://localhost:6006"
```

For more info on using Phoenix with Docker, see [Docker](https://arize.com/docs/phoenix/self-hosting/deployment-options/docker).
{% endtab %}

{% tab title="Notebook" %}
**Install packages:**

```bash
pip install arize-phoenix
```

**Launch Phoenix:**

```python
import phoenix as px
px.launch_app()
```

{% hint style="info" %}
By default, notebook instances do not have persistent storage, so your traces will disappear after the notebook is closed. See [self-hosting ](https://arize.com/docs/phoenix/self-hosting)or use one of the other deployment options to retain traces.
{% endhint %}
{% endtab %}
{% endtabs %}

## Install

```bash
pip install openinference-instrumentation-smolagents smolagents
```

## Setup

Add your `HF_TOKEN` as an environment variable:

```python
os.environ["HF_TOKEN"] = "<your_hf_token_value>"
```

Connect to your Phoenix instance using the register function.

```python
from phoenix.otel import register

# configure the Phoenix tracer
tracer_provider = register(
  project_name="my-llm-app", # Default is 'default'
  auto_instrument=True # Auto-instrument your app based on installed OI dependencies
)
```

## Create & Run an Agent

Create your Hugging Face Model, and at every run, traces will be sent to Phoenix.

```python
from smolagents import (
    CodeAgent,
    InferenceClientModel,
    ToolCallingAgent,
    VisitWebpageTool,
    WebSearchTool,
)

model = InferenceClientModel()

managed_agent = ToolCallingAgent(
    tools=[DuckDuckGoSearchTool(), VisitWebpageTool()],
    model=model,
    name="managed_agent",
    description="This is an agent that can do web search.",
)
manager_agent.run("Based on the latest news, what is happening in extraterrestrial life?")
```

## Observe

Now that you have tracing setup, all invocations and steps of your Agent will be streamed to your running Phoenix for observability and evaluation.

## Resources

* [OpenInference package](https://github.com/Arize-ai/openinference/tree/main/python/instrumentation/openinference-instrumentation-smolagents)
* [Working examples](https://github.com/Arize-ai/openinference/tree/main/python/instrumentation/openinference-instrumentation-smolagents/examples)
* [Smolagents Tracing Documentation](https://huggingface.co/docs/smolagents/en/tutorials/inspect_runs)
