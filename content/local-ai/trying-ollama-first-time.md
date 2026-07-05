+++
date = '2026-07-05T12:19:48+10:00'
draft = true
title = 'Trying Ollama First Time'
posttype= 'blog'
+++


> **TL;DR**
> - Ollama lets you run large language models locally — no cloud, no API keys, no data leaving your machine.
> - I tried it on Windows with an NVIDIA RTX 5070 Ti (16 GB VRAM) and 64 GB RAM. The install took under 2 minutes.
> - Within 10 minutes I had a local AI chatbot running entirely on my GPU. Here is exactly what I did.

## Why I Wanted to Try This

I have been using cloud-based AI tools (ChatGPT, Gemini, Claude) for a while now, but as someone working in IT and security consulting, there is always data I am uncomfortable sending to the cloud — internal configs, Active Directory exports, client tenant details, vulnerability scan results.

I kept hearing about Ollama as the easiest way to run LLMs locally on your own hardware. No accounts, no subscriptions, no data leaving the machine. So I decided to give it a go.

## My Hardware

Before I started, here is what I am working with:

| Component | Spec |
| --- | --- |
| **GPU** | NVIDIA GeForce RTX 5070 Ti |
| **VRAM** | 16 GB GDDR7 |
| **RAM** | 64 GB DDR5 |
| **OS** | Windows 11 Pro |

From what I read beforehand, VRAM is the main bottleneck. The model needs to fit into VRAM for full GPU acceleration. If it does not fit, Ollama splits it between GPU and CPU, which still works but is slower.

### What Can 16 GB VRAM Actually Run?

I looked this up before downloading anything:

| Model Size | Fits in 16 GB VRAM? | Example Models |
| --- | --- | --- |
| **7B–8B** (Q4 quantised) | ✅ Yes — ~4–5 GB | `llama3.3`, `mistral`, `gemma3` |
| **13B–14B** (Q4 quantised) | ✅ Yes — ~7–8 GB | `llama3.3:14b`, `qwen3:14b` |
| **32B** (Q4 quantised) | ⚠️ Tight — ~18 GB (spills into RAM) | `qwen3:32b`, `command-r` |
| **70B+** | ❌ No — needs 35+ GB VRAM | `llama3.3:70b` |

So my sweet spot is **8B to 14B models**. Good to know.

## Step 1: Installing Ollama

I grabbed the installer from the official site:

```powershell
Start-Process "https://ollama.com/download/windows"
```

Ran the `.exe`, clicked through the installer. It added the `ollama` CLI to my system PATH and started running as a background service. No configuration, no environment variables, nothing.

I verified it worked:

```powershell
ollama --version
```

```
ollama version is 0.7.x
```

That was it. Under 2 minutes from download to ready.

## Step 2: Pulling My First Model

Ollama works like Docker — you `pull` a model once, and it gets cached locally for reuse.

I went with Meta's Llama 3.3 as my first model since it is one of the most popular open-source LLMs right now:

```powershell
ollama pull llama3.3
```

This downloaded about 4.9 GB. The model gets stored in `C:\Users\<username>\.ollama\models\`.

While I was at it, I also pulled a few others to compare later:

```powershell
# Google's Gemma 3 — smaller and faster
ollama pull gemma3:4b

# Mistral 7B — another popular general-purpose model
ollama pull mistral

# DeepSeek Coder — optimised for code generation
ollama pull deepseek-coder-v2:16b
```

## Step 3: My First Local AI Conversation

This was the moment of truth:

```powershell
ollama run llama3.3
```

A prompt appeared, and I typed my first question. The response came back in under a second. It was genuinely fast — noticeably faster than I expected for something running entirely on my local machine.

To exit the chat, you type `/bye`.

### Did It Actually Use My GPU?

I opened a second terminal while the model was running and checked:

```powershell
nvidia-smi
```

Sure enough, `ollama_llama_server` showed up in the process list, eating about 4.9 GB of my VRAM. The GPU was doing the work. If you run this and see minimal VRAM usage, something is wrong — the model is probably falling back to CPU.

## Step 4: Scripting Against It

The chat interface is cool, but the real value for me is being able to call it from PowerShell scripts. Ollama exposes a local REST API on `http://localhost:11434` while it is running.

### Basic API Call

