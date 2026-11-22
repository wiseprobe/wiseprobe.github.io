---
layout: post
title: 'Applying NVIDIA''s "You Don''t Need Long Context" to a Coding Agent on a 4B Model and a 6GB GPU'
categories: [Technology, AI]
tags: [ai agents, agent harness, local llms, small language models, ollama, patchpal]
---

Most coding agents assume a frontier model in the cloud. That isn't always an option: the code may be sensitive, API costs add up, or you want something that runs on the laptop in front of you. Small open models like Gemma 4 E4B and Qwen3.5 4B now handle tool calling well enough to be useful, and they run on modest hardware.

On a laptop with an NVIDIA RTX A1000 (6GB of VRAM), Gemma 4 E4B and Qwen3.5 4B served by Ollama both run entirely on the GPU with a 32K or even 16K context window, using about ~4GB of GPU memory and generating about 35 tokens per second. That window is a fraction of the 200K or more that agents are usually designed around.

[PatchPal](https://github.com/amaiya/patchpal) now includes **TinyAgent**, a version of its agent built for this setup Gemma4:E4B. This post explains how to run it, how it works, and what to expect from it.

## The Constraints of a Small GPU

Running an agent on a small GPU means working within two limits at once.

**A small context window.** Every request carries the system prompt, the tool definitions, and space reserved for the model's reply before any conversation is included. With PatchPal's full tool set, that overhead alone is over 9,000 tokens, more than half of a 16K window, and a single file read can take most of the rest. Standard compaction doesn't help much either. PatchPal's normal approach summarizes older messages and keeps the last two turns word for word, and in a 16K window two turns can be the whole conversation.

**A small model.** A 4B model follows instructions more literally than a frontier model, fills gaps in what it knows with plausible guesses, and loses detail when it summarizes. An agent built for it needs its instructions spelled out, and should rely as little as possible on the model remembering things.

Pointing an existing agent harness at a local model doesn't address either limit. Harnesses designed for frontier models assume a large window, and the problems show up as soon as the first large file is read. With a small model, the harness has to adapt instead: how often it compacts, how much one tool call can return, and how explicitly it instructs the model.

## You Don't Need Long Context

TinyAgent is based on a recent paper from NVIDIA Research , [You don't need long context](https://arxiv.org/html/2611.05306v1) (Belcak et al., 2026). The authors ran agents under hard context limits as small as 8K and found two practices that keep them working:

1. **Compact often, to a handoff note.** Instead of a summary of the history, the model writes a short brief for the work still ahead: the objective, what's been established, what's been changed, and what to do next. The conversation then restarts from that note.
2. **Read selectively.** Large inputs stay on disk and are read in small, bounded pieces instead of being loaded into the conversation.

With these, frontier models lost little or nothing. On SWE-bench Verified, Opus 4.5 scored 68.8% at a 16K limit and 74.8% at 24K, against 70.0% with full history. Open models were less robust, but still workable: Qwen 3.6 27B went from 68.6% to 65.0% at 16K. The open models in the paper are still much larger than the 4B model used here (27B to 230B total parameters), so that's the closest data point, not a direct one.

## Getting Started

TinyAgent is selected automatically when the context limit is 64K tokens or less. PatchPal can't see the window your server is actually using, so set `PATCHPAL_CONTEXT_LIMIT` to match it:

```bash
# Terminal 1: serve a 4B model with a 16K window
export OLLAMA_CONTEXT_LENGTH=16384
ollama serve

# Terminal 2: tell PatchPal the window size and run it
export PATCHPAL_CONTEXT_LIMIT=16384
patchpal --model ollama_chat/gemma4:e4b
# 🔧 Tiny agent (context window 16,384 ≤ 65,536): handoff compaction, selective reading
```

It's also available from Python. The context limit still has to match the server's window:

```python
import os
os.environ["PATCHPAL_CONTEXT_LIMIT"] = "16384"  # match OLLAMA_CONTEXT_LENGTH

from patchpal.agent import create_tiny_agent

agent = create_tiny_agent(model_id="ollama_chat/qwen3.5:4b")
agent.run("What does the retry logic in client.py do?")
```

To force it on or off regardless of window size, set `PATCHPAL_TINY_MODE=true` or `false`.

## An Example Session

Here is a real session with Qwen3.5 4B at 16K, asking it to read and summarize TinyAgent's own source file, about 500 lines long:

```
You: read and summarize patchpal/agent/tiny.py

📖 Reading: patchpal/agent/tiny.py
⚠️  Tool output truncated: 23,645 chars → 7,372 chars

Let me continue reading the rest of the file to get the complete picture:

📖 Reading lines 169-350: patchpal/agent/tiny.py
⚠️  Tool output truncated: 9,318 chars → 7,372 chars
↻ Empty response from model; retrying (1/2)...
📖 Reading lines 312-458: patchpal/agent/tiny.py
⚠️  Tool output truncated: 8,207 chars → 7,372 chars

⚠️  Context window at 83% capacity. Compacting (handoff)...
✓ Compaction complete. Saved 7,191 tokens (83% → 39%)

Agent:
  Summary of patchpal/agent/tiny.py

  Purpose: TinyAgent is a function-calling agent tuned for small context windows
  (≤ 64K tokens), based on research showing frontier models maintain task accuracy
  in 8K-48K windows when using frequent compaction to structured handoff notes and
  selective reading of inputs.

  Key differences from PatchPalAgent:
    Context management   Compacts conversation to [current request + handoff note]
    Tool output caps     ~15% of context window
    System prompt        Includes selective-reading guidance
    Default tools        Minimal set: read_file, read_lines, edit_file, write_file,
                         run_shell; + web tools if enabled
  ...
```

The file is three times larger than one read allows, so the first read was cut off with a hint saying where to continue. The model followed the hints through most of the file, recovered from one empty response, and compacted once at the end, taking usage from 83% to 39%. Its summary got the file's purpose, structure and mechanisms right, including the exact formula for the tool output cap.

It wasn't perfect, as described below.

### Writing Code

Reading a file is the easy case. Here is a coding task, run with Gemma 4 E4B at 32K on the same GPU (more on that model below). It asks for a script that lists the headings in PatchPal's 25 documentation files, which is harder than it sounds: the docs contain dozens of shell examples whose `# comments` look exactly like markdown headings. The prompt points the model at `docs/configuration.md`, the file with the most `#` comments inside code blocks, and asks for its headings so the results can be checked against the script's real output.

```
You: Write a Python script at scripts/extract_headings.py that extracts all headings and
subheadings from markdown files.

- Take a directory as an optional argument (default: docs) and search it recursively
  for .md files.
- For each file, print its path, then its headings indented by level (two spaces per
  level below #).
- Skip lines inside fenced code blocks (``` or ~~~), since shell comments there start
  with # but aren't headings. A block opened with ``` closes only with ```, and one
  opened with ~~~ closes only with ~~~.
- At the end, print the number of files and the total number of headings.
- Use only the Python standard library.

Test it on the existing files in docs; don't create or modify any files there.
docs/configuration.md has many # comments inside code blocks, so check that none
of them appear in its output. When you're done, show me the final counts and the
headings found for docs/configuration.md.

📝 Patching: scripts/extract_headings.py
⚡ Running: python3 scripts/extract_headings.py

Agent:
  The script scripts/extract_headings.py has been written and tested successfully on
  the docs directory.

  Overall Processing Summary:
    • Total files processed: 25
    • Total headings found: 331

  Headings found specifically for docs/configuration.md:
    Configuration
      Model Selection
        AWS Bedrock with SSO / ADFS
      Security & Permissions
        Maximum Security Mode
        Environment Variables
      Operational Controls
      Context Window Management
      ...
```

The model wrote the script and ran it in two steps, and its results were correct: 25 files and 331 headings, with none of the 67 `#` comments inside the code blocks of `configuration.md` counted as headings. A version of the same script without the code-block check finds 760.

Reading the code showed one gap. The script treats any fence line as opening or closing a code block, so it ignores the rule that a block opened with ```` ``` ```` closes only with ```` ``` ````. None of PatchPal's docs mix the two kinds of fence, so the results weren't affected, but the model said it had done everything asked. The output was right, and the claim was not. Tiny models are not perfect, but can still be incredibly useful in resource-constrained or air-gapped settings.

## How TinyAgent Works

TinyAgent is the standard PatchPal agent with four changes, each aimed at the small window, the small model, or both.

### 1. Handoff Compaction

When the context reaches 80% full, TinyAgent asks the model for a handoff note using a structured prompt adapted from the paper. The note records:

- The objective, and any instructions or preferences the user gave
- Established facts: exact paths, commands, values and errors
- Changes made, and whether they were verified
- Dead ends that shouldn't be repeated
- The remaining plan and the immediate next step
- Files and line ranges already read

The conversation is then replaced with just three things: the project memory (`MEMORY.md`), the current request with the note attached, and the model's last reply if the turn had finished.

The note is capped at about 10% of the window (around 1,600 tokens at 16K), so compaction always shrinks the context, however large the recent turns were. In the example session above, compaction took usage from 83% to 39%.

Nothing is lost permanently. Before each compaction, the full conversation is saved to a file under `~/.patchpal/`, and the note includes its path so the model can search it if it needs an earlier detail. The 100 most recent of these files are kept.

### 2. Tool Output Sized to the Window

Each tool result is limited to about 15% of the window, around 7,400 characters at 16K. When a file read is cut off, the result says exactly where it stopped and how to continue:

```
[Truncated after line 168 (limit: ~7,372 characters). To continue, call
read_lines(path='./patchpal/agent/tiny.py', start_line=169, end_line=315).
Read only the parts you need.]
```

A small model doesn't have to keep track of which ranges it has seen. It can work through a long file in order by following the hint.

### 3. A Prompt Written for Small Models

TinyAgent adds a short section to the system prompt. Because small models take instructions literally, it spells out things a larger model would infer:

- **The working directory**, so the model always knows where the project is
- **Exact tool names**: "use `run_shell` with grep", not "use grep", since there's no `grep` tool
- **The window size and read limit**, so the model knows how much one read can return
- **Local files first**: questions about the project are answered from its files, not web search

It also tells the model to locate code with `grep`, `wc -l` or `head` before reading it, to read small ranges with `read_lines` rather than whole files, not to reread what it has already seen, and to summarize or ask the user what to focus on when a request needs more than fits.

### 4. A Lean Tool Set

By default TinyAgent enables only the core tools: `read_file`, `read_lines`, `edit_file`, `write_file` and `run_shell`, plus `web_search` and `web_fetch` if web access is on. That cuts the tool definitions from about 5,900 tokens to 1,400. A fresh 16K session starts at about 30% usage, including the system prompt and the reserve for the model's reply, which leaves most of the window for work. Fewer tools also means fewer choices, which helps small models pick the right one.

## Built-In Safeguards

Small models fail in ways frontier models rarely do, so PatchPal guards against the most common ones.

**Runaway compaction.** Small models sometimes keep trying a request that can't fit in their window, compacting and restarting without making progress. TinyAgent stops a turn after five compactions without a new message from the user, and suggests a narrower request. Typing "continue" resumes with a fresh count, and `PATCHPAL_MAX_HANDOFFS_PER_TURN` changes the limit.

**Empty responses.** Local models occasionally return nothing at all: no text and no tool call. PatchPal retries the turn up to twice before giving up, dropping the empty reply so it doesn't clutter the history:

```
↻ Empty response from model; retrying (1/2)...
```

**Malformed tool calls.** When a tool call's arguments aren't valid JSON, PatchPal doesn't run it, tells the model how to fix it (for example, one `read_file` call per file), and keeps the stored history valid so later requests aren't affected.

**Harness bugs that look like model bugs.** Some failures that seemed to be the small model's fault weren't. With Ollama, LiteLLM numbers every streamed tool call 0, so when a model asks to read several files at once, any harness that groups streamed calls by number, as the OpenAI convention says to, merges them into one invalid call. The request then fails one turn later with a confusing `APIConnectionError`. It's a [known LiteLLM issue](https://github.com/BerriAI/litellm/issues/33678) that affects many agent harnesses, and it's fixed in recent LiteLLM releases. PatchPal now handles it regardless of LiteLLM version, by starting a new call whenever the call ID changes. If you're evaluating small models, check the harness before blaming the model.

## What to Expect From a 4B Model

TinyAgent makes a 6GB GPU usable, but it doesn't change what a small model can hold in mind. Each compaction keeps only what the model writes in its note, and a 4B model writes looser notes than a frontier model, so some detail is lost each time.

That makes a 4B model on a small GPU best suited to focused tasks:

- **Works well:** questions about a specific function, fixing a reported error, small edits, running and interpreting a command, writing new code files
- **Harder:** using a 16K context window to summarize or review large files, and changes that span many files

In the example session using a 16K context window, the model's summary of `tiny.py` got the structure and most details right, but several specific values were wrong: it said compaction triggers at 90% (it's 80%) and allows 10 handoffs per turn (it's 5). Those settings are defined in a different file the model never read, so it guessed plausible values instead of saying it didn't know. The lesson for small models: trust what they read, and verify specifics they didn't.

If you have memory to spare, spend it on a larger window first: 32K gives TinyAgent much more room between compactions. A larger model helps after that. On the same 6GB GPU, Gemma 4 E4B runs entirely on the GPU with a 32K window, using about 3.2GB of GPU memory and generating about 25 tokens per second. The main example in this post uses a 16K window even though 32K works, to show TinyAgent under tighter constraints.

Here is the same request with Gemma 4 E4B at 32K:

```
You: Read and summarize ./patchpal/agent/tiny.py

📖 Reading: ./patchpal/agent/tiny.py
⚠️  Tool output truncated: 23,045 chars → 14,745 chars

The file content was truncated, so I will read the rest of the file from line 341 to complete the summary.

📖 Reading lines 341-634: ./patchpal/agent/tiny.py

Agent:
  The file ./patchpal/agent/tiny.py defines TinyAgent, a specialized agent built on top of
  PatchPalAgent, specifically optimized for models with limited context windows (typically
  <= 64K tokens).
  ...
```

With Gemma 4 E4B at 32K, the same request took two reads instead of four, needed no compaction, and the summary contained no wrong values. Rather than guessing the compaction threshold and handoff limit, it named the settings without stating their values. The model changed as well as the window, so this isn't a controlled comparison, but it shows how much a 32K window helps on a 6GB GPU.

## Conclusion

A useful coding agent doesn't need a large GPU or a cloud API. A 4B model on a 6GB laptop GPU can answer questions about your code, fix errors and make small changes, as long as the harness is built for the constraints: compact early to a note about what's left to do, read only what's needed, keep the overhead lean, and spell things out for the model. The harness does the work a larger model would do for itself.

TinyAgent is available in [PatchPal](https://github.com/amaiya/patchpal). The [small context windows docs](https://amaiya.github.io/patchpal/context-management/#small-context-windows-tinyagent) cover all the settings.
