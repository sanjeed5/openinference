# Python Workspace Guide

Python instrumentation packages for OpenInference. Uses tox + tox-uv for automation, ruff for formatting/linting, mypy for type checking, and pytest for tests. Python 3.9+ supported.

The `openinference` namespace package is composed by symlinking individual instrumentor source trees into the core package directory (`add_symlinks` tox environment).

---

## Setup

```bash
cd python
pip install tox-uv==1.11.2
pip install -r dev-requirements.txt
tox run -e add_symlinks              # compose namespace package (required before any imports work)
pip install -e openinference-instrumentation  # editable install for development
```

---

## tox Command Reference

### How tox factors work (critical, non-obvious)

An environment string like `ruff-mypy-test-openai` is a **hyphen-delimited conjunction** of 4 factors: `ruff`, `mypy`, `test`, and `openai`. tox creates one virtual environment named after the concatenation and runs all matching factor actions inside it. You will **not** find `ruff-mypy-test-openai` literally defined in `tox.ini` — it is assembled at runtime from its component factors.

The `-` is a conjunction, not a separator between distinct commands.

### Common commands

```bash
tox run-parallel                     # all CI checks in parallel (all envlist entries)
tox run -e ruff-openai               # format and lint the openai package
tox run -e mypy-openai               # type-check the openai package
tox run -e test-openai               # run tests for the openai package
tox run -e ruff-mypy-test-openai     # all three checks at once for openai
tox run -e ruff-openai,ruff-semconv  # multiple packages in one invocation (comma-separated)
```

Replace `openai` with any package token from the list below.

### Package tokens (use in tox commands)

| Token | Package directory |
|-------|------------------|
| `semconv` | `openinference-semantic-conventions/` |
| `instrumentation` | `openinference-instrumentation/` |
| `openai` | `instrumentation/openinference-instrumentation-openai/` |
| `openai_agents` | `instrumentation/openinference-instrumentation-openai-agents/` |
| `anthropic` | `instrumentation/openinference-instrumentation-anthropic/` |
| `bedrock` | `instrumentation/openinference-instrumentation-bedrock/` |
| `mistralai` | `instrumentation/openinference-instrumentation-mistralai/` |
| `groq` | `instrumentation/openinference-instrumentation-groq/` |
| `litellm` | `instrumentation/openinference-instrumentation-litellm/` |
| `langchain` | `instrumentation/openinference-instrumentation-langchain/` |
| `llama_index` | `instrumentation/openinference-instrumentation-llama-index/` |
| `dspy` | `instrumentation/openinference-instrumentation-dspy/` |
| `instructor` | `instrumentation/openinference-instrumentation-instructor/` |
| `crewai` | `instrumentation/openinference-instrumentation-crewai/` |
| `haystack` | `instrumentation/openinference-instrumentation-haystack/` |
| `vertexai` | `instrumentation/openinference-instrumentation-vertexai/` |
| `smolagents` | `instrumentation/openinference-instrumentation-smolagents/` |
| `autogen` | `instrumentation/openinference-instrumentation-autogen/` |
| `autogen_agentchat` | `instrumentation/openinference-instrumentation-autogen-agentchat/` |
| `beeai` | `instrumentation/openinference-instrumentation-beeai/` |
| `portkey` | `instrumentation/openinference-instrumentation-portkey/` |
| `mcp` | `instrumentation/openinference-instrumentation-mcp/` |
| `google_genai` | `instrumentation/openinference-instrumentation-google-genai/` |
| `google_adk` | `instrumentation/openinference-instrumentation-google-adk/` |
| `pydantic_ai` | `instrumentation/openinference-instrumentation-pydantic-ai/` |
| `openllmetry` | `instrumentation/openinference-instrumentation-openllmetry/` |
| `openlit` | `instrumentation/openinference-instrumentation-openlit/` |
| `strands_agents` | `instrumentation/openinference-instrumentation-strands-agents/` |
| `pipecat` | `instrumentation/openinference-instrumentation-pipecat/` |
| `agent_framework` | `instrumentation/openinference-instrumentation-agent-framework/` |
| `agno` | `instrumentation/openinference-instrumentation-agno/` |
| `guardrails` | `instrumentation/openinference-instrumentation-guardrails/` |
| `agentspec` | `instrumentation/openinference-instrumentation-agentspec/` |

---

## The Three Required Features

Every Python instrumentor must implement these three features.

### Feature 1 — Suppress Tracing

Check the OTel context key before creating any span. When suppressed, call through to the original function without tracing.

```python
from opentelemetry import context as context_api
from opentelemetry.context import _SUPPRESS_INSTRUMENTATION_KEY

def patched_function(*args, **kwargs):
    if context_api.get_value(_SUPPRESS_INSTRUMENTATION_KEY):
        return original_function(*args, **kwargs)  # skip tracing
    # ... tracing logic ...
```

