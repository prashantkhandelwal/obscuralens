---
title: Obscura Webpage Summarizer
description: Summarize webpages with Obscura MCP, LangChain, and a local Ollama model.
---

## Overview

This agent asks a local Ollama model to summarize a webpage. The model uses
LangChain tools discovered from Obscura's MCP server to navigate to the URL and
read a text snapshot of the rendered page. It includes both a command-line
interface and a Svelte web application.

## Prerequisites

* Python 3.13 or newer
* [uv](https://docs.astral.sh/uv/)
* [Ollama](https://ollama.com/download)
* The `obscura` executable from the
	[Obscura releases](https://github.com/h4ckf0r0day/obscura/releases)

On Windows, extract the Obscura release archive and start its HTTP MCP server:

```powershell
obscura.exe mcp --http --port 8080
```

## Setup

Install the Python dependencies and download a tool-capable Ollama model:

```powershell
uv sync
ollama pull qwen3:0.6b
cd web
npm install
cd ..
```

Ollama normally starts with Windows. If it is not running, start it in another
terminal:

```powershell
ollama serve
```

## Usage

```powershell
uv run python main.py https://example.com
```

Choose another Obscura analysis mode with `--mode`, for example:

```powershell
uv run python main.py https://example.com --mode links
uv run python main.py https://example.com --mode assets
```

Assets mode uses Obscura's browser tools to list rendered scripts,
stylesheets, images, fonts, frames, media, embedded resources, and additional
URLs requested at runtime. It returns resource URLs and types without
downloading untrusted asset bodies.

Capture a screenshot from the CLI (saved as `screenshot.png` by default):

```powershell
uv run python main.py https://example.com --mode screenshot `
	--output page.png
```

Use a different model or MCP endpoint:

```powershell
uv run python main.py https://example.com `
	--model qwen3:0.6b `
	--mcp-url http://127.0.0.1:8080/mcp
```

Set `OLLAMA_MODEL`, `OLLAMA_NUM_CTX`, and `OBSCURA_MCP_URL` to change those
defaults without CLI flags. The default 32768-token Ollama context keeps long
page snapshots from losing their title and opening content. Use `--max-chars`
to control how much page text is sent to the model. To use stealth mode,
restart the server with `obscura.exe mcp --http --port 8080 --stealth`.

Obscura blocks private and local network targets by default as SSRF protection.
The agent intentionally preserves that default and is intended for public
`http://` and `https://` pages.

## How It Works

1. LangChain connects to the running Obscura HTTP MCP endpoint.
2. `MCPAdapter` discovers Obscura's browser tools.
3. The user selects an analysis mode in the web UI.
4. The agent calls `browser_navigate`, then the matching Obscura tool:
	`browser_snapshot`, `browser_evaluate`, or `browser_screenshot`.
   Links mode evaluates all `a[href]` elements directly so duplicate and
   special-purpose anchors are not removed by `browser_links`.
	Assets mode evaluates rendered resource elements and combines them with
	`browser_network_requests`, removing duplicate URLs.
5. The local model turns snapshot content into a Markdown summary. Links and
	asset inventories bypass the model; screenshots are returned directly.

Run `uv run python main.py --help` for all options.

## Web Application

Start the API from the project root:

```powershell
uv run uvicorn main:app --reload
```

In another terminal, start the Svelte development server:

```powershell
cd web
npm run dev
```

Open `http://localhost:5173`. Vite proxies `/api` requests to the API at
`http://127.0.0.1:8000`.

## Docker

Build the image from the project root:

```powershell
docker build -t obscuralens .
```

Ollama and Obscura run outside the container. On Docker Desktop, point the
container at the host services:

```powershell
docker run --rm -p 8000:8000 `
	-e OLLAMA_HOST=http://host.docker.internal:11434 `
	-e OBSCURA_MCP_URL=http://host.docker.internal:8080/mcp `
	obscuralens
```

Open `http://localhost:8000`. The image builds the Svelte application and
serves it from the same FastAPI process as the `/api` endpoint. The local
`.env`, virtual environment, Node modules, and Obscura binaries are excluded
from the image.
