# Market Analysis Workflow

This project is a small command-line program that uses four AI agents in sequence to research a market question and write a Markdown report.

If you are coming from Java, the simplest mental model is:

- the Pydantic classes are similar to DTOs/records with runtime validation;
- each `Agent` is a configured AI worker with a name, instructions, optional tools, and an optional output type;
- `Runner.run(...)` is the SDK call that executes one worker;
- `run_workflow(...)` is the service/orchestrator method that passes each worker's result to the next one;
- `main()` is the CLI entry point.

The output is written to `market_report.md`.

## What the program does

Given a question such as:

```text
What are the bull and bear cases for NVIDIA over the next 12 months?
```

the workflow performs these steps:

```text
Command-line question
        |
        v
1. Market Data Retriever -- web search --> EvidencePacket
        |
        v
2. Market Analyst ----------------------> MarketAnalysis
        |
        v
3. Devil's Advocate / Risk Guard -------> RiskReview
        |
        v
   Deterministic Python risk gate
        |
        v
4. Final Report Writer -----------------> Markdown string
        |
        v
market_report.md + terminal output
```

The stages run sequentially. The analyst cannot begin until evidence retrieval finishes, and the risk reviewer cannot begin until the analysis finishes.

## Project contents

- `workflow.py` contains the entire application.
- `README.md` explains it.
- `market_report.md` is created or overwritten when the program completes successfully.

There is currently no dependency file such as `requirements.txt` or `pyproject.toml`, and there are no automated tests.

## Setup and running it

The type syntax in this file requires Python 3.10 or newer. From this directory, create and activate a virtual environment:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

Install the OpenAI Agents SDK and Pydantic:

```bash
python -m pip install openai-agents pydantic
```

Set an OpenAI API key in the current terminal session:

```bash
export OPENAI_API_KEY="your-api-key-here"
```

Do not put a real key in `workflow.py`, commit it to Git, or paste it into a report.

Run the workflow with a quoted research question:

```bash
python workflow.py "What are the bull and bear cases for NVIDIA over the next 12 months?"
```

If you run it without a question, it uses the `DAILY_QUESTION` defined near the
bottom of `workflow.py`:

```bash
python workflow.py
```

While running, it prints progress messages such as `[1/4] Retrieving evidence...`. On success, it prints the report and saves the same content to `market_report.md`.

The script uses the SDK's default model because no `model=...` value is set on the agents. API use is billable. There are four top-level agent runs, and the retrieval run may make one or more web-search tool calls.

## Imports

```python
from __future__ import annotations
```

This postpones the evaluation of type annotations. It can make type hints easier to work with, particularly when types refer to classes that are defined later. The file would mostly work without it on modern Python, but it is a common compatibility choice.

```python
import asyncio
import sys
from pathlib import Path
from typing import Literal
```

- `asyncio` starts Python's asynchronous event loop.
- `sys` provides `sys.argv`, the command-line arguments.
- `Path` provides an object-oriented file API, roughly comparable to Java's `java.nio.file.Path`.
- `Literal` restricts a value to specific strings, which is somewhat like using an enum in Java. Unlike a Java enum, the runtime value is still a normal string.

```python
from agents import Agent, Runner, WebSearchTool
from pydantic import BaseModel
```

- `Agent` describes an AI worker.
- `Runner` executes an agent and manages its model/tool loop.
- `WebSearchTool` gives an agent the ability to request hosted web searches.
- `BaseModel` is Pydantic's base class for validated data models.

## The Pydantic data models

These classes define the contracts between agents. They are similar in purpose to Java DTOs, although Python type hints alone normally do not enforce types at runtime. Pydantic adds parsing and validation.

### `Source`

```python
class Source(BaseModel):
    title: str
    url: str
    published_at: str | None
    supoorts: str
```

This represents one research source:

- `title` is the page or article title.
- `url` is its address.
- `published_at` accepts either a string or `None`, Python's equivalent of `null`.
- `supoorts` appears intended to explain which claim the source supports.

`supoorts` is almost certainly a typo for `supports`. Because this model becomes the AI's output schema, the AI is currently required to emit the misspelled key. Renaming it would change the schema to the intended spelling.

One subtle Pydantic point: `published_at: str | None` means the value may be null, but because it has no default, the field is still required. To make the field genuinely optional/omittable, it would normally be written as `published_at: str | None = None`.

### `EvidencePacket`

`EvidencePacket` is the retriever's result. It groups the original question, an `as_of` timestamp/date, facts, market metrics, context, catalysts, sources, and known gaps.

