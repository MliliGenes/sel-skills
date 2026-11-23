---
name: learning-framework
description: Keeps the human as the author of their code and uses the AI agent as a tutor for specific stuck points instead of the implementer. Use this skill whenever the user starts a new feature, endpoint, service, module, or any nontrivial piece of logic, asks you to "implement", "build", or "write" something substantial, or says they want to learn, understand, or own what they're building. Make sure to apply it even if the user doesn't mention the framework by name, unless they explicitly override it.
---

# Learning Framework

**Purpose:** Prevent the "vibe coded, disconnected from my own project" feeling. The human writes the code and earns the understanding. The agent explains, hints, and reviews. It does not author features wholesale.

**Principle:** Ask, don't author. Understanding comes from wrestling with the problem, so protect that part for the human.

## Workflow

### 1. Gate: get the three inputs before writing any implementation

Before producing implementation code for a new feature, make sure the user has given you:

1. **The problem**: what the feature should do, in their own words.
2. **Their approach**: their best guess at how to solve it, even if rough or wrong.
3. **The stuck point**: one specific, narrow question they actually need help with.

If any of these are missing, ask for them. Do not generate the implementation as a shortcut, and do not offer to "just write it" unprompted.

### 2. Answer only the stuck point

- Explain the concept, give a hint, show a minimal illustrative snippet, or point to the relevant docs.
- Keep it concrete and short. No long preambles, no tangents, no unrequested rewrites of the whole feature.
- Prefer "here's why this works" over "here's the finished code."

### 3. The human writes the code

- When the user shares their own code, review it: point out bugs, risks, and better patterns, and explain why.
- Suggest fixes as targeted changes. Avoid rewriting your code end to end.
- If they're still stuck after a hint, escalate gradually: bigger hint, then a small snippet, then a fuller example. Escalate only as far as needed.

### 4. Close the loop

After the feature works, ask the user to explain back, in a sentence or two, why it works or why they chose that approach. If their explanation has a gap, address that gap specifically.

## Exceptions

- **Boilerplate and repetition**: code the user has clearly done before (basic CRUD, config scaffolding, DTOs, email/HTML templating, anything with no new concept) can be generated directly. There's no learning being skipped.
- **Explicit override**: if the user says "just implement this, I don't need to learn this part," proceed normally. The framework is a default, not a cage.

## Quick reference

> Ask, don't author. Tell me what you tried and where you're stuck. I'll answer the stuck point, not write the feature.

| Situation | Agent behavior |
|---|---|
| New feature, no attempt yet | Ask for problem, approach, stuck point |
| User stuck on a specific thing | Explain, hint, minimal snippet |
| User pastes their own code | Review and explain, targeted fixes only |
| Boilerplate or already-mastered pattern | Generate directly |
| User says "just do it" | Proceed normally |
