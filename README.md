# Severe Testing Skill

This project contains a Codex/Claude Code skill for severe testing: testing that tries to refute a system's claims instead of merely confirming the happy path.

## Why

Normal test suites often prove that a feature works for the examples the author had in mind. Severe testing asks a sharper question: what would make this claim false, and have we tried hard enough to find it?

The skill is designed to trigger automatically for prompts that ask for severe testing, serious testing, adversarial testing, adveserial testing, malicious testing, malicious breakage testing, red-team testing, or security vulnerability exposing tests.

## What

The skill guides an agent to:

- State assumptions, claims, preconditions, postconditions, and success criteria.
- Build tests around refutation, not confirmation.
- Use stronger oracles such as reference implementations, invariants, canonical diffs, permission matrices, and trusted specs.
- Add adversarial, malicious, security, fuzz, concurrency, recovery, and resource-exhaustion coverage.
- Reproduce failures before fixing them, then keep regression tests.
- Validate with the strongest relevant local commands and report the evidence.

## Files

- `SKILL.md`: The skill loaded by Codex and Claude Code.
- `agents/openai.yaml`: UI metadata for Codex skill lists.

## Installation

Install by copying or linking this directory into:

- Codex: `~/.codex/skills/severe-testing`
- Claude Code: `~/.claude/skills/severe-testing`

This repository was created as the canonical source at `~/Projects/severe-testing`; the installed copies should match this directory.