Fields such as `list[str]` mean "a list whose elements must be strings." `sources: list[Source]` means each source item must match the `Source` model above.

In Java-like pseudocode, part of it would resemble:

```java
record EvidencePacket(
    String researchQuestion,
    String asOf,
    List<String> keyFacts,
    List<Source> sources,
    List<String> dataGaps
) {}
```

The actual Python model has more fields than this abbreviated example.

### `MarketAnalysis`

This is the analyst's structured result. It contains a thesis, bull and bear cases, scenarios, assumptions, and confidence.

```python
confidence: Literal["low", "medium", "high"]
```

That field can contain only one of those three strings. The restriction helps prevent inconsistent values such as `"fairly confident"` from leaking into later code.

### `RiskItem` and `RiskReview`

`RiskItem` describes a single risk and records its likelihood, impact, supporting evidence, and something to mitigate or monitor.

`RiskReview` is the full devil's-advocate result. Its `status` is restricted to:

- `PASS`
- `PASS_WITH_CAVEATS`
- `REVISE`
- `INSUFFICIENT_EVIDENCE`

Its highest severity is restricted to `low`, `medium`, `high`, or `critical`. These constrained values matter because regular Python code later uses them to choose the risk-gate message.

## The four agents

An `Agent(...)` declaration does not run anything. It creates configuration that will be used later by `Runner.run(...)`.

### 1. `retrieval_agent`

This agent receives the user's question directly. Its prompt tells it to collect current price/valuation information, fundamentals, market context, catalysts, citations, and missing data.

It is the only agent with a tool:

```python
tools=[WebSearchTool()]
```

That tool lets the model request web searches while the SDK manages the tool-call loop. The prompt also tells the model to treat webpages as untrusted, cross-check important claims, and avoid inventing missing data.

Its output contract is:

```python
output_type=EvidencePacket
```

This asks the model for structured output and makes the SDK parse/validate the result as an `EvidencePacket`, instead of returning arbitrary prose.

### 2. `analysis_agent`

This agent has no web-search tool. It receives only:

- the original research question; and
- the serialized `EvidencePacket` from stage 1.

Its instructions explicitly prohibit introducing facts that are absent from the evidence. It returns a validated `MarketAnalysis` object.

Keeping research and interpretation separate makes the data flow easier to inspect, although the instruction is still a behavioral request to an LLM rather than a mathematical guarantee.

### 3. `risk_agent`

This agent receives the question, evidence, and market analysis. It is asked to challenge unsupported conclusions, weak sources, bias, missing downside scenarios, and excessive confidence.

It returns a `RiskReview`. Despite "Risk Guard" in its name, this is an ordinary reviewing agent, not an SDK input/output guardrail. Its judgment is produced by an LLM.

### The Python risk gate

After the risk review, normal Python code maps the structured result to one of three messages:

```text
REVISE or INSUFFICIENT_EVIDENCE, or critical severity
    -> BLOCKED FOR DECISION USE

PASS_WITH_CAVEATS
    -> REVIEW REQUIRED

anything else (currently PASS)
    -> RESEARCH COMPLETE
```

This mapping is deterministic: once `risk.status` and `risk.highest_severity` exist, Python follows the same branches every time. However, the values being mapped came from the AI risk reviewer. The code does not independently prove that its assessment is correct.

Also, "blocked" means the final report receives a warning label. It does **not** stop report generation—the report agent still runs in all cases.

### 4. `report_agent`

This agent receives all previous structured data plus the selected risk-gate message. Its instructions request twelve Markdown sections and require it to preserve uncertainty, contradictory evidence, data gaps, and source URLs.

It has no `output_type`, so its final output is a plain string rather than a Pydantic object. That is appropriate here because the desired result is human-readable Markdown.

## How `run_workflow()` orchestrates the agents

The function is declared with `async def` because SDK calls perform network I/O and are awaitable:

```python
async def run_workflow(question: str) -> str:
```

Java analogy: `async def` creates an async function, and each `await` is conceptually similar to waiting on a `CompletableFuture` without writing callback chains. In this particular workflow the awaits are sequential, not parallel, because each stage needs the preceding stage's result.

Each stage follows the same pattern:

```python
result = await Runner.run(agent, input_text)
typed_value = result.final_output
```

For the first three agents, `final_output` is the Pydantic type declared in `output_type`. For the report agent, it is a Markdown string.

The handoff between stages uses:

```python
evidence.model_dump_json(indent=2)
```

