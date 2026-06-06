# CLAUDE.md

This file provides guidance to Claude Code when working with this repository.

## Project Overview

This is the **conversational-graph-skills** repository — the architecture for building a conversational agent as a code-driven graph / state machine (focused nodes + forward/back transitions + deterministic gates), implemented in your own stack rather than a single mega-prompt or a hosted workflow platform.

**Repository**: https://github.com/milojarow/conversational-graph-skills

## Repository Structure

```
conversational-graph-skills/
├── .claude-plugin/          # Claude Code plugin configuration (marketplace.json + plugin.json)
├── CLAUDE.md                # This file
├── README.md                # Project overview
├── LICENSE                  # MIT License
├── evaluations/             # GREEN test scenarios for the skill
└── skills/
    └── conversational-graph/
        ├── SKILL.md          # Entry point: when to use + the model + the map
        └── reference/        # the-model, transitions-and-gotchas, code-vs-platform
```

## The skill

### conversational-graph
The methodology for a code-driven conversational graph engine. Covers the data model (node = prompt + tools + tool-gate + edges; edge = target + type `unconditional`/`llm`/`expression` + condition/predicate; node-centric so "back" is just an edge to an earlier node), the runtime loop (node answers → evaluate edges in priority order, `expression` inline + `llm` lazy → transition), the hard-won transition gotchas (route on user intent not the node's reply; completion by `expression` not LLM; classifier balance; security via `expression` tool-gates), and the decision of code graph vs hosted platform vs simple 2-way router.

## Skill Activation

Activates when the task is designing or debugging a multi-stage conversational agent in code — splitting a chatbot into routed stages, adding/fixing transitions, enforcing a verification gate, or diagnosing a bot that mis-routes or hallucinates a completed step.

## Updating this skill

After any session that discovers a new conversational-graph pattern or wall. Keep entries generic — patterns and examples, never client data, real IDs, hostnames, or business specifics. The git log of this repo is the diary.
