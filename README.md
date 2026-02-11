# AI-Assisted Development: A Practical Guide

Thoughts on incorporating AI coding agents into real engineering workflows and what I learned so far.

I'm currently introducing AI-assisted development at the company I work at. This repository is my attempt to organize what I know, what I'm still figuring out, and what resources helped me the most.

## Table of Contents

- [Why Should You Care](#why-should-you-care)
- [The Risks Are Real](#the-risks-are-real)
- [Engineering Skills Still Matter](#engineering-skills-still-matter)
- [Understanding LLMs](#understanding-llms)
- [How Agents Actually Work](#how-agents-actually-work)
- [Context Management Is Everything](#context-management-is-everything)
- [Writing Good Prompts for Existing Agents](#writing-good-prompts-for-existing-agents)
- [What AI Makes Possible](#what-ai-makes-possible)
- [Credits and References](#credits-and-references)
- [License](#license)

## Why Should You Care

Nolan Lawson [wrote a great piece](https://nolanlawson.com/2026/02/07/we-mourn-our-craft/) about the emotional side of this shift. He described it honestly: "The worst fact about these tools is that they work." He is not celebrating the new world, but he is also not resisting it.

Whether you like it or not, AI coding tools are changing how we work. Your junior colleagues are already using Cursor, Claude Code, Copilot. They write code faster. Not always better, but faster. And the tools keep improving.

The question is not "should I use AI for coding?" anymore. The question is: how do I use it without making a mess?

## The Risks Are Real

Jake Nations wrote about this in [Vibe Coding Our Way to Disaster](https://www.arthropod.software/p/vibe-coding-our-way-to-disaster). His argument is based on Rich Hickey's ideas about simplicity vs. ease. The short version: vibe coding (just chatting with AI and letting it write whatever) is choosing ease over simplicity. It feels productive but creates tangled, complex systems.

### Vibe Coding vs. Disciplined AI Coding

```
VIBE CODING (ease)                         DISCIPLINED AI CODING (simplicity)

You: "make a login page"                   You: research auth flow in our codebase
  AI writes 200 lines                        AI maps existing patterns
You: "it doesn't work, fix it"              You: review research, plan approach
  AI rewrites 150 lines                      AI creates implementation plan
You: "now add validation"                  You: review plan, approve
  AI patches on top of patches               AI implements following the plan
You: "why is everything broken?"           You: review code, run tests
  AI apologizes, rewrites again              Working code that fits the codebase

Result: tangled mess of corrections        Result: clean code that follows
        buried in context                          existing patterns
```

The key problems with naive AI coding:

- **Context complexity becomes code complexity.** When you have long conversations with AI, corrections and clarifications pile up. The AI starts making connections between unrelated parts of the conversation. Your code becomes a reflection of that mess.
- **AI amplifies your approach.** If you rush to code without understanding the problem, AI helps you build the wrong thing faster. If you think first, AI becomes a powerful implementation tool.
- **Most critical bugs come from misunderstanding the problem, not from implementation errors.** This was true before AI, and it is even more true now when AI can generate hundreds of lines of code from a vague prompt.
- **The Stanford study** found that AI tools often lead to rework. Code shipped with AI one week gets rewritten next week. In large established codebases, AI can actually make developers less productive.

This is not a reason to avoid AI tools. It is a reason to use them with discipline.

## Engineering Skills Still Matter

AI does not replace the need to understand your system. You still need to:

- Know how your codebase works before asking AI to change it
- Review generated code with the same rigor as human-written code
- Design systems that are simple, not just easy to generate
- Understand when the AI is wrong (and it will be wrong sometimes)

As Dex Horthy from [HumanLayer](https://humanlayer.dev) puts it in the [12-Factor Agents](https://github.com/humanlayer/12-factor-agents) guide: the best production AI agents are "comprised of mostly just software." The LLM is a powerful component, but the engineering around it is what makes it reliable.

### The Leverage Pyramid

Where you spend your human attention matters. A mistake at the research level cascades into everything below it.

```
                    /\
                   /  \          A bad line of RESEARCH
                  / re \         = misunderstanding the codebase
                 / search\       = thousands of bad lines of code
                /----------\
               /            \    A bad line of a PLAN
              /    plan      \   = wrong approach
             /                \  = hundreds of bad lines of code
            /------------------\
           /                    \
          /   implementation     \  A bad line of CODE
         /                        \ = a bad line of code
        /__________________________\

        HUMAN EFFORT GOES HERE ^^^
        (review research and plans, not just code)
```

You need to be able to read the research AI produces and tell when it is wrong. You need to be able to look at a plan and spot the flaw. The human review at research and planning stages is the highest-leverage intervention in the whole process.

## Understanding LLMs

Before we talk about agents, it helps to understand what an LLM actually is.

### LLM Is a Stateless Function

An LLM is a function. You give it text, it gives you text back. That is it.

```
f(input_text) → output_text
```

There is no memory between calls. There is no hidden state. Every time you send a message, the model sees the entire conversation from scratch. What feels like a "conversation" is actually your client re-sending the full history every single time.

```
Call 1:  f("What is 2+2?")                                         → "4"
Call 2:  f("What is 2+2?" + "4" + "Now multiply by 3")             → "12"
Call 3:  f("What is 2+2?" + "4" + "Now multiply by 3" + "12"
           + "What was the original number?")                       → "4"
```

The model did not "remember" that the original number was 4. It saw the full conversation in the input and found the answer there. If you removed the earlier messages, it would have no idea.

This has practical consequences:

- **Context is everything.** The model only knows what you put in the input. If you do not include it, it does not exist.
- **Longer conversations degrade.** Every message adds tokens. At some point the input is so large that the model loses focus on what matters.
- **You pay for every token, every time.** The full conversation is re-sent on each call. A 50-message conversation means message 1 has been sent 50 times.

### What the Model Actually Sees

When you type a message in ChatGPT or Claude, it looks like a simple chat. Behind the scenes, the API call looks more like this:

```
┌─────────────────────────────────────────────────────────┐
│  API Call                                               │
│                                                         │
│  messages: [                                            │
│    { role: "system",    content: "You are a helpful..." │
│    },                                                   │
│    { role: "user",      content: "What is 2+2?"        │
│    },                                                   │
│    { role: "assistant", content: "4"                    │
│    },                                                   │
│    { role: "user",      content: "Now multiply by 3"   │
│    }                                                    │
│  ]                                                      │
│                                                         │
│  → model reads ALL of this, generates the next response │
└─────────────────────────────────────────────────────────┘
```

The model does not have a session. It does not "know" it already answered the first question. It receives the entire list of messages and produces the next one. The chat interface is an illusion maintained by the client.

### Temperature: Controlled Randomness

You may have heard that LLMs are "non-deterministic." This is half true. The randomness is a design choice, not a flaw.

At each step, the model predicts the probability of every possible next token. Temperature controls how it picks from those probabilities:

```
Prompt: "The capital of France is"

Token probabilities:
  "Paris"    → 92%
  "Lyon"     → 3%
  "a"        → 2%
  "the"      → 1%
  ...

Temperature = 0:    Always picks "Paris" (highest probability)
Temperature = 0.7:  Usually picks "Paris", sometimes surprises
Temperature = 1.0:  More random, might pick "Lyon" or "a"
```

For coding tasks, lower temperature is almost always better. You want predictable, correct output, not creative variation. Most coding agents run at low temperature by default.

### Tokens, Not Characters

LLMs do not read characters or words. They read tokens. A token is roughly 3-4 characters in English, but it varies.

```
"Hello, world!"       → ["Hello", ",", " world", "!"]           = 4 tokens
"def fibonacci(n):"   → ["def", " fibon", "acci", "(n", "):"]   = 5 tokens
"東京"                 → ["東", "京"]                              = 2 tokens
```

This matters because:

- **Context windows are measured in tokens.** When Claude says 200k context, that is 200k tokens, not characters. Roughly 150k words, or about 500 pages of text.
- **You pay per token.** Both input and output. Reading a 5000-line file costs more than reading a 100-line file.
- **Code is token-expensive.** Variable names, syntax, and whitespace all consume tokens. A 200-line function might cost more tokens than a 200-word paragraph.

### Why This Matters for Agents

Everything in the rest of this article builds on these basics:

- **Agents are loops around a stateless function.** Every iteration, the agent re-sends the conversation plus new tool results. The context grows with every step.
- **Context management is the core skill.** Since the model only sees what you send it, managing what goes into the context window is how you control the output quality.
- **Fresh context beats long conversations.** Splitting work into research/plan/implement phases works because each phase starts with a clean, focused input instead of a bloated conversation history.
- **Sub-agents protect the main context.** When a sub-agent does 15 grep searches, all that noise stays in the sub-agent's context. The main agent only sees a compact summary.

If you remember one thing from this section: the LLM does not know anything you did not tell it. Everything else follows from that.

## How Agents Actually Work

I did an internal presentation at my company about how to write good agents, based on the [12-Factor Agents](https://github.com/humanlayer/12-factor-agents) series by [Dex Horthy](https://github.com/dexhorthy) (HumanLayer, YC24). I did not take all 12 factors because many of them are about building agent frameworks, which is not what most of us do day-to-day. We use agents, we do not build runtimes for them. Claude Code and Copilot control the runtime; we can partially control the tools and fully control the prompts.

### The Agent Loop

At its core, every agent is just this:

```
┌─────────────────────────────────────────────┐
│                                             │
│  ┌─────────┐    ┌──────────┐    ┌────────┐ │
│  │         │    │          │    │        │ │
│  │ Context ├───>│   LLM    ├───>│  Tool  │ │
│  │ window  │    │ (decide  │    │  call  │ │
│  │         │<───┤  next    │    │(execute│ │
│  │         │    │  action) │    │ action)│ │
│  └─────────┘    └──────────┘    └───┬────┘ │
│       ^                             │      │
│       │         result              │      │
│       └─────────────────────────────┘      │
│                                             │
│              Repeat until "done"            │
└─────────────────────────────────────────────┘
```

**The problem:** after many iterations, the context window fills up. The agent starts looping on the same broken approach. It forgets what it tried. Even as models support longer context, focused prompts always work better.

### Four Components You Control

| Component | What it is | What you control |
|-----------|-----------|-----------------|
| **Prompt** | Instructions for the LLM | Fully. You write it. |
| **Context** | Accumulated history of steps and results | Partially. You shape what goes in. |
| **Tools** | Actions the agent can take (read files, run commands, etc.) | Partially. You pick which tools are available. |
| **Loop** | Keep going until done | Partially. You define when to pause/stop. |

### Five Factors That Matter for Prompt Engineering

From the original 12 factors, these five are most relevant when you write prompts for coding assistants:

#### Factor 1: Natural Language Becomes Tool Calls

Your words become actions. The prompt defines what tools are available and how to use them.

```
What you type:              What the agent actually does:
─────────────               ─────────────────────────────
"/commit"                   → git status
                            → git diff
                            → git add <files>
                            → git commit -m "..."

"find auth code"            → Grep: "auth"
                            → Glob: **/auth/**
                            → LS: src/services/auth/

"explain the login flow"    → Read: src/auth/login.ts
                            → Read: src/auth/middleware.ts
                            → Trace calls between files
```

#### Factor 2: Own Your Prompts

Do not copy-paste prompts without reading them. A prompt for React will not work for Spring Boot. If you cannot explain why each instruction is there, you do not own it.

> "Our library gives you the best output!" ... "SHOW ME THE PROMPT."

#### Factor 3: Own Your Context Window

LLMs are stateless functions. Every call, you send the full context. The quality of the output depends on how you structure the input. Andrej Karpathy coined the term "context engineering" for this.

```
Context window (200k tokens):
┌──────────────────────────────────────────────────────┐
│ [system prompt] [documents] [conversation] [tools]   │
│                                                      │
│  40% used ✓ Good     │░░░░░░░░░░░░░░░░░░░░░░░░░░░░│ │
│  60% used ~ OK       │████████████░░░░░░░░░░░░░░░░│ │
│  80% used ✗ Danger   │████████████████████░░░░░░░░│ │
│  95% used ✗ Lost     │████████████████████████████│ │
│                                                      │
│  More noise = worse output                           │
│  Focused context = better output                     │
└──────────────────────────────────────────────────────┘
```

#### Factor 7: Contact Humans with Tool Calls

Build checkpoints into prompts so the agent knows when to stop and ask.

```
# From implement_plan.md
# (https://github.com/humanlayer/humanlayer/blob/main/.claude/commands/implement_plan.md):

"Phase [N] Complete - Ready for Verification.
 Automated checks passed:
 - [x] Tests pass
 - [x] Lint clean

 Please perform manual verification:
 - [ ] Feature works in UI
 - [ ] No regressions

 Let me know when complete so I can proceed to Phase [N+1]."
```

#### Factor 10: Small, Focused Agents

Instead of one big agent, create small agents that each do one specific thing.

```
BAD: One Universal Agent                GOOD: Focused Micro Agents
┌─────────────────────────┐             ┌──────────────────┐
│ Universal Researcher    │             │ codebase-locator │
│                         │             │ Tools: Grep,     │
│ Tools: ALL OF THEM      │             │   Glob, LS       │
│                         │             │ Job: find files   │
│ - Find files            │             └──────────────────┘
│ - Analyze code          │             ┌──────────────────┐
│ - Query database        │             │ codebase-analyzer│
│ - Understand patterns   │             │ Tools: Read,     │
│ - Synthesize findings   │             │   Grep, Glob, LS │
│                         │             │ Job: explain code │
│ 50+ steps               │             └──────────────────┘
│ Huge context             │             ┌──────────────────┐
│ Gets lost               │             │ web-researcher   │
│                         │             │ Tools: WebSearch, │
│                         │             │   WebFetch, Read  │
│                         │             │ Job: find docs    │
└─────────────────────────┘             └──────────────────┘

                                        Each: 5-10 steps, stays focused
```

### Practical Tips

These patterns come from real prompt engineering experience. They are not in the 12 Factors.

#### Tip 1: Negative Instructions

Tell the agent what NOT to do. This prevents drift.

```
# Bad: only positive instructions
"Analyze the codebase and describe what you find."

# Good: positive + negative instructions
"Analyze the codebase and describe what you find.
 DO NOT suggest improvements.
 DO NOT perform root cause analysis.
 DO NOT critique the implementation.
 ONLY describe what exists, how it works, and how components interact."
```

Without negative instructions, the agent starts "helping": suggesting improvements, critiquing code, going off on tangents. With them, it stays focused.

#### Tip 2: Output Templates

Define exact format for consistent, parseable results.

```
# In codebase-analyzer.md
# (https://github.com/humanlayer/humanlayer/blob/main/.claude/agents/codebase-analyzer.md):

## Analysis: [Component Name]

### Overview
[2-3 sentence summary]

### Entry Points
- `file.ts:45` - description of what's there

### Core Implementation
#### 1. [Step name] (`file.ts:15-32`)
- What it does
- How it connects to the next step

### Data Flow
1. Request arrives at `api/routes.ts:45`
2. Routed to `handlers/webhook.ts:12`
3. Validated at `handlers/webhook.ts:15-32`
```

Without a template, every response looks different. With a template, results are predictable and can be parsed by other agents.

#### Tip 3: Tool Selection Controls Capability

Limit tools to limit what the agent CAN do. This is a physical constraint, not just instructions.

| Agent | Tools | What it CAN do | What it CANNOT do |
|-------|-------|----------------|-------------------|
| [`codebase-locator`](https://github.com/humanlayer/humanlayer/blob/main/.claude/agents/codebase-locator.md) | Grep, Glob, LS | Find files | Read file contents |
| [`codebase-analyzer`](https://github.com/humanlayer/humanlayer/blob/main/.claude/agents/codebase-analyzer.md) | Read, Grep, Glob, LS | Read and analyze | Run commands, edit files |
| [`web-researcher`](https://github.com/humanlayer/humanlayer/blob/main/.claude/agents/web-search-researcher.md) | WebSearch, WebFetch, Read | Search the web | Modify local files |

If the agent does not have the `Edit` tool, it physically cannot edit files. Not just "please don't", but "literally impossible."

#### Tip 4: Read Before Spawn

The orchestrator must understand context before delegating to sub-agents.

```
WRONG:                              RIGHT:

User asks question                  User asks question
  │                                   │
  ├──> Spawn agent 1                  ├──> READ mentioned files first
  ├──> Spawn agent 2                  │     (understand the full context)
  └──> Spawn agent 3                  │
                                      ├──> Plan sub-tasks based on
  Agents get vague tasks              │     what you actually read
  Results are unfocused               │
                                      ├──> Spawn agent 1 (specific task)
                                      ├──> Spawn agent 2 (specific task)
                                      └──> Spawn agent 3 (specific task)

                                      Agents get precise tasks
                                      Results are focused
```

#### Tip 5: No Open Questions

Stop and ask instead of guessing. Five seconds to clarify saves hours of rework.

```
# From create_plan.md
# (https://github.com/humanlayer/humanlayer/blob/main/.claude/commands/create_plan.md):
"If you encounter open questions during planning, STOP.
 Research or ask for clarification immediately.
 Do NOT write the plan with unresolved questions."

# From implement_plan.md
# (https://github.com/humanlayer/humanlayer/blob/main/.claude/commands/implement_plan.md):
"When things don't match the plan:
 Issue in Phase [N]:
   Expected: [what the plan says]
   Found: [actual situation]
   Why this matters: [explanation]

 How should I proceed?"
```

## Context Management Is Everything

Dex Horthy wrote a detailed article called [Advanced Context Engineering for Coding Agents](https://github.com/humanlayer/12-factor-agents) about why context management is the most important skill for working with AI coding tools. The key insight: LLMs are stateless functions. The context window is the ONLY lever you have to affect the quality of the output.

### What Eats Up Your Context

```
Context window filling up:

 [system prompt ████]
 [user message ██]
 [grep results ████████████████████]          <-- searching for files
 [file contents ████████████████████████]     <-- reading code
 [more grep ████████████]                     <-- more searching
 [edit attempts ████████████████]             <-- trial and error
 [test output ████████████████████████████]   <-- build logs
 [error logs ████████████████]                <-- debugging
 [more edits ████████████████████]            <-- fixes

 ════════════════════════════════════════════
 Context: 87% full. Agent is lost.
 It forgot the original goal 40 messages ago.
```

### Frequent Intentional Compaction

Design your entire workflow around context management. Keep utilization in the 40-60% range. Split work into three phases:

```
Phase 1: RESEARCH                Phase 2: PLAN                Phase 3: IMPLEMENT
(fresh context)                  (fresh context)              (fresh context)

┌─────────────────┐              ┌─────────────────┐         ┌─────────────────┐
│ Input:           │              │ Input:           │         │ Input:           │
│ - ticket/issue   │              │ - research.md    │         │ - plan.md        │
│ - codebase       │              │ - ticket/issue   │         │ - codebase       │
│                  │              │                  │         │                  │
│ Agent searches,  │              │ Agent creates    │         │ Agent follows    │
│ reads, maps the  │              │ step-by-step     │         │ plan phase by    │
│ codebase         │              │ implementation   │         │ phase            │
│                  │              │ plan             │         │                  │
│ Output:          │              │ Output:          │         │ Output:          │
│ research.md      │              │ plan.md          │         │ working code     │
└────────┬────────┘              └────────┬────────┘         └────────┬────────┘
         │                                │                           │
         v                                v                           v
   ┌───────────┐                    ┌───────────┐               ┌───────────┐
   │  HUMAN    │                    │  HUMAN    │               │  HUMAN    │
   │  REVIEW   │                    │  REVIEW   │               │  REVIEW   │
   │           │                    │           │               │           │
   │ Is the    │                    │ Is the    │               │ Does the  │
   │ research  │                    │ plan      │               │ code      │
   │ correct?  │                    │ sound?    │               │ work?     │
   └───────────┘                    └───────────┘               └───────────┘

   Highest leverage!                High leverage!               Standard review
```

Each phase starts with a **fresh context window**. The output of one phase becomes a compact input for the next. This is the core idea: instead of one long messy conversation, you have three focused sessions.

### Sub-Agents for Context Control

Sub-agents are not about role-playing. They are about using a fresh context window for searching and summarizing, so the main agent stays clean.

```
Main Agent (orchestrator)
Context: 35% used
┌──────────────────────────────────────────────┐
│ system prompt + user question + sub-agent    │
│ results (compact summaries)                  │
│░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│
└──────────────────────────────────────────────┘

Sub-agent 1 (locator)     Sub-agent 2 (analyzer)     Sub-agent 3 (researcher)
Uses own context           Uses own context            Uses own context
┌────────────────┐         ┌────────────────┐         ┌────────────────┐
│ 15 grep calls  │         │ reads 8 files  │         │ 5 web searches │
│ 10 glob calls  │         │ traces 3 flows │         │ 3 page fetches │
│ 80% used       │         │ 70% used       │         │ 60% used       │
└───────┬────────┘         └───────┬────────┘         └───────┬────────┘
        │                          │                          │
        v                          v                          v
  Returns: 15 lines          Returns: 40 lines          Returns: 20 lines
  (file locations)           (code analysis)            (documentation)

All that noise stays in sub-agent context.
Main agent only sees the compact summaries.
```

### The `.claude` Prompts

The prompts I reference throughout this article are taken from the [humanlayer/humanlayer/.claude](https://github.com/humanlayer/humanlayer/tree/main/.claude) repository. Originally written by the [CodeLayer](https://humanlayer.dev) team (Dex Horthy) for use with Claude Code inside the CodeLayer IDE. You can look at the originals to understand the full picture. They are a good example of "prompts as code" that you can version control, test, and share.

| | File | What it does |
|---|------|-------------|
| **Agents** | [codebase-analyzer.md](https://github.com/humanlayer/humanlayer/blob/main/.claude/agents/codebase-analyzer.md) | Reads and explains code |
| | [codebase-locator.md](https://github.com/humanlayer/humanlayer/blob/main/.claude/agents/codebase-locator.md) | Finds files (no Read tool!) |
| | [codebase-pattern-finder.md](https://github.com/humanlayer/humanlayer/blob/main/.claude/agents/codebase-pattern-finder.md) | Finds code patterns |
| | [web-search-researcher.md](https://github.com/humanlayer/humanlayer/blob/main/.claude/agents/web-search-researcher.md) | Searches the web |
| **Commands** | [commit.md](https://github.com/humanlayer/humanlayer/blob/main/.claude/commands/commit.md) | Simple: analyze changes, commit |
| | [create_plan.md](https://github.com/humanlayer/humanlayer/blob/main/.claude/commands/create_plan.md) | Workflow: research, plan, iterate |
| | [describe_pr.md](https://github.com/humanlayer/humanlayer/blob/main/.claude/commands/describe_pr.md) | Simple: generate PR description |
| | [implement_plan.md](https://github.com/humanlayer/humanlayer/blob/main/.claude/commands/implement_plan.md) | Workflow: execute plan phase by phase |
| | [iterate_plan.md](https://github.com/humanlayer/humanlayer/blob/main/.claude/commands/iterate_plan.md) | Workflow: update existing plans |
| | [research_codebase.md](https://github.com/humanlayer/humanlayer/blob/main/.claude/commands/research_codebase.md) | Orchestrator: spawn agents, synthesize |

### Three Types of Prompts

Not all prompts are the same. Here is how they differ (see [commit.md](https://github.com/humanlayer/humanlayer/blob/main/.claude/commands/commit.md), [implement_plan.md](https://github.com/humanlayer/humanlayer/blob/main/.claude/commands/implement_plan.md), [research_codebase.md](https://github.com/humanlayer/humanlayer/blob/main/.claude/commands/research_codebase.md)):

```
SIMPLE PROMPT (commit.md)
─────────────────────────
User: /commit
  │
  ├─ git status + diff
  ├─ analyze changes
  ├─ present plan ─────── Human: "looks good" ──── execute commits
  │
  One task. Linear. One human checkpoint.


WORKFLOW PROMPT (implement_plan.md)
───────────────────────────────────
User: /implement plan.md
  │
  ├─ read plan
  ├─ execute Phase 1
  ├─ update checkboxes
  ├─ ─── Human verifies Phase 1 ───
  ├─ execute Phase 2
  ├─ update checkboxes
  ├─ ─── Human verifies Phase 2 ───
  └─ ... until all phases done

  Sequential. Multiple human gates. Persistent state (plan file).


ORCHESTRATOR PROMPT (research_codebase.md)
──────────────────────────────────────────
User: /research "how does auth work?"
  │
  ├─ READ mentioned files first
  │
  ├─ Spawn sub-agents in parallel:
  │   ├─ codebase-locator ──── finds files
  │   ├─ codebase-analyzer ─── explains code
  │   └─ web-researcher ────── finds docs
  │
  ├─ WAIT for all sub-agents
  │
  └─ Synthesize into research document

  Delegates work. Parallel execution. Synthesis focus.
```

| Type | Who does the work | Sub-agents | Human interaction |
|------|------------------|------------|-------------------|
| Simple | Agent directly | None | Confirm then execute |
| Workflow | Agent, phase by phase | Optional | Gates between phases |
| Orchestrator | Sub-agents | Core mechanism | Minimal (review synthesis) |

**Rule of thumb:** Start simple. Add workflow when you need human checkpoints between phases. Add orchestrator when you need parallel research.

## Writing Good Prompts for Existing Agents

You cannot just tell the agent "use ticket NUMBER-123 and research." That is too vague. The agent will not know what to look for, what is important, or when to stop.

### Bad vs. Good Prompts

```
BAD                                         GOOD
───                                         ────

"Research ticket ENG-1234"                  "Research the payment processing flow.
                                             Focus on Stripe webhook handling.
                                             I need to understand how payment
                                             status gets updated in the database.
                                             Relevant code: src/services/payments/
                                             and src/api/webhooks/."

"Fix the bug"                               "/create_plan eng_1234.md

                                             Think about the migration strategy.
                                             We cannot have downtime.
                                             Look at how we handled PR #456."

"Implement the feature"                     "/implement plan.md

                                             Start with Phase 1 only.
                                             Run tests after each change.
                                             If something does not match the plan,
                                             stop and tell me."
```

### The Pattern

Every good prompt to an existing agent follows this structure:

```
┌──────────────────────────────────────────┐
│  1. SCOPE: What exactly to work on       │
│     "Research the payment processing     │
│      flow in our codebase"               │
│                                          │
│  2. FOCUS: Where to look                 │
│     "Relevant code is probably in        │
│      src/services/payments/"             │
│                                          │
│  3. CONTEXT: What matters and why        │
│     "We need to understand this because  │
│      we are migrating to Stripe v3"      │
│                                          │
│  4. BOUNDARIES: When to stop or ask      │
│     "If you find more than 3 services    │
│      involved, stop and tell me before   │
│      going deeper"                       │
└──────────────────────────────────────────┘
```

The [prompts in `.claude/commands/`](https://github.com/humanlayer/humanlayer/tree/main/.claude/commands) already have good structure built in (negative instructions, output templates, step-by-step strategies, human checkpoints). Your job is to give them specific context to work with, not vague directions.

### Anatomy of a Well-Written Agent Prompt

Here is what makes the [prompts in `.claude/agents/`](https://github.com/humanlayer/humanlayer/tree/main/.claude/agents) effective. Using [`codebase-analyzer.md`](https://github.com/humanlayer/humanlayer/blob/main/.claude/agents/codebase-analyzer.md) as an example:

```yaml
---
name: codebase-analyzer
tools: Read, Grep, Glob, LS            # Limited tools = limited scope
model: sonnet                           # Cheaper model for focused tasks
---
```

```markdown
# Role (one sentence)
"You are a specialist at understanding HOW code works."

# Negative instructions (prevent drift)
"DO NOT suggest improvements"
"DO NOT critique the implementation"
"ONLY describe what exists"

# Step-by-step strategy (how to do the job)
Step 1: Read Entry Points
Step 2: Follow the Code Path
Step 3: Document Key Logic

# Output template (consistent format)
## Analysis: [Name]
### Overview
### Entry Points
  - `file:line` - description
### Core Implementation
### Data Flow

# Closing reminder
"REMEMBER: You are a documentarian, not a critic."
```

This structure works because each part prevents a specific failure mode:
- **Limited tools** prevent the agent from doing things outside its scope
- **Negative instructions** prevent it from drifting into "helpful" suggestions
- **Step-by-step strategy** prevents random, inconsistent analysis
- **Output template** prevents unparseable responses
- **Closing reminder** reinforces the constraints (LLMs pay attention to the end of prompts)

## What AI Makes Possible

I do not want to sound like a hype article, but there are things that are genuinely hard to do without AI tools:

- **Navigating unfamiliar codebases.** Dex Horthy and a colleague shipped a PR to a 300k LOC Rust codebase they had never seen before. The research phase mapped the codebase, the plan targeted the right fix, and the implementation landed. Would it work without AI? Sure. Would it take days instead of hours? Probably.
- **Parallel research.** You can spawn multiple focused agents to investigate different parts of the codebase at the same time. One finds files, another analyzes code, another checks the database schema. The orchestrator synthesizes everything.
- **Consistent code generation.** Once you have a good plan, the implementation phase is straightforward. The agent follows the spec, and the code style matches your existing codebase because the agent read it first.
- **Onboarding.** New team members can use research prompts to get up to speed on unfamiliar parts of the codebase. An intern at HumanLayer shipped 2 PRs on his first day and 10 on his 8th day.
- **Maintaining mental alignment.** Instead of reading 2000 lines of code in a PR, you read 200 lines of a well-written implementation plan. You know what is being built and why.

These are real benefits. They do not make you 10x faster at everything. But they make some previously painful tasks genuinely easier.

## Credits and References

This work is heavily based on and inspired by other people's work. I want to give proper credit.

**[12-Factor Agents](https://github.com/humanlayer/12-factor-agents)** by [Dex Horthy](https://github.com/dexhorthy) ([HumanLayer](https://humanlayer.dev), YC24). The foundation for understanding how to build reliable AI agents. My article adapts 5 of the 12 factors for prompt engineering use. The original content is licensed under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/).

**[Advanced Context Engineering for Coding Agents](https://github.com/humanlayer/12-factor-agents)** by Dex Horthy. The article about frequent intentional compaction and the research/plan/implement workflow.

**[We Mourn Our Craft](https://nolanlawson.com/2026/02/07/we-mourn-our-craft/)** by Nolan Lawson. An honest and emotional piece about accepting the AI shift in software development.

**[Vibe Coding Our Way to Disaster](https://www.arthropod.software/p/vibe-coding-our-way-to-disaster)** by Jake Nations. About the risks of unstructured AI coding, based on Rich Hickey's ideas about simplicity vs. ease.

**[Context Engineering](https://x.com/karpathy/status/1937902205765607508)** - term coined by Andrej Karpathy for the art of providing all the context needed for a task to be plausibly solvable by an LLM.

**The `.claude` prompts** in this repo are taken from [humanlayer/humanlayer/.claude](https://github.com/humanlayer/humanlayer/tree/main/.claude), originally created by the [CodeLayer](https://humanlayer.dev) team (Dex Horthy) for use with Claude Code inside the CodeLayer IDE.

**[Specs Are the New Code](https://www.youtube.com/watch?v=8rABwKRsec4)** by Sean Grove. The idea that specifications will become the real source code.

**[Stanford Study on AI's Impact on Developer Productivity](https://www.youtube.com/watch?v=tbDDYKRFjhk)** - research showing that AI tools sometimes reduce productivity in established codebases.

## About Me

I'm a software engineer who is currently incorporating AI-assisted development into the company I work at. I am still in the process of understanding agents and LLMs. This repository is my notes and thoughts, not a definitive guide.

If you find mistakes or disagree with something, that is expected. I am learning, and this will evolve over time.

## License

Content in this repository is licensed under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/), consistent with the [12-Factor Agents](https://github.com/humanlayer/12-factor-agents) content license.

Code examples are licensed under [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0).