This converts a Pydantic object into JSON text that is placed inside the next agent's prompt. It is similar to serializing a Java DTO with Jackson before passing it to another service, except here the JSON is prompt content rather than an HTTP request body.

Finally:

```python
Path("market_report.md").write_text(report, encoding="utf-8")
```

writes the report. This path is relative to the process's current working directory, not necessarily the directory containing `workflow.py`. If you launch the script from another directory, the report will be created there.

## How `main()` handles the command line

`sys.argv` contains command-line tokens:

```text
python workflow.py "Is gold attractive?"
  [0]        [1]
```

- `sys.argv[0]` is normally the script name.
- `sys.argv[1:]` contains the question tokens.

If arguments are supplied, the program joins them into one question:

```python
" ".join(sys.argv[1:])
```

If the resulting string is empty, Python's `or` expression selects
`DAILY_QUESTION` instead:

```python
question = " ".join(sys.argv[1:]).strip() or DAILY_QUESTION
```

Quoting a custom question in the shell is still a good habit.

This common Python entry-point pattern:

```python
if __name__ == "__main__":
    asyncio.run(main())
```

means "run `main()` only when this file is executed directly." If another Python file imports `workflow`, its classes, agents, and functions become available, but the workflow does not start automatically. This is roughly comparable to Java's `public static void main(String[] args)`, with an extra guard that distinguishes direct execution from import.

`asyncio.run(main())` creates an event loop, runs the async `main()` function, and closes the loop afterward.

## What is enforced versus requested

It is useful to separate code-level controls from prompt-level instructions.

Enforced by Python/Pydantic:

- the first three agent outputs must fit their declared data shapes;
- `Literal` fields must use one of their allowed strings;
- risk statuses are mapped to risk-gate messages by explicit `if`/`elif`/`else` code;
- a successful run writes a file named `market_report.md`.

Requested through natural-language instructions:

- using strong and current sources;
- not inventing facts or citations;
- separating facts from estimates;
- not adding facts during analysis;
- preserving every caveat and URL in the final report;
- avoiding financial recommendations.

Structured output makes the shape reliable, but it does not make every value factually correct. Prompts reduce unwanted behavior but do not prove that it cannot happen.

## Important limitations and improvement ideas

For a learning project, the design is clear and deliberately simple. Before treating it as a production workflow, consider these points:

1. **Fix `supoorts`.** Rename it to `supports` so the schema describes its purpose correctly.
2. **Pin dependencies.** Add a `requirements.txt` or `pyproject.toml` so SDK updates do not silently change behavior.
3. **Choose a model explicitly.** No agent declares `model=...`, so behavior follows the installed SDK's default configuration.
4. **Add error handling.** A network error, invalid model output, missing API key, or filesystem error currently ends the whole program with an exception.
5. **Add retries/timeouts thoughtfully.** External model and search calls can fail temporarily.
6. **Anchor the output path if desired.** Using `Path(__file__).resolve().parent / "market_report.md"` would always place the report beside the script.
7. **Validate the final report.** Its section list, source URLs, and claims are requested by prompt but are not checked by Python after generation.
8. **Add provenance checks.** The program does not mechanically verify that every report claim maps to a cited source or that every URL came from the evidence packet.
9. **Add logging/tracing and tests.** Progress `print()` calls are useful, but tests with mocked agent results would make the control flow safer to change.
10. **Remember the financial context.** A four-agent design can catch some weaknesses, but it is not a substitute for verified primary data, a qualified human reviewer, or professional financial advice.

## Useful terms

- **LLM (large language model):** the model that reads prompts and generates text or structured data.
- **Agent:** an LLM configured with instructions, tools, and an expected output.
- **Tool call:** a request by the model to use an external capability, such as web search.
- **Prompt:** the instructions and input text sent to the model.
- **Structured output:** model output constrained to a defined schema instead of free-form prose.
- **Serialization:** converting an in-memory object to a transferable form such as JSON.
- **Orchestration:** coordinating multiple operations and passing results between them.
- **Hallucination:** plausible-sounding but unsupported or incorrect model output.
- **Prompt injection:** malicious or irrelevant instructions embedded in external content that try to override the agent's intended task.

## Official references

- [OpenAI Agents SDK quickstart](https://openai.github.io/openai-agents-python/quickstart/)
- [Agents and structured outputs](https://openai.github.io/openai-agents-python/agents/)
- [Running agents](https://openai.github.io/openai-agents-python/running_agents/)
- [Tools and `WebSearchTool`](https://openai.github.io/openai-agents-python/tools/)
- [Pydantic models](https://docs.pydantic.dev/latest/concepts/models/)
