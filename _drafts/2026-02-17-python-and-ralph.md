---
layout: post
title: "The Unreasonable Effectiveness of a Python API to a Coding Agent"
categories: [Technology, AI]
tags: [ai agents, python, ralph wiggum, autopilot, api, patchpal]
---

<img src="/images/posts/patchpal/ralph-wiggum.jpg" alt="Ralph Wiggum" width="400"/>

> "I'm learnding!" - Ralph Wiggum

**Cross-posted from X / @amaiya**

In fact the only thing that changed was having a Python API. That's it.

I maintain a little "hobby project", PatchPal, an open-source coding agent with a Python API. I've so far authored ~1,300 commits, mostly playing around and making incremental improvements here and there when I see a pain point.

Why bother, you ask? Because having a programmatic API changes everything.

## 0x0: The Wrong Question

The conversation right now is almost entirely about which coding agent is best, Windsurf or Cursor, Claude Code or Copilot. This framing misses the point because it treats the agent as a black box you interact with through a chat interface, when in reality the bottleneck is something much more basic: **you can't programmatically control it**.

When your coding agent is locked behind a proprietary UI, you get exactly one workflow: type prompt, wait, review changes, repeat. No composition. No automation. No integration with your actual development process.

The Ralph Wiggum technique—where an agent iterates autonomously until completion—demonstrates this perfectly. Geoffrey Huntley [pioneered the approach](https://ghuntley.com/ralph/) in July 2025 with a 5-line bash script:

```bash
while :; do 
  cat PROMPT.md | claude-code
done
```

That's it. A bash loop that feeds the same prompt to Claude Code repeatedly. It worked surprisingly well—bash gave you just enough programmatic control to check tests after each iteration and continue until everything passed.

By December 2025, Anthropic had formalized it into an [official plugin](https://github.com/anthropics/claude-code/tree/main/plugins/ralph-wiggum) with "Stop Hooks" that intercept Claude's exit internally. Cleaner UX, but now the loop is opaque—hidden in a markdown state file, sensitive to permissions, easy to break. As Dexter Horthy [discovered](https://www.dreamhost.com/blog/ralph-wiggum/), it "dies in cryptic ways unless you use `--dangerously-skip-permissions`."

Both approaches work for the basic loop. But both are fundamentally limited:

**Bash scripts** give you control but no state. You can't inspect the agent's message history, track token usage, or access cost data. You pipe text in, get text out, and that's it. Want to switch models mid-session? Switch to a cheaper model after the first attempt? Implement a budget limit? You're writing brittle shell parsing code or just can't do it.

**Plugins** give you integration but no composability. The loop runs inside Claude Code's process, so you can't import it into a Python script, call it from a Jupyter notebook, or integrate it into CI/CD. The abstraction is opaque—when something breaks, you're debugging markdown state files and permission flags, not looking at clear code.

This is the API problem. Not "which model is smarter" but "can you programmatically compose agents into real workflows?"

## 0x1: What Is Ralph?

Before I explain why a Python API matters, it's worth understanding the technique.

The Ralph Wiggum technique is deceptively simple:

```
1. Agent works on task
2. Agent tries to exit  
3. Stop hook intercepts ← The key!
4. Same prompt fed back
5. Agent sees previous work in conversation history
6. Agent adjusts approach
7. Repeat until completion promise found
```

The agent never actually "completes"—every time it tries to return, you check for a completion promise (a specific string like `<promise>COMPLETE</promise>`). If not found, you feed the same prompt back. The agent sees its previous work, notices failing tests, spots bugs, and refines iteratively.

It's surprisingly effective. Tasks that fail on first attempt often succeed after 5-10 iterations as the agent debugs its own output.

But here's the catch: **this only works if you can intercept the agent's output and programmatically feed prompts back**. You need an API.

Claude Code implements this internally. Great—if you use Anthropic's models exclusively and don't need to modify the behavior. What if you want to:

- Use a local model (zero API cost)?
- Run multiple sequential phases with different completion criteria?
- Integrate Ralph into CI/CD to auto-fix failing tests?
- Add custom logic between iterations?
- Track costs and context usage programmatically?

You're out of luck. No API means no customization.

## 0x2: ralph.py

Now bear with me here. What if the entire Ralph technique was just a Python function you could import?

```python
#!/usr/bin/env python3
"""ralph.py - Autonomous coding via iterative refinement"""

from patchpal.agent import create_agent

def ralph_loop(prompt: str, completion_promise: str, max_iterations: int = 50):
    """
    The stop hook pattern: agent never actually completes until it outputs
    the completion promise. Each iteration, agent sees previous work in history.
    """
    agent = create_agent()
    
    for iteration in range(max_iterations):
        print(f"🔄 Ralph Iteration {iteration + 1}/{max_iterations}")
        
        # Same prompt every time - agent's message history accumulates
        response = agent.run(prompt)
        
        # Check for completion promise
        if completion_promise in response:
            print(f"✅ COMPLETION after {iteration + 1} iterations!")
            print(f"Total cost: ${agent.cumulative_cost:.4f}")
            return response
        
        # Stop hook: no completion found, continue
        print("⚠️ No completion promise. Continuing...")
    
    print(f"⚠️ Max iterations reached")
    return None
```

The core loop is ~20 lines. Run it from the command line:

```bash
python ralph.py --prompt "Build a REST API with tests. Run pytest. Fix failures. Output: <promise>COMPLETE</promise>" --completion-promise "COMPLETE" --max-iterations 30
```

Or import it into your own code:

```python
from ralph import ralph_loop

# In a CI pipeline
ralph_loop(prompt=f"Fix failing tests: {test_output}", completion_promise="FIXED")

# In a Jupyter notebook  
ralph_loop(prompt="Analyze data.csv and create plots", completion_promise="DONE")

# With custom logic
agent = create_agent()
response = ralph_loop(prompt, completion_promise)
if agent.cumulative_cost > 10.0:
    alert_team("Ralph exceeded budget")
```

This is what a programmatic API enables. Not "can you make it loop"—bash does that. Not "can you integrate with one tool"—plugins do that. But "can you compose the agent into arbitrary workflows using actual code?"

### The Difference

Let's compare the three approaches:

| Capability | Bash Script | Claude Plugin | Python API |
|-----------|-------------|---------------|------------|
| **Basic loop** | ✅ Yes | ✅ Yes | ✅ Yes |
| **Inspect agent state** | ❌ No | ❌ Opaque | ✅ `agent.messages` |
| **Track costs** | ❌ Parse logs | ❌ External | ✅ `agent.cumulative_cost` |
| **Switch models mid-session** | ❌ Restart process | ❌ Not supported | ✅ `create_agent(model=...)` |
| **Budget limits** | ❌ Shell math | ❌ Not supported | ✅ `if cost > limit: stop` |
| **Multi-phase workflows** | ⚠️ Multiple scripts | ❌ Manual | ✅ Sequential function calls |
| **Import into Python** | ❌ Subprocess | ❌ No | ✅ `from ralph import ralph_loop` |
| **CI/CD integration** | ⚠️ Shell script | ❌ No | ✅ Native Python |
| **Jupyter notebooks** | ❌ No | ❌ No | ✅ Import and run |
| **Error handling** | ⚠️ Exit codes | ❌ Opaque | ✅ Try/catch, inspect state |
| **Debugging** | ⚠️ Parse stdout | ❌ Markdown files | ✅ Python debugger |
| **Model agnostic** | ⚠️ One CLI tool | ❌ Claude only | ✅ Any LiteLLM model |

The bash script works for the basic case. The plugin is cleaner if you only use Claude Code interactively. But neither gives you **composability**—the ability to treat the agent as a library you can build on top of.

A Python API (or any proper programmatic API) makes the agent a **primitive**. You can inspect its state, control its execution, compose multiple agents, integrate into existing tools, and add arbitrary logic. You're not fighting with shell parsing or reverse-engineering markdown files—you're writing normal code.

## 0x3: The Benchmark

Since my primary concern was practical utility, I tested the three implementations across real scenarios:

**Test Setup:**
- Task: Build a Flask TODO API with CRUD endpoints, input validation, and pytest tests
- Success: All tests pass
- Implementations: Bash script (Huntley's original), Claude plugin (Anthropic's official), Python API (ralph.py)

**Scenario 1: Basic Ralph Loop**

All three work. The bash script pipes prompts in a loop, the plugin uses stop hooks, the Python API calls `ralph_loop()`. Results are roughly equivalent—7-8 iterations, ~$1.15 in API costs with Claude Sonnet 4.5, complete TODO API with passing tests.

**Scenario 2: Multi-Phase Development**

Build the API in three sequential phases with different completion criteria:

```python
# Phase 1: Core CRUD
ralph_loop(
    prompt="Build Flask API with CRUD endpoints. Tests must pass. Output: PHASE1_COMPLETE",
    completion_promise="PHASE1_COMPLETE",
    max_iterations=20
)

# Phase 2: Add validation
ralph_loop(
    prompt="Add input validation to existing API. Tests must pass. Output: PHASE2_COMPLETE", 
    completion_promise="PHASE2_COMPLETE",
    max_iterations=15
)

# Phase 3: Documentation
ralph_loop(
    prompt="Create README with API docs and examples. Output: PHASE3_COMPLETE",
    completion_promise="PHASE3_COMPLETE",
    max_iterations=5
)
```

**Results:**
- Bash script: ⚠️ Requires three separate script files, manual coordination, can't share state
- Claude plugin: ❌ Not possible—plugin runs interactively, no way to chain phases programmatically
- Python API: ✅ Works perfectly—sequential function calls, shared context, each phase builds on previous

**Scenario 3: CI/CD Integration**

Integrate Ralph into a CI pipeline to auto-fix failing tests:

```python
# In .github/workflows/test.yml or similar
if tests_failed():
    ralph_loop(
        prompt=f"Fix failing tests: {get_test_output()}. Output: TESTS_FIXED",
        completion_promise="TESTS_FIXED",
        max_iterations=10
    )
```

**Results:**
- Bash script: ⚠️ Can call from CI but no access to test output, costs, or agent state—just pipes text
- Claude plugin: ❌ Not possible—plugin runs in interactive Claude Code session, can't be invoked from CI
- Python API: ✅ Works perfectly—native Python integration, full access to CI context and agent state

**Scenario 4: Cost-Optimized Development**

Use an expensive model for the initial attempt, then switch to a local model for iteration:

```python
# First attempt with GPT-5.2
agent = create_agent(model="openai/gpt-5.2")
response = agent.run(prompt)

if not is_complete(response):
    # Switch to free local model for iteration
    agent = create_agent(model="hosted_vllm/openai/gpt-oss-20b")
    ralph_loop(agent, prompt, completion_promise, max_iterations=20)
```

**Results:**
- Bash script: ❌ Can't switch models mid-session—would need to restart with different CLI tool
- Claude plugin: ❌ Not supported—locked to Claude, can't switch models
- Python API: ✅ Works perfectly—programmatic model switching, cost tracking, budget enforcement

### Summary

| Scenario | Bash Script | Claude Plugin | Python API |
|----------|-------------|---------------|------------|
| Basic Ralph | ✅ ~$1.20, 8 iter | ✅ ~$1.20, 8 iter | ✅ ~$1.15, 7 iter |
| Multi-Phase | ⚠️ 3 scripts, manual | ❌ Not possible | ✅ Works perfectly |
| CI/CD Integration | ⚠️ Text pipes only | ❌ Not possible | ✅ Works perfectly |
| Cost Optimization | ❌ Can't switch models | ❌ Claude only | ✅ Works perfectly |
| Local Models (vLLM) | ⚠️ If CLI exists | ❌ Not supported | ✅ $0.00 cost |

The basic Ralph loop works in all three. But any real-world workflow—composition, integration, cost control, model flexibility—requires a programmatic API. Not "harder with bash"—literally impossible. You can't compose shell scripts into Jupyter notebooks. You can't import a plugin into CI/CD. You can't inspect agent state through text pipes.

## 0x4: So What?

Having a Python API means the agent becomes a **composable primitive** instead of a monolithic application.

Three examples that became trivial once I had `ralph.py`:

**Example 1: Jupyter Notebook Development**
```python
# In a Jupyter cell
from ralph import ralph_loop

# Analyze dataset
ralph_loop(
    prompt="Load data.csv, generate summary statistics, create visualizations. Output: ANALYSIS_COMPLETE",
    completion_promise="ANALYSIS_COMPLETE",
    max_iterations=10
)

# Shows plots inline, updates as agent iterates
```

**Example 2: Automated Code Review Fixes**
```python
# post-review.py - run after code review
review_comments = load_review_comments()

for comment in review_comments:
    ralph_loop(
        prompt=f"Address review comment: {comment.text} in {comment.file}:{comment.line}. Output: FIXED",
        completion_promise="FIXED",
        max_iterations=5
    )
```

**Example 3: Cost-Optimized Development**
```python
# Use expensive model for first attempt, cheap model for iterations
agent = create_agent(model="anthropic/claude-sonnet-4-5")
response = agent.run(prompt)

if not is_complete(response):
    # Switch to local model for refinement iterations
    agent = create_agent(model="hosted_vllm/openai/gpt-oss-20b")
    ralph_loop(prompt, completion_promise, max_iterations=20)
```

None of this is hypothetical. These are scripts I run daily. They work because the agent is a Python library I can import and compose.

## 0x5: The Vendors

Anthropic recently [blocked OpenCode](https://github.com/OpenCodeInterpreter/OpenCodeInterpreter/issues/99), a popular open-source agent, from accessing Claude through Claude Code subscriptions. Their position—"OpenCode reverse-engineered a private API"—is technically fair. Their infrastructure, their rules.

But look at what the action signals: **Don't build programmatic tools. Use our chat interface.**

This is backwards. OpenCode had 8,000+ GitHub stars because developers wanted programmatic access. They wanted to integrate Claude into their workflows, not adopt Claude Code's workflow. Blocking them doesn't make the demand disappear—it just means developers use OpenAI or vLLM instead.

Here's why this matters: I just showed that `ralph.py` enables workflows literally impossible through a UI. Multi-phase development, CI/CD integration, cost optimization, custom orchestration—these aren't edge cases. They're how professional developers actually work.

No vendor will build API-first experiences for competitors' models. Anthropic won't optimize for OpenAI. OpenAI won't optimize for Gemini. But an open-source Python agent with a clean API works with all of them, because contributors use different models and fix what they personally need.

**The model is the moat. The API is the bridge.** Burning bridges just means fewer people bother to cross.

## 0x6: The Implementation

The full `ralph.py` is ~200 lines with argument parsing and logging. The core loop is genuinely ~20 lines:

```python
def ralph_loop(prompt: str, completion_promise: str, max_iterations: int = 50, model: str = None):
    """The stop hook pattern."""
    os.environ["PATCHPAL_REQUIRE_PERMISSION"] = "false"  # Autonomous mode
    
    agent = create_agent(model_id=model)
    
    for iteration in range(max_iterations):
        print(f"🔄 Ralph Iteration {iteration + 1}/{max_iterations}")
        
        response = agent.run(prompt)
        
        if completion_promise in response:
            print(f"✅ COMPLETION after {iteration + 1} iterations!")
            print(f"Total cost: ${agent.cumulative_cost:.4f}")
            return response
        
        print("⚠️ No completion promise detected. Continuing...")
        print(f"Context usage: {get_context_usage(agent)}%")
    
    print(f"⚠️ Max iterations reached")
    return None
```

Key features that require API access:

1. **Conversation history preservation** - `agent.messages` accumulates across iterations
2. **Cost tracking** - `agent.cumulative_cost` tracks spending in real-time
3. **Context management** - `agent.context_manager` handles automatic compaction
4. **Model switching** - Can swap models mid-session
5. **State inspection** - Access tokens, message count, tool calls at any point

Try implementing any of that through a web UI. You can't. The API isn't a "nice to have"—it's the difference between "can use the technique" and "can't use the technique."

## 0x7: Cost Control

Running 30+ iterations with Claude Sonnet 4.5 can get expensive. With a Python API, you have options:

**Option 1: Local vLLM (Zero Cost)**
```bash
# Terminal 1: Start vLLM server once
vllm serve openai/gpt-oss-20b --tool-call-parser openai --enable-auto-tool-choice

# Terminal 2: Unlimited Ralph iterations at zero cost  
python ralph.py --model hosted_vllm/openai/gpt-oss-20b --prompt "..." --completion-promise "COMPLETE" --max-iterations 100
```

**Option 2: Hybrid Approach**
```python
# Use GPT-5.2 for complex reasoning, local model for iteration
agent = create_agent(model="openai/gpt-5.2")
initial_response = agent.run(prompt)

if not is_complete(initial_response):
    # Switch to free local model
    agent = create_agent(model="hosted_vllm/openai/gpt-oss-20b")
    ralph_loop(agent, prompt, completion_promise)
```

**Option 3: Budget Limits**
```python
def ralph_loop_with_budget(prompt, completion_promise, max_cost=5.0):
    agent = create_agent()
    
    for iteration in range(100):
        if agent.cumulative_cost > max_cost:
            print(f"⚠️ Budget limit reached: ${agent.cumulative_cost:.2f}")
            return None
        
        response = agent.run(prompt)
        if completion_promise in response:
            return response
```

These optimizations require programmatic access to cost tracking and model configuration. Impossible through a UI.

## 0x8: Real Example

Here's an actual session from yesterday:

```bash
$ python ralph.py --prompt "Build Flask TODO API. Must have: GET/POST/PUT/DELETE endpoints, input validation, tests with pytest. Run tests. Fix failures until all pass. Output: COMPLETE" --completion-promise "COMPLETE" --max-iterations 20

🔄 Ralph Iteration 1/20
[Agent creates app.py and test_app.py, runs pytest]
FAILED test_app.py::test_get_todos - AssertionError: expected 200, got 404
FAILED test_app.py::test_create_todo - KeyError: 'title'
⚠️ No completion promise detected. Continuing...

🔄 Ralph Iteration 2/20  
[Agent reviews failures in message history, fixes route and validation]
FAILED test_app.py::test_delete_todo - KeyError: '3'
⚠️ No completion promise detected. Continuing...

🔄 Ralph Iteration 3/20
[Agent fixes last bug, re-runs tests]
test_app.py::test_get_todos PASSED
test_app.py::test_create_todo PASSED  
test_app.py::test_update_todo PASSED
test_app.py::test_delete_todo PASSED

All tests passing! <promise>COMPLETE</promise>

✅ COMPLETION after 3 iterations!
Total cost: $0.73
```

The agent created code, ran tests, saw failures, debugged, fixed issues, verified with tests, and confirmed completion—completely autonomously. The Python API made this possible by preserving conversation history and enabling the stop hook pattern.

## 0x9: Why This Matters

The API problem is real, measurable, and it's blocking entire categories of workflows. The gap between "cool demo" and "useful tool" isn't model intelligence. It's whether you can programmatically compose the agent into your actual development process.

**The harness problem** (as Can Bölük called it) is about how agents express edits. **The API problem** is about whether you can programmatically orchestrate agents at all.

Ralph demonstrates this perfectly: the technique is powerful, but you can only use it if you have API access. Claude Code implements Ralph internally—great for Anthropic users. But developers who want to:

- Use different models (cheaper, local, or just different capabilities)
- Compose multiple agents into pipelines  
- Integrate agents into existing tools (CI/CD, notebooks, scripts)
- Add custom logic (budget limits, logging, monitoring)

...are locked out. Not because the technique is hard—the implementation is literally 20 lines—but because they don't have an API.

The API problem will be solved. The question is whether it gets solved by one company, in private, for one model—or by a community, in the open, for all of them.

## Getting Started

```bash
pip install patchpal

# Download ralph.py
curl -O https://raw.githubusercontent.com/amaiya/patchpal/main/examples/ralph/ralph.py

# Try it
python ralph.py --prompt "Build a simple Flask hello world app with a test" --completion-promise "COMPLETE" --max-iterations 10

# Or use as a library
python -c "from patchpal.autopilot import autopilot_loop; autopilot_loop(prompt='...', completion_promise='COMPLETE')"
```

All code, benchmarks, and examples: [github.com/amaiya/patchpal](https://github.com/amaiya/patchpal)

---

**Update (Feb 16, 2026):** Added examples of multi-phase development and CI/CD integration based on reader feedback.