```powershell
$body = @{
    model  = "llama3.3"
    prompt = "Explain the difference between RBAC and ABAC in two sentences."
    stream = $false
} | ConvertTo-Json

$response = Invoke-RestMethod -Uri "http://localhost:11434/api/generate" `
    -Method Post `
    -Body $body `
    -ContentType "application/json"

$response.response
```

This worked first try. The response came back as a PowerShell object — no parsing needed.

### Summarising a Log File

This is the use case I was most excited about — feeding a local log file to the model without it ever touching the internet:

```powershell
$logContent = Get-Content -Path "C:\Logs\security-audit.log" -Raw -ErrorAction Stop

$body = @{
    model  = "llama3.3"
    prompt = "Summarise the following security audit log. Highlight any failed login attempts, privilege escalations, or unusual access patterns:`n`n$logContent"
    stream = $false
} | ConvertTo-Json -Depth 3

$response = Invoke-RestMethod -Uri "http://localhost:11434/api/generate" `
    -Method Post `
    -Body $body `
    -ContentType "application/json"

Write-Output $response.response
```

### Explaining a PowerShell Error

I also tried piping a caught error directly to the model to see if it could explain what went wrong:

```powershell
try {
    Get-ADUser -Identity "nonexistent.user" -ErrorAction Stop
} catch {
    $errorMessage = $_.Exception.Message

    $body = @{
        model  = "llama3.3"
        prompt = "I got this PowerShell error. Explain what caused it and how to fix it:`n`n$errorMessage"
        stream = $false
    } | ConvertTo-Json

    $response = Invoke-RestMethod -Uri "http://localhost:11434/api/generate" `
        -Method Post `
        -Body $body `
        -ContentType "application/json"

    Write-Output $response.response
}
```

This actually gave a surprisingly useful explanation. Not perfect, but good enough to point me in the right direction.

## Step 5: Managing Models

A few commands I found useful for housekeeping:

```powershell
# List all downloaded models
ollama list

# Show model details (size, quantisation, parameters)
ollama show llama3.3

# Delete a model I no longer need
ollama rm mistral

# Check what is currently loaded in memory
ollama ps
```

## Performance: What I Saw

Here is what I observed on my RTX 5070 Ti. These are rough numbers from my own testing, not official benchmarks:

**Llama 3.3 (8B Q4):**

| Metric | Value |
| --- | --- |
| **First token latency** | ~0.5 seconds |
| **Token generation speed** | ~80–100 tokens/second |
| **VRAM usage** | ~4.9 GB |
| **RAM usage** | ~1.2 GB |

**Qwen 3 (14B Q4):**

| Metric | Value |
| --- | --- |
| **First token latency** | ~1 second |
| **Token generation speed** | ~40–55 tokens/second |
| **VRAM usage** | ~8.5 GB |
| **RAM usage** | ~1.5 GB |

The 8B model felt instant. The 14B model had a slight pause before responding but the quality of the answers was noticeably better — especially for code generation and multi-step reasoning.

## First Impressions

**What surprised me:**
- How easy the setup was. I expected driver issues, CUDA configuration, Python dependencies — none of that. It just worked.
- The speed. Running on the GPU, the 8B model responds faster than some cloud APIs I have used.
- The scripting potential. Being able to call a local LLM from `Invoke-RestMethod` opens up a lot of automation possibilities.

**What I want to explore next:**
- **Open WebUI** — a self-hosted ChatGPT-style web interface that connects to Ollama. Gives you a browser-based chat with conversation history and model switching.
- **Obsidian integration** — connecting the local model to my Obsidian vault for private, offline note querying.
- **RAG pipeline** — building a retrieval-augmented generation setup to query my SANS course materials and home lab documentation locally.

## Wrapping Up

| Command | What It Does |
| --- | --- |
| `ollama pull <model>` | Download a model to your local machine |
| `ollama run <model>` | Start an interactive chat session |
| `ollama list` | Show all downloaded models |
| `ollama show <model>` | Display model metadata and parameters |
| `ollama ps` | Show models currently loaded in memory |
| `ollama rm <model>` | Delete a downloaded model |
| `nvidia-smi` | Verify GPU acceleration is active |
| `Invoke-RestMethod` to `localhost:11434` | Script against the local API |
