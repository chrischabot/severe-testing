---
name: severe-testing
description: Severe, serious, adversarial, adveserial, malicious, and security-focused testing for code, systems, UI flows, APIs, data pipelines, agents, and infrastructure. Use when the user asks for severe testing, serious testing, adversarial testing, adveserial testing, malicious testing, malicious breakage testing, red-team testing, security vulnerability exposing tests, security vunerability exposing tests, abuse-case tests, fuzzing, property-based tests, stress tests, failure-mode tests, root-cause validation, or stronger verification after an implementation or bug fix.
---

# Severe Testing

Use this skill to replace "it seems to work" with falsifiable evidence. Severe testing is a fair but aggressive attempt to refute an implementation's claims, expose root causes, and find failures before users or attackers do.

## Core Stance

- State assumptions, preconditions, and success criteria before designing tests.
- Test behavioral claims, not defaults, constructors, type annotations, or incidental internals.
- Prefer tests that would fail for plausible broken implementations.
- Reproduce known bugs before fixing them, then keep the reproducer as a regression test.
- Investigate failing tests to root cause before adding fixes or relaxing assertions.
- Use the project's real validation surface: unit, integration, browser, build, lint, typecheck, performance, security, and runtime checks as appropriate.
- Run destructive, malicious, or failure-injection scenarios only in disposable fixtures, sandboxes, temporary directories, test tenants, or authorized staging environments.
- Never attack third-party systems, production systems, or real user data. Model hostile behavior safely inside authorized test environments.

## Workflow

1. **Name the claim.** Write what the code claims to do, the valid input space, postconditions, invariants, performance budgets, security boundaries, and failure semantics.
2. **List refutations.** For each claim, name observations that would prove it false: wrong value, silent corruption, leaked data, unauthorized access, stale state, race, timeout, resource blowup, missing audit trail, or unsafe fallback.
3. **Choose oracles.** Prefer independent references, specs, mathematical identities, normalized diffs, state-machine invariants, security policies, permission matrices, schema validators, accessibility trees, or trusted library behavior.
4. **Attack the input space.** Cover boundaries, just-outside-boundaries, malformed data, Unicode, nulls, huge values, ordering changes, duplicate names, collisions, stale caches, retries, interrupted writes, concurrent operations, and random operation sequences.
5. **Add malicious breakage tests.** Simulate attacker-controlled input, hostile ordering, compromised dependencies, confused-deputy paths, privilege escalation attempts, forged identities, tampered storage, replayed requests, and resource exhaustion.
6. **Run the strongest relevant checks.** Add targeted tests when existing checks do not refute the claim. Prefer automated evidence over inspection alone.
7. **Report evidence.** Summarize what was tested, what failed, root cause, fix, remaining risk, and exact commands or artifacts that verify the outcome.

## Severity Ladder

- **Weak:** no-crash checks, shape/type assertions, or one obvious happy path.
- **Moderate:** representative cases, edge cases, invalid input handling, and clear assertions.
- **Strong:** property-based tests, fuzzing, reference oracles, invariant checks, concurrency tests, and integration validation.
- **Severe:** adversarially selected inputs plus independent oracles, malicious/abuse cases, failure injection, resource limits, persistence/restart checks, authorization boundaries, and regression coverage for discovered root causes.

Important code should have at least one strong or severe test. Critical code should combine several independent oracles.

## Malicious And Security Coverage

Include the relevant categories whenever the code handles identity, data, files, network calls, plugins, user content, external input, generated code, credentials, or cross-service operations.

- **Authorization:** prove users, tenants, roles, scopes, object ownership, and row-level rules cannot be bypassed by direct IDs, stale sessions, replay, cross-account references, hidden UI calls, or forged client state.
- **Authentication/session:** test expired tokens, revoked tokens, missing CSRF protection, refresh races, session fixation, logout invalidation, clock skew, and multi-device state.
- **Injection:** test SQL/NoSQL/LDAP/template/command/path/header/HTML/Markdown/JS/CSS injection using payloads that prove output is encoded, parameterized, or rejected.
- **File and path handling:** test traversal, symlink escapes, absolute paths, alternate separators, case collisions, reserved names, zip slip, oversized files, MIME confusion, and partial-write recovery.
- **SSRF and network egress:** test internal IPs, metadata endpoints, redirects, DNS rebinding patterns, non-HTTP schemes, allowlist bypasses, timeout behavior, and response-size limits.
- **Deserialization and parsing:** test malformed JSON/YAML/XML/protobuf, entity expansion, recursive/deep nesting, duplicate keys, prototype pollution, schema drift, and legacy compatibility traps.
- **Secrets and privacy:** prove logs, telemetry, errors, snapshots, caches, URLs, browser storage, test artifacts, and model prompts do not leak tokens, PII, private files, or tenant data.
- **Concurrency and consistency:** test simultaneous writes, stale reads, retry storms, idempotency keys, double-submit, lock ordering, transaction rollback, cache invalidation, and crash/restart recovery.
- **Resource exhaustion:** test payload size, fanout, recursion depth, regex backtracking, decompression bombs, memory growth, CPU loops, unbounded queues, rate limits, and cancellation.
- **Supply chain and plugins:** test untrusted packages/plugins/extensions, manifest spoofing, update downgrade, signature or checksum failure, dependency confusion, lifecycle cleanup, and sandbox escape.
- **UI abuse:** test hidden or disabled controls, keyboard-only paths, drag/drop spoofing, clipboard payloads, hostile filenames, focus traps, clickjacking-sensitive flows, and accessibility regressions.
- **AI/agent behavior:** test prompt injection, tool misuse, data exfiltration through tool outputs, unsafe code execution, policy bypass by indirect instructions, and least-privilege tool access.

## Adversarial Input Seeds

Use domain-specific generators where possible. Seed hand-written corpora with:

- Empty, single, duplicate, sorted, reverse-sorted, huge, deeply nested, and highly repetitive structures.
- `0`, `-0`, min/max integers, `NaN`, infinities, subnormal floats, epsilon-scale differences, and ill-conditioned matrices.
- Unicode normalization variants, combining marks, RTL override, emoji, null bytes, control characters, very long strings, and mixed newline styles.
- `../`, absolute paths, Windows drive paths, UNC paths, symlinks, reserved filenames, invisible characters, and case-only collisions.
- `"'`; SQL comments, shell metacharacters, HTML/script tags, template delimiters, markdown links/images, CSS escapes, and header-breaking CRLF.
- Duplicate IDs, stale versions, reordered events, dropped events, repeated retries, clock jumps, interrupted saves, and process restarts.

## Test File Shape

Document important claims near the tests:

```text
CLAIM: The operation preserves X and rejects Y.
PRECONDITIONS: Inputs, identities, permissions, state, and environment assumed valid.
POSTCONDITIONS: Observable guarantees after success or failure.
ORACLE: Independent reference, invariant checker, policy matrix, normalized diff, or trusted spec.
SEVERITY: Moderate | Strong | Severe, with rationale.
```

## Evidence Standard

A severe-testing pass is incomplete until it has:

- A failing or plausibly failing condition that would catch a real class of bug.
- Assertions against observable behavior or an independent oracle.
- Coverage for malicious, invalid, boundary, and recovery paths where relevant.
- Commands run and their results.
- A precise note of untested risk, skipped categories, or weakened oracles.

Do not call work done just because tests pass if the tests do not actually threaten the claim.
