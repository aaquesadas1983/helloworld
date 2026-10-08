# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A Python package of document-processing tools (e.g. converting DOCX/PDF to markdown) exposed to AI assistants through an MCP server built on `mcp.server.fastmcp.FastMCP`.

## Commands

```bash
uv venv --python 3.13 && source .venv/bin/activate   # see Python version note below
uv pip install -e .

uv run main.py      # start the MCP server (stdio transport; meant to be launched by an MCP client)
uv run pytest       # run all tests
uv run pytest tests/test_document.py::TestBinaryDocumentToMarkdown::test_binary_document_to_markdown_with_pdf   # single test
```

**Python version:** use Python 3.13 or lower. `uv.lock` pins `onnxruntime` 1.21.1 (pulled in by `markitdown`), which has no wheels for 3.14, so `uv run` fails on a 3.14 venv.

## Architecture

- `main.py` creates the `FastMCP("docs")` server and registers tools with `mcp.tool()(fn)`. A function in `tools/` is only exposed to clients once it is registered here.
- `tools/` contains plain Python functions, one module per area (`math.py`, `document.py`). FastMCP builds each tool's MCP description from the function's docstring and its input schema from the type hints and pydantic `Field` descriptions.
- `tools/document.py` wraps `markitdown`: it takes raw bytes and a file extension and returns markdown text. Its supported formats come from the `markitdown[docx,pdf]` extras in `pyproject.toml`.
- `tests/` exercises the tool functions directly, without going through MCP. Sample documents live in `tests/fixtures/` (`mcp_docs.docx`, `mcp_docs.pdf`).

## Code conventions

- Always apply appropriate type hints to function arguments.

## Defining MCP tools

Tools are plain Python functions. To add one:

1. Write the function in the right module under `tools/` (or create a new module).
2. Import it in `main.py` and register it with `mcp.tool()(my_function)`.
3. Add tests under `tests/` that call the function directly.

### Parameters

Annotate every parameter with a type hint and give it a default of `pydantic.Field(description=...)`. FastMCP reads these to build the tool's JSON input schema, so the description is what the AI client sees for each argument. Explain what the parameter means and what it does, not just its type.

```python
from pydantic import Field

def my_tool(
    param1: str = Field(description="Detailed description of this parameter"),
    param2: int = Field(description="Explain what this parameter does"),
) -> ReturnType:
    """Comprehensive docstring here"""
```

Because a `Field(...)` with no default makes the parameter required, these functions are meant to be called through MCP (or with every argument passed explicitly), not with Python-style defaults.

### Docstrings

The docstring becomes the tool description an AI client uses to decide whether and how to call the tool, so it must be complete. In this order, it should:

1. **Begin with a one-line summary.**
2. **Explain in detail what the tool does**, including what inputs it handles.
3. **Explain when to use the tool, and when not to.**
4. **Include usage examples with expected input and output.**

`tools/math.py:add` is the reference implementation:

```python
def add(
    a: float = Field(description="First number to add"),
    b: float = Field(description="Second number to add"),
) -> float:
    """Add two numbers together.

    Takes two numerical inputs and returns their sum. This tool handles
    integers and floating point numbers.

    When to use:
    - When you need to perform simple addition
    - When you need precise numerical calculation

    Examples:
    >>> add(2, 3)
    5.0
    >>> add(2.5, 3.5)
    6.0
    """
    return a + b
```

### Current gap

`tools/document.py:binary_document_to_markdown` doesn't meet these conventions yet. Its parameters have no `Field` descriptions, its docstring is a single line, and it isn't registered in `main.py`, so MCP clients can't call it.