To permanently disable tracing, implement `_uninstrument()` to reverse all monkey-patching.

### Feature 2 — Context Attribute Propagation

Read session ID, user ID, metadata, and tags from OTel context and attach them to spans.

```python
from opentelemetry.trace import Tracer
from openinference.instrumentation import get_attributes_from_context

# Pass at span creation time:
span = tracer.start_span(
    name="my-span",
    attributes=dict(get_attributes_from_context()),
)

# Or set after span creation:
with tracer.start_as_current_span(name="my-span") as span:
    span.set_attributes(dict(get_attributes_from_context()))
```

Available context managers (imported from `openinference.instrumentation`):

| Context Manager | Purpose |
|----------------|---------|
| `using_session(id)` | Attach a session ID to all spans in the block |
| `using_user(id)` | Attach a user ID to all spans in the block |
| `using_metadata({})` | Attach custom metadata to all spans in the block |
| `using_tag([])` | Attach tags to all spans in the block |
| `using_prompt_template(template, version, variables)` | Attach prompt template info |
| `using_attributes(...)` | Set multiple context attributes at once |

### Feature 3 — OITracer (TraceConfig masking)

Wrap the raw OTel tracer with `OITracer` so every span respects the user's `TraceConfig` settings (e.g., masking PII).

```python
from openinference.instrumentation import OITracer, TraceConfig
import opentelemetry.trace as trace_api

def _instrument(self, **kwargs: Any) -> None:
    tracer_provider = kwargs.get("tracer_provider") or trace_api.get_tracer_provider()
    config = kwargs.get("config") or TraceConfig()
    assert isinstance(config, TraceConfig)
    tracer = OITracer(
        trace_api.get_tracer(__name__, __version__, tracer_provider),
        config=config,
    )
    # Use `tracer` (not the raw OTel tracer) for all span creation
```

---

## Instrumentor Class Pattern

```python
from typing import Any, Collection
from opentelemetry.instrumentation.instrumentor import BaseInstrumentor

class MyFrameworkInstrumentor(BaseInstrumentor):
    def instrumentation_dependencies(self) -> Collection[str]:
        return ("my-framework >= 1.0",)

    def _instrument(self, **kwargs: Any) -> None:
        # 1. Get tracer_provider and config from kwargs
        # 2. Wrap with OITracer
        # 3. Apply monkey-patches to the target library

    def _uninstrument(self, **kwargs: Any) -> None:
        # Reverse all monkey-patches; restore original functions
```

---

## Creating a New Python Instrumentor

1. **Copy the canonical package** as a starting point:
   ```bash
   cp -r python/instrumentation/openinference-instrumentation-openai/ \
         python/instrumentation/openinference-instrumentation-<name>/
   ```

2. **Required files**:
   - `pyproject.toml` — package metadata, dependencies, tool config
   - `src/openinference/instrumentation/<name>/__init__.py` — instrumentor class
   - `src/openinference/instrumentation/<name>/version.py` — `__version__` string
   - `src/openinference/instrumentation/<name>/package.py` — package name constant

3. **Register in `python/tox.ini`**:
   - Add `<token>: instrumentation/openinference-instrumentation-<name>/` under `changedir`
   - Add `<token>: uv pip install ...` lines under `commands_pre`

4. **Register in `release-please-config.json`** — add the new package path so release-please tracks it for version bumps and PyPI releases.

---

## Testing Patterns

Test location: `tests/openinference/instrumentation/<name>/`

Required test categories (all three must be present):

```python
# 1. Suppress tracing test
def test_suppress_tracing(instrumentor, tracer_provider):
    with suppress_tracing():
        result = my_framework_call()
    assert len(get_spans()) == 0  # no spans created

# 2. Context attribute propagation test
def test_context_attributes(instrumentor, tracer_provider):
    with using_session("test-session-id"):
        result = my_framework_call()
    spans = get_spans()
    assert spans[0].attributes["session.id"] == "test-session-id"

# 3. Trace configuration masking test
def test_trace_config_masking(tracer_provider):
    config = TraceConfig(hide_inputs=True)
    instrumentor = MyFrameworkInstrumentor()
    instrumentor.instrument(tracer_provider=tracer_provider, config=config)
    result = my_framework_call(input="sensitive data")
    spans = get_spans()
    assert "input.value" not in spans[0].attributes
```

Pytest configuration in `pyproject.toml`:
```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
```

---

## Semantic Conventions

