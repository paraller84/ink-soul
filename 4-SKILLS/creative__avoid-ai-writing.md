---
name: avoid-ai-writing
description: Audit and rewrite content to remove AI writing patterns ("AI-isms"). Use this skill when asked to "remove AI-isms," "clean up AI writing," "edit writing for AI patterns," "audit writing for AI tells," or "make this sound less like AI." Supports a detection-only mode that flags patterns without rewriting.
version: 3.3.1
license: MIT
compatibility: Any AI coding assistant that supports agentskills.io SKILL.md format (Claude Code, Cursor, VS Code Copilot, Hermes Agent, OpenHands, etc.) or OpenClaw. No external tools or APIs required.
metadata:
  author: Conor Bronsdon
  tags: writing editing voice quality
  agentskills_spec: "1.0"
  openclaw:
    emoji: "✍️"
---

# Avoid AI Writing — Audit & Rewrite

You are editing content to remove AI writing patterns ("AI-isms") that make text sound machine-generated.

## Modes

This skill operates in one of two modes:

**`rewrite`** (default) — Flag AI-isms and rewrite the text to fix them.

**`detect`** — Flag AI-isms only. No rewriting.

## What to Remove or Fix

### Formatting
- **Em dashes (— and --)**: Replace with commas, periods, parentheses, or rewrite as two sentences. Target: zero. Hard max: one per 1,000 words.
- **Bold overuse**: One bolded phrase per major section at most.
- **Emoji in headers**: Remove entirely. Exception: social posts (1–2 emoji at end of line).
- **Excessive bullet lists**: Convert to prose paragraphs unless genuinely list-like.

### Sentence Structure
- **"It's not X — it's Y"**: Rewrite as direct positive statement. Max one per piece.
- **Hollow intensifiers**: Cut `genuine`, `truly`, `quite frankly`, `to be honest`, `let's be clear`, `it's worth noting that`.
- **Vague endorsement ("worth [verb]ing")**: Cut or replace with specific reason.
- **Hedging**: Cut `perhaps`, `could potentially`, `it's important to note that`.
- **Missing bridge sentences**: Each paragraph should connect to the last.
- **Compulsive rule of three**: Vary groupings. Max one triad per piece.

### Tier 1 — Always Replace
delve → explore, dig into | landscape (metaphor) → field, space, industry | paradigm → model, approach | robust → strong, reliable | comprehensive → thorough, complete | leverage (verb) → use | pivotal → important, key | seamless → smooth, easy | utilize → use | game-changer → describe specific change | deep dive → look at, examine | unpack → explain, break down | actionable → practical, concrete | impactful → effective, significant | learnings → lessons, findings | best practices → what works, proven methods | synergy → describe actual combined effect | in order to → to | due to the fact that → because | serves as → is | features (verb) → has, includes

### Tier 2 — Flag in Clusters (2+ in same paragraph)
harness, navigate, foster, elevate, unleash, streamline, empower, bolster, resonate, revolutionize, facilitate, underpin, nuanced, crucial, ecosystem (metaphor), myriad, plethora, encompass, catalyze, reimagine, galvanize, augment, cultivate, illuminate, transformative, cornerstone, paramount, burgeoning, nascent

### Tier 3 — Flag by Density (≥3% of words)
significant, innovative, effective, dynamic, scalable, compelling, unprecedented, exceptional, remarkable, sophisticated, world-class, state-of-the-art

### Template Phrases to Avoid
- "Whether you're X or Y" (false breadth)
- "In today's X" / "In an era where" → cut or state specific context
- "In conclusion" / "In summary" → conclusion should be obvious
- "When it comes to" → talk directly

### Structural Issues
- **Uniform paragraph length**: Vary deliberately (1-2 sentence paragraphs mixed with longer ones)
- **Formulaic openings**: Avoid "In the rapidly evolving world of..."
- **Suspiciously clean grammar**: Allow fragments, "And"/"But" starts
- **Significance inflation**: Delete "marking a pivotal moment in the evolution of..."
- **Copula avoidance**: Default to "is"/"has" instead of "serves as", "features", "boasts"
- **Synonym cycling**: If the same word is correct, repeat it. Don't thesaurus-swap.
- **Vague attributions**: "Experts believe", "Studies show" → cite specific source or drop.
- **Generic conclusions**: "The future looks bright", "Only time will tell" → cut.
- **Chatbot artifacts**: "I hope this helps!", "Certainly!", "Great question!" → remove entirely.
- **"Let's" constructions**: "Let's explore", "Let's take a look" → just start with the point.
- **Promotional language**: "nestled within breathtaking foothills" → plain description.
- **Emotional flatline**: "What surprised me most", "I was fascinated to discover" → show don't tell.

## Severity Tiers
- **P0 — Credibility killers**: Cutoff disclaimers, chatbot artifacts, vague attributions
- **P1 — Obvious AI smell**: Word-list violations, template phrases, "let's" openers, synonym cycling
- **P2 — Stylistic polish**: Generic conclusions, uniform paragraph length, copula avoidance

## Output Format (Rewrite Mode)
1. **Issues found** — bulleted list of every AI-ism with offending text quoted
2. **Rewritten version** — full rewritten content
3. **What changed** — brief summary of major edits
4. **Second-pass audit** — re-check rewrite for remaining AI tells

## Context Profiles
- **linkedin**: relaxed emoji/bold, skip excessive bullets
- **blog** (default): all rules at full strength
- **technical-blog**: technical terms get a pass; skip word table for `robust`, `comprehensive`, `ecosystem`
- **investor-email**: extra strict on promotional language
- **docs**: relaxed on most rules; clarity over voice
- **casual**: P0 only (chatbot artifacts, cutoff disclaimers)

## Tone Calibration
1. **Vary sentence length** — mix short with long. Fragments are fine.
2. **Be concrete** — replace vague claims with numbers, names, dates, or examples.
3. **Have a voice** — use first person, state preferences.
4. **Cut the neutrality** — humans have opinions.
5. **Earn your emphasis** — don't tell the reader something is interesting. Make it interesting.

## When to Rewrite from Scratch
If text has 5+ flagged vocabulary hits across multiple categories AND 3+ pattern categories triggered AND uniform sentence/paragraph length → advise full rewrite. State the core point in one sentence, then rebuild.

## Self-Reference Escape Hatch
When writing *about* AI writing patterns, quoted examples and code blocks are exempt from flagging.
