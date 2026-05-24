# Severe Testing

A reusable Codex and Claude Code skill for designing tests that try to refute a system's claims instead of merely confirming the happy path.

## Why This Exists

Many test suites prove that software works for the examples the author expected. Severe testing asks a harder question: what observation would prove this claim false, and have we made a serious attempt to find it?

The skill guides an agent toward tests with real evidential weight: adversarial inputs, independent oracles, malicious abuse cases, failure injection, concurrency pressure, recovery checks, and security boundary validation.

## When It Should Trigger

The skill metadata is written to trigger for prompts such as:

- severe testing
- serious testing
- adversarial testing
- malicious testing
- malicious breakage testing
- red-team testing
- abuse-case testing
- security vulnerability exposing tests
- fuzzing, property-based testing, stress testing, or failure-mode testing

It also includes common misspellings such as `adveserial` and `vunerability` so the skill still activates when a prompt is typed quickly.

## What It Teaches The Agent To Do

- State assumptions, claims, preconditions, postconditions, and success criteria before writing tests.
- Build tests around refutation rather than confirmation.
- Prefer strong oracles such as reference implementations, invariants, normalized diffs, permission matrices, and trusted specifications.
- Add adversarial, malicious, security, fuzz, concurrency, recovery, and resource-exhaustion coverage where relevant.
- Fan out broad surfaces by independent threat-category lenses (authz, injection, concurrency, resource exhaustion, supply chain, secrets/privacy) so one lens's blind spot does not silence another.
- Score each candidate finding on a 0-100 confidence scale weighted toward demonstrated evidence over speculation, and filter out the speculative tail before reporting.
- Suppress the common false positives of over-eager security testing (no real attack surface, implausible preconditions, mitigations at a higher layer, unreachable internal helpers, generic warnings without a concrete path).
- Reproduce failures before fixing them, then keep regression tests.
- Validate with the strongest relevant project checks and report the evidence plainly.

## Installation

Clone the repository:

```bash
git clone https://github.com/chrischabot/severe-testing.git
```

Install for Codex by copying or linking the cloned directory into the Codex skills directory:

```bash
mkdir -p "$HOME/.codex/skills"
ln -s "$(pwd)/severe-testing" "$HOME/.codex/skills/severe-testing"
```

Install for Claude Code by copying or linking the cloned directory into the Claude skills directory:

```bash
mkdir -p "$HOME/.claude/skills"
ln -s "$(pwd)/severe-testing" "$HOME/.claude/skills/severe-testing"
```

If symlinks are inconvenient on your platform, copy the directory instead. The installed skill directory must contain `SKILL.md` at its root.

## Files

- `SKILL.md`: The skill instructions and trigger metadata.
- `agents/openai.yaml`: Optional UI metadata for Codex skill lists.

## Safety Boundary

This skill is for authorized testing. It tells agents to model hostile behavior only inside disposable fixtures, sandboxes, test tenants, local harnesses, or approved staging environments. It should not be used to attack third-party systems, production systems, or real user data.

## Design Principle

Passing tests are not proof. Confidence comes from failing to refute a precise claim despite tests that could realistically have exposed a bug.