```python
from openinference.semconv.trace import SpanAttributes, OpenInferenceSpanKindValues

# Required on every span:
span.set_attribute(
    SpanAttributes.OPENINFERENCE_SPAN_KIND,
    OpenInferenceSpanKindValues.LLM.value,
)

# Common LLM attributes:
span.set_attribute(SpanAttributes.LLM_MODEL_NAME, "gpt-4o")
span.set_attribute(SpanAttributes.INPUT_VALUE, prompt_text)
span.set_attribute(SpanAttributes.OUTPUT_VALUE, response_text)
span.set_attribute(SpanAttributes.LLM_TOKEN_COUNT_PROMPT, prompt_tokens)
span.set_attribute(SpanAttributes.LLM_TOKEN_COUNT_COMPLETION, completion_tokens)
span.set_attribute(SpanAttributes.LLM_TOKEN_COUNT_TOTAL, total_tokens)

# Flattened message arrays (zero-based index):
for i, msg in enumerate(messages):
    span.set_attribute(f"llm.input_messages.{i}.message.role", msg["role"])
    span.set_attribute(f"llm.input_messages.{i}.message.content", msg["content"])
```

---

## All Python Instrumentation Packages (33 total)

| Framework | Package name | tox token |
|-----------|-------------|-----------|
| OpenAI | `openinference-instrumentation-openai` | `openai` |
| OpenAI Agents | `openinference-instrumentation-openai-agents` | `openai_agents` |
| Anthropic | `openinference-instrumentation-anthropic` | `anthropic` |
| AWS Bedrock | `openinference-instrumentation-bedrock` | `bedrock` |
| Mistral AI | `openinference-instrumentation-mistralai` | `mistralai` |
| Groq | `openinference-instrumentation-groq` | `groq` |
| LiteLLM | `openinference-instrumentation-litellm` | `litellm` |
| LangChain | `openinference-instrumentation-langchain` | `langchain` |
| LlamaIndex | `openinference-instrumentation-llama-index` | `llama_index` |
| DSPy | `openinference-instrumentation-dspy` | `dspy` |
| Instructor | `openinference-instrumentation-instructor` | `instructor` |
| CrewAI | `openinference-instrumentation-crewai` | `crewai` |
| Haystack | `openinference-instrumentation-haystack` | `haystack` |
| Vertex AI | `openinference-instrumentation-vertexai` | `vertexai` |
| SmolAgents | `openinference-instrumentation-smolagents` | `smolagents` |
| AutoGen | `openinference-instrumentation-autogen` | `autogen` |
| AutoGen AgentChat | `openinference-instrumentation-autogen-agentchat` | `autogen_agentchat` |
| BeeAI | `openinference-instrumentation-beeai` | `beeai` |
| Portkey | `openinference-instrumentation-portkey` | `portkey` |
| MCP | `openinference-instrumentation-mcp` | `mcp` |
| Google GenAI | `openinference-instrumentation-google-genai` | `google_genai` |
| Google ADK | `openinference-instrumentation-google-adk` | `google_adk` |
| Pydantic AI | `openinference-instrumentation-pydantic-ai` | `pydantic_ai` |
| OpenLLMetry | `openinference-instrumentation-openllmetry` | `openllmetry` |
| OpenLIT | `openinference-instrumentation-openlit` | `openlit` |
| Strands Agents | `openinference-instrumentation-strands-agents` | `strands_agents` |
| Pipecat | `openinference-instrumentation-pipecat` | `pipecat` |
| Agent Framework | `openinference-instrumentation-agent-framework` | `agent_framework` |
| Agno | `openinference-instrumentation-agno` | `agno` |
| Guardrails | `openinference-instrumentation-guardrails` | `guardrails` |
| AgentSpec | `openinference-instrumentation-agentspec` | `agentspec` |
| Semantic Conventions | `openinference-semantic-conventions` | `semconv` |
| Core Instrumentation | `openinference-instrumentation` | `instrumentation` |

---

## Publishing

### PyPI

Release-please automates version bumps and publishes to PyPI on PR merge. To add a new instrumentor:
1. Add the package path to `release-please-config.json`
2. Merge a release PR to trigger the GitHub release and PyPI upload

### Conda-Forge

After PyPI publication, create the initial Conda feedstock:

1. Fork `conda-forge/staged-recipes`, add a recipe in `recipes/`
2. Run `grayskull pypi <package-name>` to generate `meta.yaml`
3. Review `meta.yaml`: maintainers, test imports, home URL
4. Open a PR; ping `@conda-forge/help-python` when CI passes
5. After merge, add bot automerge issue: `@conda-forge-admin, please add bot automerge`

Subsequent releases are handled automatically by the Conda bot. Note: Conda-Forge has a several-hour delay detecting PyPI releases; if multiple releases happen the same day, only the last is usually caught.
