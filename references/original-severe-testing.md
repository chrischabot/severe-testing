---
name: severe-testing
description: Rigorously verify code correctness by articulating falsifiable claims and designing severe tests that could actually refute them. Use after writing algorithms, ML components, data processing, critical functionality, or any code where correctness matters. Follows Popperian epistemology - confidence comes from failing to refute, not from confirming.
---

# Severe Testing

A methodology for gaining genuine confidence in code correctness through falsifiable claims and severe tests.

## Core Principle

We gain confidence in code not by confirming it works, but by *failing to refute* it despite sincere attempts. A test that cannot fail provides no evidence. Design tests that would actually catch bugs if they existed.

**Critical mindset shift:** Tests should be *adversarial* - actively trying to break the code through fair but aggressive means. A good test suite should frequently reveal new information about the implementation: its actual performance envelope, numerical limits, edge behaviors, and failure modes. If running your tests never surprises you, they aren't severe enough.

## When to Apply

After writing any non-trivial code. Scale the rigour to match the code's importance and complexity:

| Code Type | Rigour Level | Approach |
|-----------|--------------|----------|
| Simple utilities, glue code | Light | Basic claim + 1-2 tests covering main path and one edge case |
| Standard business logic | Moderate | Full claim structure + property-based or systematic edge case tests |
| Algorithms, ML components, numerical code | High | Full claim structure + reference implementation + property tests |
| Critical/safety-relevant code | Maximum | All of the above + oracle verification + formal properties where possible |

## Anti-Patterns: Tests That Waste Time

**Do NOT write these tests** - they provide false confidence and break on irrelevant changes:

| Anti-Pattern | Why It's Worthless | What To Do Instead |
|--------------|-------------------|-------------------|
| Testing default initialization | `assert obj.field == default` tells you nothing about behavior | Test behavior under actual usage conditions |
| Testing that constructors don't crash | `obj = MyClass()` without assertions is not a test | Test the object actually works for its purpose |
| Testing type annotations hold | `assert isinstance(result, int)` is usually redundant | Test the *value* is correct, not just the type |
| Trivial round-trips | `assert parse(serialize(x)) == x` with one hardcoded `x` | Fuzz with random/adversarial inputs |
| Happy-path-only with obvious inputs | `assert add(2, 2) == 4` | Find inputs where bugs actually hide |
| Testing implementation details | `assert obj._internal_state == ...` | Test observable behavior, not internals |

**The litmus test:** If your test would pass even if the implementation were replaced with a trivial stub, it's not testing anything real.

## The Adversarial Mindset

Approach testing as a **red team exercise**. Your goal is to break the code, not confirm it works.

### Think Like an Attacker

For every function, ask:
- What inputs would cause this to fail silently (return wrong results without crashing)?
- What inputs would cause numerical instability, overflow, or precision loss?
- What happens at the boundaries of the valid input space?
- What happens just *outside* the boundaries?
- What timing, ordering, or concurrency issues could cause failures?
- What resource exhaustion scenarios are possible?

### Severity Means Difficulty

A severe test should be **hard to pass by accident**. If a buggy implementation could still pass your test, the test is too weak.

**Weak:** Test passes if output is non-null
**Moderate:** Test passes if output matches a few hardcoded examples
**Strong:** Test passes only if output matches a reference oracle across hundreds of random inputs
**Severe:** Test passes only if output matches oracle across adversarially-chosen inputs designed to expose common bug patterns

### Fair But Aggressive

"Fair" means the test respects the stated preconditions - don't pass invalid inputs and complain it crashes. "Aggressive" means you explore the full space of valid inputs, especially the corners where bugs hide.

### Quantifying Test Strength

Think of each test as having a **false positive rate (β)**: the probability it passes even when the code is wrong.

| Test Type | Typical β | Interpretation |
|-----------|-----------|----------------|
| Only checks no crash | ~0.9 | Buggy code usually doesn't crash either |
| Shape/type checks only | ~0.6 | Wrong values often have right shapes |
| Few hardcoded examples | ~0.3 | Bug might not hit those specific inputs |
| Property-based with weak oracle | ~0.2 | Good coverage, weak correctness check |
| Property-based with reference oracle | ~0.01 | Bug would have to fool both implementations |
| Formal specification check | ~0.001 | Near-certain detection of violations |

**Rule of thumb:** If β > 0.1, the test is too weak to provide real confidence.

### Evidence Quantity Matters

Even strong tests need sufficient quantity. One perfect test isn't enough - you need coverage across the input space.

| Evidence Accumulated | Maximum Reasonable Confidence |
|---------------------|------------------------------|
| 1-2 strong tests | ~60% - "probably works" |
| 3-5 strong tests | ~80% - "likely works" |
| 10+ strong tests across diverse scenarios | ~95% - "high confidence" |

Don't mistake a single passing test for proof, regardless of how severe that test is.

## Discovery-Oriented Testing

The best tests don't just validate - they **reveal information** about your implementation you didn't know.

### Tests Should Produce Artifacts

When testing numerical code, algorithms, or systems with complex behavior, tests should output:

- **Performance envelopes:** "Function maintains <1ms latency up to N=10,000 inputs, degrades linearly after"
- **Numerical bounds:** "Stable for inputs in range [1e-10, 1e10], produces NaN outside this range"
- **Error characteristics:** "Mean error 1e-8, max observed error 2.3e-6 at input x=..."
- **Resource profiles:** "Memory usage scales as O(n), peaks at 2.3GB for n=1M"

### Refining Claims Through Testing

Initial claim: "Sorts the input list"
After adversarial testing: "Sorts lists of up to 10M elements in O(n log n) time. Stable sort. Handles duplicates. Fails with RecursionError above 10M elements due to stack depth."

**The claim should get MORE PRECISE as you test**, not just get confirmed. If testing doesn't refine your understanding, you're not testing hard enough.

### Discovery Patterns

```python
def test_discover_numerical_limits():
    """Find where the implementation breaks down numerically."""
    for magnitude in range(-15, 15):
        x = 10.0 ** magnitude
        result = function_under_test(x)
        if not np.isfinite(result) or relative_error(result, reference(x)) > 1e-6:
            print(f"Breaks down at magnitude 10^{magnitude}")
            break
    # Now you know the actual working range

def test_discover_performance_cliff():
    """Find where performance degrades."""
    for n in [10, 100, 1000, 10000, 100000]:
        start = time.perf_counter()
        function_under_test(generate_input(n))
        elapsed = time.perf_counter() - start
        print(f"n={n}: {elapsed:.4f}s")
    # Now you know the actual scaling behavior
```

## The Workflow

### Step 1: Articulate the Claim

Before or immediately after writing code, state explicitly what it claims to do. Be precise and falsifiable.

**Ask yourself:**
- What does this code claim to compute/do?
- Under what conditions does this claim apply? (preconditions)
- What must be true when it completes? (postconditions)
- Are there resource constraints? (time, memory, etc.)
- Is this deterministic or statistical?

**Bad claim:** "This function processes data correctly"
**Good claim:** "Given a non-empty list of positive floats, returns their geometric mean. Output is always positive. For single-element input, returns that element unchanged."

### Step 2: Identify What Would Refute It

For each postcondition, ask: "What test result would prove this claim false?"

- Output shape mismatch → claim refuted
- Output differs from reference implementation → claim refuted
- Violates mathematical identity → claim refuted
- Crashes on valid input → claim refuted
- Exceeds resource budget → claim refuted

### Step 3: Design Tests by Severity

Not all tests are equally informative. Design from weak to strong:

**Weak tests** (low refutation power):
- Only checks that code runs without crashing
- Only checks output type/shape
- Single hardcoded happy-path input

**Moderate tests** (medium refutation power):
- Handcrafted edge cases (empty input, single element, boundary values)
- Range/sanity checks on output
- Multiple representative inputs

**Strong tests** (high refutation power):
- Property-based/fuzz testing with random valid inputs
- Comparison against reference implementation
- Mathematical identity verification
- Formal specification checking

**Always include at least one strong test for important code.**

### Step 4: Select or Create an Oracle

An oracle determines whether output is correct. Options ranked by strength:

1. **Formal specification** - Mathematical definition that can be checked mechanically
2. **Reference implementation** - Known-correct implementation (trusted library, textbook algorithm)
3. **Mathematical identities** - Properties that must hold (e.g., inverse operations, commutativity)
4. **Consistency checks** - Output must be consistent with itself or related computations
5. **Range/type checks** - Output must be in valid range, correct type
6. **No crash** - Code completes without exception (very weak)

For numerical/algorithmic code, prefer reference implementations or mathematical identities.

### Step 5: Verify the Oracle (When Warranted)

For critical code with non-trivial oracles, ask: "How do I know my test is correct?"

**When to verify the oracle:**
- The oracle contains non-trivial logic (reference implementations, custom generators)
- The oracle is load-bearing (primary evidence for an important claim)
- The oracle has been wrong before

**The verification hierarchy:** Your tests verify your code, but what verifies your tests? This chain must terminate somewhere:

1. **Mathematical definitions** - Formal specs checkable mechanically (Lean, Coq, z3)
2. **Hand-computed analytic cases** - Simple inputs where humans compute the answer by hand
3. **Widely-trusted libraries** - High confidence from extensive community usage (numpy, scipy)
4. **Cross-checking independent sources** - Multiple references that agree
5. **Obvious correctness** - For trivial tests (`assert len(result) == 3`), no verification needed

**When NOT to verify:** Shape checks, type checks, and tests with trivially-correct logic don't need verification. Only invest in verifying oracles that contain real logic.

**Example verification chain:**
```
Your GAE implementation
  └─ tested by: property test with reference_gae oracle
       └─ reference_gae verified by:
            ├─ Hand-computed cases for known simple scenarios
            ├─ Mathematical identity: A_t = δ_t + γλ(1-d_t)A_{t+1}
            └─ Cross-check against stable-baselines3 (trusted library)
```

### Step 6: Document in the Test File

The test file is the specification. Document claims where they're tested:

```python
"""
CLAIM: compute_advantages implements Generalized Advantage Estimation
       per Schulman et al. (2016)

PRECONDITIONS:
- rewards, values, dones are float tensors with matching batch dimensions
- gamma in [0, 1], lambda_ in [0, 1]

POSTCONDITIONS:
- Output shape equals rewards.shape
- Output matches reference implementation within rtol=1e-6
- Satisfies GAE recurrence: A_t = delta_t + gamma * lambda_ * (1 - done_t) * A_{t+1}

SEVERITY: High (property-based + reference oracle)
"""
```

## Test Design Patterns

### Pattern: Reference Implementation Testing

For algorithms with known-correct implementations:

```python
def reference_implementation(...):
    """Known-correct version (from trusted source, or deliberately simple/slow)."""
    # Simple, obviously-correct implementation
    ...

@given(valid_inputs())
def test_matches_reference(inputs):
    """Strong test: random inputs compared to reference."""
    result = implementation_under_test(inputs)
    expected = reference_implementation(inputs)
    assert_close(result, expected, rtol=1e-6)
```

### Pattern: Mathematical Identity Testing

For code that must satisfy mathematical properties:

```python
@given(valid_inputs())
def test_inverse_property(x):
    """encode then decode should return original."""
    assert decode(encode(x)) == x

@given(valid_inputs())
def test_commutativity(a, b):
    """operation should be commutative."""
    assert operation(a, b) == operation(b, a)

def test_identity_element():
    """zero should be identity for addition."""
    assert add(x, zero) == x
```

### Pattern: Gradient Checking (ML)

For neural network components:

```python
def test_gradient_correctness():
    """Finite difference gradient check."""
    x = torch.randn(..., requires_grad=True)
    assert torch.autograd.gradcheck(function_under_test, x, eps=1e-6)
```

### Pattern: Statistical Claim Testing

For stochastic code:

```python
def test_training_succeeds_reliably():
    """
    CLAIM: Training achieves target performance with >90% probability
    """
    successes = 0
    n_trials = 20
    for seed in range(n_trials):
        result = train_with_seed(seed)
        if result.score >= TARGET:
            successes += 1

    # Binomial test: is success rate consistent with >= 90%?
    assert successes >= 15, f"Only {successes}/{n_trials} succeeded"
```

### Pattern: Edge Case Enumeration

For functions with tricky boundaries:

```python
class TestEdgeCases:
    """Systematic edge case coverage."""

    def test_empty_input(self):
        """Empty input should return empty output (or raise ValueError)."""
        ...

    def test_single_element(self):
        """Single element - no aggregation needed."""
        ...

    def test_boundary_values(self):
        """Values at exact boundaries (0, 1, max_int, etc.)."""
        ...

    def test_large_input(self):
        """Performance/correctness at scale."""
        ...
```

## Domain-Specific Adversarial Patterns

### String/Text Processing

```python
ADVERSARIAL_STRINGS = [
    "",                          # empty
    " ",                         # whitespace only
    "  \t\n  ",                  # mixed whitespace
    "a" * 10_000_000,            # very long
    "\x00\x01\x02",              # null bytes and control chars
    "é" * 1000,                  # unicode requiring multiple bytes
    "🎉" * 1000,                 # emoji (4-byte unicode)
    "a\u0300",                   # combining characters (a + accent)
    "\u202e" + "secret",         # RTL override (text appears reversed)
    "AAAA%n%n%n%n",             # format string attack pattern
    "<script>alert(1)</script>", # XSS payload
    "'; DROP TABLE users; --",   # SQL injection
    "../../../etc/passwd",       # path traversal
]

@given(st.text(alphabet=st.characters(blacklist_categories=[]), min_size=0, max_size=10000))
def test_handles_arbitrary_unicode(text):
    """Must handle any valid unicode without crashing or corrupting."""
    result = process_text(text)
    # Key check: roundtrip preserves text, or transformation is well-defined
```

### API/Network Code

```python
class TestAPIAdversarial:
    """Adversarial conditions for network-dependent code."""

    def test_timeout_handling(self):
        """What happens when the server never responds?"""
        with mock_slow_server(delay=999):
            start = time.time()
            result = api_call(timeout=1)
            assert time.time() - start < 2  # Timeout actually works
            assert result.is_error  # Handled gracefully

    def test_partial_response(self):
        """Server sends half the data then disconnects."""
        with mock_truncated_response():
            # Should not crash, should not return corrupt data
            ...

    def test_malformed_json(self):
        """Server returns invalid JSON."""
        malformed = ['{"unclosed', '{"a":undefined}', '{trailing:,}', '\x00{"a":1}']
        for payload in malformed:
            with mock_response(payload):
                result = api_call()
                assert result.is_error  # Not crash, not silent corruption

    def test_retry_storms(self):
        """Ensure retries don't cause exponential request explosion."""
        with mock_failing_server() as server:
            api_call(retries=3)
            assert server.request_count <= 4  # Original + 3 retries, not more

    def test_concurrent_requests(self):
        """Race conditions under concurrent access."""
        results = []
        with ThreadPoolExecutor(max_workers=100) as ex:
            futures = [ex.submit(api_call) for _ in range(100)]
            results = [f.result() for f in futures]
        # All should succeed, no corruption, no deadlocks
```

### Data Structures

```python
class TestDataStructureAdversarial:
    """Adversarial patterns for custom data structures."""

    def test_hash_collision_attack(self):
        """Performance with pathological hash collisions."""
        # Generate keys that all hash to same bucket
        colliding_keys = generate_hash_collisions(n=10000)
        d = MyHashMap()
        start = time.time()
        for k in colliding_keys:
            d[k] = 1
        elapsed = time.time() - start
        # Should not degrade to O(n²) - or document that it does
        assert elapsed < 1.0, f"Pathological: {elapsed}s for {len(colliding_keys)} inserts"

    def test_iterator_invalidation(self):
        """Mutation during iteration."""
        container = MyContainer([1, 2, 3, 4, 5])
        with pytest.raises(RuntimeError):  # Should fail-fast, not corrupt
            for item in container:
                container.remove(item)

    def test_deep_nesting(self):
        """Deeply nested structures (recursion limits)."""
        deep = value = {}
        for _ in range(10000):
            value["nested"] = {}
            value = value["nested"]
        # Should handle or raise clear error, not segfault
        result = serialize(deep)

    @given(st.lists(st.integers(), min_size=0, max_size=10000))
    def test_maintains_invariants_under_random_operations(self, ops):
        """Invariants hold after arbitrary operation sequences."""
        structure = MyStructure()
        for op in ops:
            if op % 3 == 0:
                structure.insert(op)
            elif op % 3 == 1:
                structure.delete(op)
            else:
                structure.lookup(op)
            assert structure.check_invariants()  # After EVERY operation
```

### Concurrency

```python
class TestConcurrencyAdversarial:
    """Race conditions, deadlocks, and synchronization bugs."""

    def test_concurrent_writes(self):
        """Simultaneous writes shouldn't corrupt state."""
        shared = SharedState()
        errors = []

        def writer(id):
            try:
                for i in range(1000):
                    shared.update(id, i)
            except Exception as e:
                errors.append(e)

        threads = [Thread(target=writer, args=(i,)) for i in range(10)]
        for t in threads:
            t.start()
        for t in threads:
            t.join(timeout=10)  # Timeout catches deadlocks

        assert not errors
        assert all(t.is_alive() == False for t in threads)  # No deadlocks
        assert shared.is_consistent()  # No corruption

    def test_publish_subscribe_ordering(self):
        """Messages arrive in order under load."""
        received = []
        pubsub = PubSub()
        pubsub.subscribe(lambda msg: received.append(msg))

        sent = list(range(10000))
        with ThreadPoolExecutor(max_workers=10) as ex:
            list(ex.map(pubsub.publish, sent))

        # All messages received, in a valid order per publisher
        assert set(received) == set(sent)

    @given(st.lists(st.sampled_from(['read', 'write']), min_size=100))
    def test_no_deadlock_under_random_lock_patterns(self, operations):
        """Random read/write patterns don't deadlock."""
        lock = RWLock()
        def run_op(op):
            if op == 'read':
                with lock.read():
                    time.sleep(0.001)
            else:
                with lock.write():
                    time.sleep(0.001)

        with ThreadPoolExecutor(max_workers=20) as ex:
            futures = [ex.submit(run_op, op) for op in operations]
            for f in as_completed(futures, timeout=30):
                f.result()  # Timeout = deadlock detected
```

### File/IO Operations

```python
class TestFileAdversarial:
    """Adversarial filesystem conditions."""

    def test_disk_full(self):
        """Behavior when disk is full."""
        with simulated_full_disk():
            with pytest.raises(IOError):
                write_large_file()
            # Partial writes should be cleaned up
            assert not os.path.exists(temp_file_path)

    def test_permission_denied(self):
        """Graceful handling of permission errors."""
        with read_only_directory():
            result = safe_write("test.txt", "data")
            assert result.is_error
            assert "permission" in result.error.lower()

    def test_symlink_attacks(self):
        """Symlinks shouldn't escape sandbox."""
        os.symlink("/etc/passwd", "innocent.txt")
        with pytest.raises(SecurityError):
            read_file("innocent.txt")

    def test_concurrent_file_access(self):
        """Multiple processes reading/writing same file."""
        # Uses file locking properly, no corruption
```

### Numerical/Scientific Code

```python
class TestNumericalAdversarial:
    """Numerical edge cases that break naive implementations."""

    ADVERSARIAL_FLOATS = [
        0.0, -0.0,                           # signed zero
        float('inf'), float('-inf'),         # infinities
        float('nan'),                        # NaN
        sys.float_info.min,                  # smallest positive normal
        sys.float_info.max,                  # largest finite
        sys.float_info.epsilon,              # smallest x where 1+x != 1
        2.2250738585072014e-308,            # smallest positive normal
        4.9406564584124654e-324,            # smallest positive subnormal
        1.0000000000000002,                  # 1 + epsilon
        0.1 + 0.2,                           # famously != 0.3
    ]

    def test_catastrophic_cancellation(self):
        """Subtraction of nearly-equal numbers."""
        a = 1.0000000000000001
        b = 1.0000000000000000
        # Naive: a - b loses all precision
        # Robust implementations use compensated algorithms
        result = subtract_accurately(a, b)
        assert abs(result - 1e-16) < 1e-17

    @given(st.floats(allow_nan=True, allow_infinity=True))
    def test_handles_all_floats(self, x):
        """Function handles any float without crashing."""
        result = function_under_test(x)
        # NaN in -> NaN out (or documented error), not crash
        if math.isnan(x):
            assert math.isnan(result) or isinstance(result, Error)

    def test_accumulation_error(self):
        """Error accumulation over many operations."""
        # Naive summation accumulates error
        values = [0.1] * 1_000_000
        result = accurate_sum(values)
        expected = 100000.0
        # Kahan summation should achieve near-full precision
        assert abs(result - expected) < 1e-10

    def test_matrix_conditioning(self):
        """Behavior on ill-conditioned matrices."""
        # Near-singular matrix
        ill_conditioned = np.array([[1, 1], [1, 1.0001]])
        result = matrix_operation(ill_conditioned)
        # Should either succeed with documented precision loss,
        # or raise a clear error - not silently return garbage
```

## Severity Calibration Guide

**Light (simple utilities):**
```python
def test_basic():
    assert util_function(typical_input) == expected_output

def test_edge():
    assert util_function(edge_case) == expected_edge_output
```

**Moderate (standard logic):**
```python
"""
CLAIM: [One sentence describing what the code does]
POSTCONDITIONS: [Key properties that must hold]
"""

def test_typical_cases():
    ...

def test_edge_cases():
    ...

@given(valid_inputs())
def test_properties(input):
    # At least one property-based test
    ...
```

**High (algorithms, ML, numerical):**
```python
"""
CLAIM: [Precise statement with mathematical grounding if applicable]
PRECONDITIONS: [Input requirements]
POSTCONDITIONS: [Output guarantees]
SEVERITY: High
"""

def reference_impl(...):
    """Trusted reference."""
    ...

@given(valid_inputs())
def test_reference_match(input):
    ...

def test_mathematical_identities():
    ...

def test_numerical_stability():
    # Edge cases that stress numerical precision
    ...
```

**Maximum (critical code):**

All of the above, plus:

```python
"""
ORACLE VERIFICATION:
The reference implementation is verified by:
1. Hand-computed test cases (see test_reference_analytic)
2. Cross-check against [trusted library]
3. Mathematical definition verification
"""

class TestOracleCorrectness:
    """Tests for the reference implementation itself."""

    def test_reference_analytic(self):
        """Hand-computed cases where we know the answer."""
        ...

    def test_reference_cross_check(self):
        """Compare reference to another trusted source."""
        ...
```

## Common Pitfalls to Avoid

1. **Trivial tests**: Testing default values, constructor existence, type annotations. These are waste.
2. **Confirmation bias**: Writing tests that can only pass, not fail. Ask: "What bug would this catch?"
3. **Weak oracles**: Only checking that code doesn't crash. Crashes are easy; silent corruption is the danger.
4. **Happy path only**: Missing edge cases where bugs hide. The bug is never at `add(2, 2)`.
5. **Insufficient fuzzing**: A handful of handpicked inputs vs. thousands of random ones. Use hypothesis/property-based testing.
6. **Untested test code**: Complex test logic that could itself be wrong
7. **Brittle tests**: Tests that break on irrelevant changes (testing implementation, not behavior)
8. **Missing preconditions**: Not documenting what inputs are valid
9. **No discovery**: Tests that only confirm, never reveal. If tests never surprise you, they're too weak.
10. **Narrow input ranges**: Testing with "nice" numbers (0, 1, 10) instead of adversarial ones (MAX_INT, epsilon, NaN)

## Quick Reference

After writing code, ask:

1. **What does it claim?** Write it down precisely.
2. **What would refute it?** List concrete failure conditions.
3. **How severe are my tests?** Would a buggy implementation pass? If yes, test is too weak.
4. **Am I being adversarial?** Am I actively trying to break this, or just confirming it works?
5. **What did I learn?** Tests should reveal bounds, limits, edge behaviors - not just pass/fail.
6. **Is my oracle trustworthy?** For critical code, verify the verifier.
7. **Did I avoid trivial tests?** No testing defaults, constructors, or type annotations.

**The goal:** After testing, your claim should be MORE PRECISE than before. You should know the actual performance envelope, numerical limits, and edge behaviors - not just "it works."

---

# Granite Severe Testing Matrix

This section is the Granite-specific severe-test companion to
`/Users/chabotc/Projects/granite/specs/product/24_acceptance_criteria.md`.
Each row states the falsifiable claim, the strongest practical oracle, and the
test version that should exist before the spec is called complete.

## Required Gates

Every implementation sweep must run:

- `bun run typecheck`
- `bun run test`
- `bun run build`
- `bun run lint`
- Browser smoke test for UI-visible work
- Completion audit mapping each changed spec item to code, test, and manual evidence

## Acceptance Criteria Severe Tests

| Spec | Claim | Severe test version | Oracle / refutation |
|---|---|---|---|
| 24.1 Vault/files | File operations preserve content, metadata, links, and config semantics under adversarial vault shapes. | Generate vaults with 1, 10k, and 100k files, mixed extensions, excluded folders, external edits, rename storms, delete modes, and interrupted saves. | Canonical file tree diff, link graph diff, fs-event latency histogram, no partial files after kill. |
| 24.2 Editor | Source, Live Preview, Reading, Vim, multi-cursor, rectangular selection, and folding behave per CodeMirror/Obsidian semantics. | CM6 integration fixture that simulates mode toggles, Vim normal/insert/visual operations, Alt-click cursors, Shift-Alt drag rectangles, fold/unfold, reload, and marker hiding on inactive lines. | DOM selection/range oracle, persisted workspace JSON, rendered marker-range oracle, hand-authored markdown examples. |
| 24.3 Parser fidelity | Markdown rendering matches CommonMark/GFM plus documented Obsidian extensions. | Import official CommonMark and GFM fixtures; add adversarial Obsidian cases for nested callouts, math, Mermaid, comments, footnotes, embeds, HTML blocks. | Official expected HTML where available; canonical rendered DOM snapshots for Obsidian extensions. |
| 24.4 Linking/metadata | Links, aliases, headings, blocks, backlinks, outgoing links, and exclusions agree across all surfaces. | Fixture vault with duplicate stems, aliases, headings with punctuation, block IDs, excluded paths, renamed files, and unresolved links. | Independent metadata-index oracle built from raw markdown; UI counts must match oracle exactly. |
| 24.5 Properties | Frontmatter edits preserve meaning, quoting, type registry, legacy migration, and YAML canonicalization. | Fuzz YAML/JSON frontmatter with links, quoted strings, lists, dates, tags, singular legacy keys; round-trip through UI and format converter. | YAML AST equivalence and raw-string invariants for link-containing Text/List values. |
| 24.6 Tags | Tags unify across body/YAML, nest only when configured, reject numeric-only tags, preserve display case. | Unicode/case/numeric/nested tag corpus across body and properties, with toggle permutations. | Tag tree oracle from parser plus explicit rejected-token list. |
| 24.7 Search | Search operators, regex, properties, sort, case, embedded query blocks, and performance satisfy spec. | 10k-note generated vault with adversarial regex metacharacters, negation, property nulls, subqueries, duplicate lines, and embedded queries. | Slow reference search engine over raw files; first-paint and regex latency budgets. |
| 24.8 Graph | Graph nodes, edges, filters, groups, colors, force/display sliders, local graph, and state persistence are correct and performant. | Synthetic vaults with known graph topology at 100, 10k nodes; mutate filters/groups/sliders; pan for 10 seconds. | Reference adjacency list, persisted graph config JSON, frame-budget telemetry >= 30 fps. |
| 24.9 Canvas | JSON Canvas round-trips and interactions preserve geometry, edges, anchors, embeds, and gestures. | Fixture canvases from another conforming app; simulate pan/zoom/marquee/multi-select/snap/duplicate/axis lock/embedded interaction. | Canonical JSON diff and DOM/selection geometry oracle. |
| 24.10 Bases | Table/List/Cards/Map, filters, sort, group, formulas, summaries, embeds, and round-trip behavior match `.base` semantics. | Fixture `.base` files with formulas, invalid formulas, nulls, groups, summaries, embedded `this`, and text edits. | Independent evaluator and canonical `.base` JSON/YAML diff. |
| 24.11 Workspace/tabs | Splits, stacked groups, pop-outs, workspaces, pins, and sidebar pinning preserve layout intent. | Random workspace-operation sequences with split/move/pin/close/restore across reloads and windows. | Workspace model invariant checker plus persisted snapshot diff. |
| 24.12 Sidebars | Sidebars collapse, reorder, pop out, split vertically, and keep vault profile behavior stable. | Drag/reorder/pop-out/split interaction matrix in browser tests. | DOM order, active target, and workspace-state oracle. |
| 24.13 Status bar | Word count, mode chip, and plugin status items update and clean up accurately. | CJK/Latin/mixed note corpus; plugin add/update/remove lifecycle; mode switches. | Independent tokenizer, active leaf state, plugin registry lifecycle oracle. |
| 24.14 Hotkeys | Defaults, system shortcuts, custom bindings, conflicts, multi-binding, and physical-key normalization work. | Keyboard event matrix for macOS/Windows/Linux layouts, conflicts, duplicate bindings, editor vs global focus. | Command invocation log and expected normalized-display table. |
| 24.15 Settings | Every setting exists, defaults match spec, filters live, plugin pages gate by enabled state, and settings persist to vault config. | Enumerate spec settings against UI controls; mutate every setting, reload, disable plugins, switch vaults. | Settings schema oracle and `.granite/` config diff. |
| 24.16 Themes/snippets | Theme, accent, light/dark, snippets, and snippet watching apply instantly and safely. | Toggle theme matrix; write/delete snippet files; invalid CSS; high-contrast/accent permutations. | Computed CSS variable snapshots and watcher latency budget. |
| 24.17 Plugins | Restricted mode, browse/install/enable, lifecycle cleanup, data persistence, update checks, and min-version handling are compatible. | Fixture registry with three plugins covering commands, status, settings, events, data, teardown, update mismatch. | Plugin API call log, filesystem diff, registry oracle, cleanup leak detector. |
| 24.18 Drag/drop | Every source x destination drag behavior matches spec, including external OS/browser modifiers. | Playwright drag matrix for files, tabs, sidebars, editor embeds, canvas cards, external files and URLs with Ctrl/Option. | Workspace/file-tree diff and visible drop-zone assertion. |
| 24.19 Performance | All published budgets hold under realistic load. | Generated 1k/10k/50k/100k vaults; typing a 100k-character note; 4-hour soak profile. | Perf telemetry with p95/p99 latency, frame drops, memory trend, and budget assertions. |
| 24.20 Accessibility | Core flows are keyboard reachable, announced, labeled, high-contrast, and reduced-motion safe. | Axe/Lighthouse plus `verify:keyboard-browser`, `verify:keyboard-populated-browser`, `verify:icon-a11y-browser`, `verify:a11y-announcements-browser`, and `verify:theme-contrast-browser` across vault picker, editor, modals, menus, graph, canvas, settings, notices. | Accessibility tree snapshots, focus-order trace, contrast calculator, reduced-motion computed styles. |
| 24.21 i18n/RTL | All visible strings externalize; RTL locale and per-note direction flip correctly. | Pseudo-locale expansion, Hebrew demo locale, `dir: rtl` notes, mixed LTR/RTL content. | Missing-key scanner, screenshot/DOM direction oracle, date-locale assertions. |
| 24.22 Crash safety | Restart restores workspace and recovery snapshots without data loss. | 100 random kill-and-restart cycles during save-heavy edits with injected write failures. | Atomic-write invariant, snapshot availability, canonical file-content diff. |
| 24.23 Compatibility | Existing Obsidian vaults, `.canvas`, `.base`, notes, and themes open and round-trip without semantic change. | 200-note Obsidian fixture with themes/plugins disabled; edit subset in Granite; reopen in Obsidian-compatible parser. | Canonical semantic diff for markdown/frontmatter/canvas/base plus theme screenshot comparison. |

## Current Phase 12 Severe Tests

For the editor fidelity work, the minimum severe test set is:

- Unit: `src/core/markdown/cm-livepreview-decorations.test.ts` must refute wrong hidden-marker ranges for bold, italic, nested bold inside asterisk italic, nested asterisk italic inside bold, nested underscore italic inside underscore bold, nested underscore bold inside underscore italic, highlight, strikethrough, wikilinks, embeds, Markdown links/images, footnote-reference marker chrome, comments, multiline block comments, math, callouts, heading markers, horizontal-rule lines, standard and custom task checkbox markers, paragraph and standalone block-id markers, GFM table pipes/separators, cursor-line raw behavior, fenced code opening/closing markers, top-of-file frontmatter no-parse behavior, single-backtick and multi-backtick inline code, same-line and multiline HTML element no-parse behavior, and escaped inline formatting markers.
- Unit: `src/core/workspace/store.test.ts` must prove new Markdown leaves use `defaultEditingMode` for Editing view, Reading view remains honored when configured, and a `source` leaf is not treated as Live Preview merely because the global settings store has Live Preview defaults.
- DOM integration: `src/core/markdown/cm-livepreview-decorations.test.ts` must mount the real `livePreviewDecorations` CodeMirror extension and prove non-cursor inline, table, and callout markers disappear from rendered editor text while the cursor line stays raw.
- Browser: `bun run verify:live-preview-browser` must launch Chromium against the Vite-served fixture and prove inactive Live Preview hides bold, wikilink, Markdown-link, and table separator source markers while Source mode keeps them raw and the active Live Preview line reveals them again.
- Unit: `src/core/workspace/folds.test.ts` must refute invalid, duplicate, out-of-range, or unsorted persisted fold ranges.
- Integration: `src/core/workspace/persist.test.ts` must prove markdown leaf state survives workspace snapshot restore.
- Browser: `bun run verify:fold-persistence-browser` must fold a heading, reload, verify it remains folded, then unfold and verify the persisted range clears.
- Browser: `bun run verify:vim-mode-browser` must toggle Vim mode, enter insert text with `i`, leave insert mode with `Esc`, and verify normal-mode navigation does not mutate text.
- Browser: `bun run verify:multi-cursor-browser` must Alt-click multiple cursors and Shift-Alt drag a rectangular selection, then type once and verify every selected range changes.

## Current 24.1 Trash Severe Tests

For accepted native file formats, the minimum severe test set is:

- Unit: `src/core/fs/file-formats.test.ts` must prove the native format classifier covers every extension in `20_file_storage.md` §20.2 and does not classify unsupported extensions as native app-opened formats.
- Unit: `src/core/workspace/store.test.ts` must prove `workspaceStore.openPath()` routes `.canvas` and `.base` to their dedicated leaves and routes image/audio/video/PDF formats to asset leaves with the correct category.
- Unit: `src/ui/views/ReadingView.test.tsx` must remain green while Reading mode media embeds use the same shared classifier and mime lookup as direct asset opening.
- Build/typecheck: `bun run build` must prove the new asset leaf state is accepted by workspace rendering, native history, persistence types, and File Explorer routing.
- Browser: `bun run verify:native-formats-browser` must open one file from each native category (`.md`, `.canvas`, `.base`, image, audio, video, `.pdf`) through the workspace router and verify the rendered view matches the file type.

For configurable deletion behavior, the minimum severe test set is:

- Unit: `src/core/fs/trash.test.ts` must prove permanent mode calls only `remove()` and never writes `.trash/`.
- Unit: `src/core/fs/trash.test.ts` must prove vault-trash mode moves `dir/file.ext` to `.trash/dir/file.ext` while preserving the original relative subpath.
- Unit: `src/core/fs/trash.test.ts` must prove vault-trash collisions choose a new deterministic filename instead of overwriting an existing trashed file.
- Unit: `src/core/fs/trash.test.ts` must prove browser/system-trash unsupported mode does not silently fall back to permanent delete.
- Unit: `src/core/fs/trash.test.ts` must prove a host-provided `FileSystemImpl.moveToSystemTrash()` is used when present.
- Unit: `src/core/fs/native-trash.test.ts` must prove only the trusted `window.graniteHost.fs.moveToSystemTrash()` bridge is detected as a native system-trash capability.
- Unit: `src/core/fs/handle-adapter.test.ts` must prove `handleAdapter()` exposes `moveToSystemTrash()` only when a native host bridge is supplied or detected.
- Unit: `src/core/fs/handle-adapter.test.ts` must prove folder-pick unavailable, folder-permission denied, and OPFS-unavailable browser capability failures throw coded errors instead of user-visible English strings from the adapter.
- Browser: `bun run verify:trash-settings-browser` must toggle Confirm file deletion and Deleted files through Settings, then delete from File Explorer and verify the confirmation text plus resulting notice for Vault trash, Permanent deletion, unsupported System trash in a browser OPFS vault, and host-backed System trash through a mocked `window.graniteHost.fs.moveToSystemTrash()` bridge.
- Browser/host-contract: the host-backed System trash browser case must prove File Explorer calls the bridge with the vault root name and relative path, the vault copy is absent after the host bridge resolves, and no `.trash/` directory is created. Packaged native-host smoke remains the only valid evidence for the OS recycle bin/trash placement itself.

## Current 24.1 Multi-Window Vault Severe Tests

For simultaneous multi-window vault behavior, the minimum severe test set is:

- Unit: `src/core/vault/window-url.test.ts` must prove standalone vault-window URLs set `vaultWindow=1`, preserve the requested `vaultId`, and strip inherited pop-out leaf state.
- Unit: `src/core/vault/window-url.test.ts` must prove `?vaultWindow=1&vaultId=...` parses as a standalone vault-window request.
- Unit: `src/core/vault/window-url.test.ts` must prove existing `?popout=1&vaultId=...&leaf=...` links still parse as leaf pop-out requests.
- Unit: `src/core/vault/window-url.test.ts` must reject window requests without a vault id.
- Unit: `src/core/vault/permissions.test.ts` must prove a stored FSA handle with read/write permission opens directly, a prompt-state handle calls `requestPermission({ mode: "readwrite" })`, a granted request succeeds, a denied request fails, and legacy handles without permission APIs remain usable.
- Browser: `bun run verify:multi-window-vault-browser` must register two OPFS vaults, open Vault Picker, click "Open in new window" for the inactive vault, verify the original window keeps its current active vault, and verify the new window opens the requested vault.
- Browser: `bun run verify:multi-window-vault-browser` must right-click a workspace tab, choose "Open in new window", and verify the pop-out bootstraps the original vault plus the requested markdown leaf instead of only opening the vault root window.
- Architecture: `src/ui/vault/VaultContext.tsx` must route both standalone vault-window FSA bootstrap and in-window FSA reopen through `ensureReadwritePermission()` so the same tested permission handshake protects both paths.

## Current 24.12 Sidebar Pop-Out Severe Tests

For sidebar tabs opened in the central workspace, the minimum severe test set is:

- Unit: `src/core/workspace/sidebar-view.test.ts` must prove a sidebar tab can open as a central `sidebar` workspace leaf with side and tab id preserved.
- Unit: `src/core/workspace/sidebar-view.test.ts` must prove opening the same sidebar tab again focuses the existing central leaf instead of duplicating it.
- Integration: `src/core/workspace/persist.test.ts` must prove sidebar leaf state round-trips through workspace snapshot restore.
- Browser: `bun run verify:sidebar-central-browser` must activate one tab in each sidebar, click "Open in central area", verify the matching views appear as central tabs with expected content, verify the original sidebar remains independently usable, flush/reload the workspace, and verify the central sidebar leaves restore.

## Current 24.12 Sidebar Vertical Group Severe Tests

For multiple vertical groups within a sidebar, the minimum severe test set is:

- Unit: `src/ui/shell/sidebar-groups.test.ts` must prove changing a tab in one group does not mutate any sibling group.
- Unit: `src/ui/shell/sidebar-groups.test.ts` must prove splitting inserts a new group below the source and copies the source active tab.
- Unit: `src/ui/shell/sidebar-groups.test.ts` must prove the last remaining sidebar group cannot be closed.
- Unit: `src/ui/shell/sidebar-groups.test.ts` must prove closing a non-final group removes only that group.
- Browser: `bun run verify:sidebar-groups-browser` must split the left sidebar twice, set each group to a different tab, verify all groups remain visible and independently interactive, split/close a right sidebar group while preserving the remaining active tab, and verify split/close icon-only buttons expose accessible names.

## Current 24.6 Tags Severe Tests

For nested tag display behavior, the minimum severe test set is:

- Unit: `src/ui/views/sidebar/tags-model.test.ts` must prove `work/client` and `work/internal` aggregate under `work` when nested display is enabled.
- Unit: `src/ui/views/sidebar/tags-model.test.ts` must prove slash-separated tags remain flat full-name rows when nested display is disabled.
- Unit: `src/ui/views/sidebar/tags-model.test.ts` must prove sorting is deterministic by count descending, then tag text ascending.
- Browser: `bun run verify:tags-browser` must toggle the Tags sidebar Show nested tags checkbox, verify the visible tree updates immediately, reload to prove the setting persists, and verify clicking a flat slash tag and a nested child tag both dispatch the same `tag:<fullName>` search query.
- Unit: `src/core/metadata/parser.test.ts` must prove body tags and YAML tags deduplicate case-insensitively within one note while preserving the first display casing.
- Unit: `src/core/metadata/cache.test.ts` must prove vault-wide tag counts aggregate `Work`/`work`/`WORK` case-insensitively and preserve first display casing.
- Unit: `src/core/metadata/cache.test.ts` must prove repeated case variants in one file count only once for the Tags view.

## Current 24.4/24.5 Properties Severe Tests

For property display and frontmatter round-trip behavior, the minimum severe test set is:

- Unit: `src/core/metadata/property-format.test.ts` must prove ISO date strings render differently under `en-US` and `en-GB`.
- Unit: `src/core/metadata/property-format.test.ts` must prove YAML `Date` objects at UTC midnight render as date-only values without off-by-one timezone drift.
- Unit: `src/core/metadata/property-format.test.ts` must prove ISO datetime strings render localized date and time, not raw ISO text.
- Unit: `src/core/metadata/frontmatter.test.ts` must prove internal wikilinks in Text properties are still quoted after `updateFrontmatterValue`.
- Unit: `src/core/metadata/frontmatter.test.ts` must prove internal wikilinks in List properties are still quoted after `updateFrontmatterValue`.
- Unit: `src/core/metadata/frontmatter.test.ts` must prove JSON-style frontmatter inside `---` fences rewrites to YAML after a property edit.
- Browser: `bun run verify:properties-browser` must render a note in Reading mode and verify Date plus Date & time fields display in the browser locale, then render the same note in Source mode and verify canonical YAML values remain unchanged.

## Current 24.5 Format Converter Severe Tests

For legacy default-property migration, the minimum severe test set is:

- Unit: `src/core/plugins-core/format-converter.test.ts` must prove singular `tag` migrates to `tags` as a list and strips a leading `#`.
- Unit: `src/core/plugins-core/format-converter.test.ts` must prove singular `alias` migrates to `aliases` and singular `cssclass` migrates to `cssclasses`.
- Unit: `src/core/plugins-core/format-converter.test.ts` must prove migration merges with existing plural keys without duplicating values.
- Unit: `src/core/plugins-core/format-converter.test.ts` must prove files without legacy keys are left byte-identical.
- Browser: `bun run verify:format-converter-browser` must run the registered Format Converter migration command, verify every Markdown file is scanned, verify only changed Markdown files are written, verify non-Markdown files are untouched, and verify the reported note/key counts.

## Current 24.9 Canvas Snap Severe Tests

For Canvas snap-toggle behavior, the minimum severe test set is:

- Unit: `src/core/canvas/interactions.test.ts` must prove snap-on coordinates round to the configured grid.
- Unit: `src/core/canvas/interactions.test.ts` must prove snap-off coordinates round only to whole pixels, preserving sub-grid placement.
- Unit: `src/core/canvas/interactions.test.ts` must prove invalid/non-finite coordinates cannot leak into saved canvas geometry.
- Unit: `src/core/canvas/interactions.test.ts` must prove keyboard movement uses 10 px / 50 px steps when snapping is enabled.
- Unit: `src/core/canvas/interactions.test.ts` must prove keyboard movement uses 1 px / 10 px steps when snapping is disabled.
- Browser: `bun run verify:canvas-snap-browser` must toggle Snap to grid off, drag and resize a node to non-10 px coordinates, reload the canvas, and verify the JSON geometry preserves those coordinates.
- Browser: `bun run verify:canvas-snap-browser` must toggle Snap to grid on, drop a file, drag a node, resize a node, and use arrow keys; all resulting coordinates and sizes must land on the 10 px grid.

## Current 24.9 Canvas Marquee/Duplicate Severe Tests

For Canvas marquee, multi-select, duplicate, and axis-lock behavior, the minimum severe test set is:

- Unit: `src/core/canvas/interactions.test.ts` must prove reverse marquee drags normalize to a positive rectangle.
- Unit: `src/core/canvas/interactions.test.ts` must prove marquee selection includes every node whose box intersects the rectangle and excludes non-intersecting nodes.
- Unit: `src/core/canvas/interactions.test.ts` must prove Shift-axis-lock keeps the dominant drag axis and zeroes the weaker axis.
- Unit: `src/core/canvas/interactions.test.ts` must prove Alt/Option duplicate assigns fresh ids to every duplicated node.
- Unit: `src/core/canvas/interactions.test.ts` must prove Alt/Option duplicate clones only internal selected edges and rewires them to duplicated endpoint ids.
- Browser: `bun run verify:canvas-marquee-browser` must Shift-drag an empty canvas region across two cards, verify both cards show selected resize handles, then press Cmd/Ctrl+Backspace and verify both cards and their connecting edge are removed.
- Browser: `bun run verify:canvas-marquee-browser` must marquee-select multiple cards, drag one selected card, and verify every selected card moves by the same delta while unselected cards remain fixed.
- Browser: `bun run verify:canvas-marquee-browser` must marquee-select multiple cards, Shift-drag diagonally, and verify movement is constrained to the dominant axis.
- Browser: `bun run verify:canvas-marquee-browser` must marquee-select two connected cards, Alt/Option-drag one selected card, and verify duplicated cards plus their internal edge are created with new JSON ids while the original cards and edge remain unchanged.

## Current 24.9 Canvas Embed Severe Tests

For embedded Canvas behavior inside host notes, the minimum severe test set is:

- Integration: `src/ui/views/ReadingView.test.tsx` must prove `![[file.canvas]]` mounts `.canvas-view` inside `.canvas-embed.is-interactive`, not a click-only card.
- Integration: `src/ui/views/ReadingView.test.tsx` must prove the embed header uses the target canvas path and parsed node/edge counts.
- Browser: `bun run verify:canvas-embed-browser` must create a note containing `![[board.canvas]]`, render it in Reading mode, and verify the embedded canvas displays nodes, zoom controls, snap control, and add-text control inside the note.
- Browser: `bun run verify:canvas-embed-browser` must interact with an embedded canvas by panning, zooming, selecting, and dragging a card; verify the host note does not navigate away or replace the embed.
- Browser: `bun run verify:canvas-embed-browser` must edit geometry inside the embedded canvas, wait for the debounce save, reopen `board.canvas` as JSON, and verify the saved geometry matches the interaction.
- Browser: `bun run verify:canvas-embed-browser` must click the embedded canvas Open button with and without Cmd/Ctrl and verify it opens the same `.canvas` in the expected current/new tab target.
- Browser: `bun run verify:canvas-embed-browser` must edit the host markdown so the canvas embed is removed, then verify no orphaned embedded canvas toolbar, event handlers, or console errors remain.

## Current 24.14 Hotkey Multi-Binding Severe Tests

For hotkey multi-binding behavior, the minimum severe test set is:

- Unit: `src/core/commands/hotkeys.test.ts` must prove two custom bindings on one command both dispatch that command.
- Unit: `src/core/commands/hotkeys.test.ts` must prove adding the same custom binding twice does not create duplicate stored bindings.
- Unit: `src/core/commands/hotkeys.test.ts` must prove removing one custom binding does not clear unrelated bindings for the command.
- Unit: `src/core/commands/hotkeys.test.ts` must prove default command hotkeys are restored after all custom bindings are cleared.
- Browser: `bun run verify:hotkeys-browser` must show every effective binding for a command as a comma-separated list in Settings → Hotkeys.
- Browser: `bun run verify:hotkeys-browser` must add a second binding in Settings → Hotkeys while keeping the first binding active and visible.
- Browser: `bun run verify:hotkeys-browser` must remove the most recent binding while leaving older custom bindings active, then Reset must clear all custom bindings.
- Browser: `bun run verify:hotkeys-browser` must prove a command with custom bindings does not also fire from its default binding until Reset restores defaults.

## Current 24.14 Hotkey Physical-Key Severe Tests

For US-layout physical-key hotkey behavior, the minimum severe test set is:

- Unit: `src/core/commands/hotkeys.test.ts` must prove a stored `Q` binding fires from `KeyboardEvent.code === "KeyQ"` even when `KeyboardEvent.key` is a different character.
- Unit: `src/core/commands/hotkeys.test.ts` must prove a stored `Q` binding does not fire from a different physical letter slot whose `KeyboardEvent.key` happens to be `Q`.
- Unit: `src/core/commands/hotkeys.test.ts` must prove top-row punctuation such as `Backquote` normalizes to the US-layout display key.
- Browser: `bun run verify:hotkeys-browser` must assign a hotkey from Settings using a simulated non-US physical key event and verify the displayed binding uses the US physical key label.
- Browser: `bun run verify:hotkeys-browser` must trigger the command from the same physical key position after capture even when the produced character differs.
- Browser: `bun run verify:hotkeys-browser` must verify semantic keys such as arrows still use their semantic key names and are not rewritten through the US-layout map.

## Current 24.15 Settings Persistence Severe Tests

For settings section coverage, the minimum severe test set is:

- Unit: `src/ui/prompts/settings-filter.test.ts` must prove every built-in Options section required by the settings spec, including About, appears for an empty query.
- Unit: `src/ui/prompts/settings-filter.test.ts` must prove the About section is discoverable by version/license search terms.
- Unit: `src/ui/prompts/SettingsModal.test.tsx` must render Settings, open About from the sidebar, and prove version, license, and credits rows appear with the shared `APP_VERSION`.
- Unit: `src/core/i18n/externalization.test.ts` must prove About labels and rendered section title are routed through i18n keys.

For settings persistence to the vault config folder, the minimum severe test set is:

- Unit: `src/core/settings/store.test.ts` must prove binding a vault with no settings writes the full `DEFAULT_SETTINGS` object to `.granite/settings.json`.
- Unit: `src/core/settings/store.test.ts` must prove `.granite/settings.json` wins over legacy `granite.settings.v1` localStorage when both exist.
- Unit: `src/core/settings/store.test.ts` must prove legacy localStorage settings migrate into `.granite/settings.json` when the disk file is missing.
- Unit: `src/core/settings/store.test.ts` must prove `settingsStore.update()` writes the active vault's `.granite/settings.json` after settings are bound.
- Browser: `bun run verify:settings-persistence-browser` must open a fresh vault, verify `.granite/settings.json` is created, and confirm it contains every default setting key.
- Browser: `bun run verify:settings-persistence-browser` must change Settings → Appearance font size, reload the vault, and verify the setting is restored from `.granite/settings.json` without relying on browser global state.
- Browser: `bun run verify:settings-persistence-browser` must open two different vaults with different `.granite/settings.json` values and verify switching vaults hydrates the active vault's own settings.

## Current 24.3 GFM Parser Severe Tests

For GFM parser behavior, the minimum severe test set is:

- Unit: `src/core/markdown/renderer.test.ts` must prove `~~strikethrough~~` renders as `<s>`.
- Unit: `src/core/markdown/renderer.test.ts` must prove pipe tables render as `<table>` with left and right column alignment preserved.
- Unit: `src/core/markdown/renderer.test.ts` must prove bare URL autolinks render as anchors and do not absorb trailing punctuation.
- Unit: `src/core/markdown/renderer.test.ts` must prove bare email autolinks render with `mailto:`.
- Unit: `src/core/markdown/renderer.test.ts` must prove CommonMark angle URL and email autolinks render correctly next to punctuation.
- Unit: `src/core/markdown/renderer.test.ts` must prove custom task markers such as `[?]` and `[-]` render as task-list checkboxes and preserve their marker state.
- Browser: `bun run verify:gfm-browser` must render a note containing a table, strikethrough, bare URL, bare email, angle autolinks, `[?]` task, and `[-]` task in Reading mode and verify the DOM matches the unit-test expectations.

## Current 24.3 CommonMark Conformance Severe Tests

For CommonMark conformance, the minimum severe test set is:

- Fixture: `src/core/markdown/fixtures/commonmark-0.31.2.json` must be the official CommonMark 0.31.2 JSON test suite from `https://spec.commonmark.org/0.31.2/spec.json`.
- Unit: `src/core/markdown/commonmark-conformance.test.ts` must prove the official fixture contains more than 600 examples and starts at example 1, so an empty or stub fixture cannot pass.
- Unit: `src/core/markdown/commonmark-conformance.test.ts` must run every fixture example through `renderCommonMark()`.
- Unit: `src/core/markdown/commonmark-conformance.test.ts` must assert the pass rate is at least 99%.
- Unit: `src/core/markdown/commonmark-conformance.test.ts` must print failing example numbers and sections in the assertion message so regressions are actionable.
- Unit: `src/core/markdown/renderer.test.ts` must continue proving Granite extensions such as wikilinks, embeds, tags, comments, callouts, GFM tables, and task markers still render through the app renderer.
- Browser: `bun run verify:commonmark-browser` must render representative CommonMark examples for headings, lists, blockquotes, links, code fences, and raw HTML in a development harness and compare the HTML against the official expected output.
- Browser: `bun run verify:commonmark-browser` must render Granite-specific Markdown extensions in Reading mode after the CommonMark harness runs to verify the base conformance path did not disable app extensions.

## Current 24.21 RTL Severe Tests

For RTL locale and per-note direction behavior, the minimum severe test set is:

- Unit: `src/core/i18n/direction.test.ts` must prove `he` and regional variants such as `he-IL` resolve to RTL while English resolves to LTR.
- Unit: `src/core/i18n/direction.test.ts` must prove only `dir: rtl` and `dir: ltr` frontmatter values are accepted for per-note direction.
- Unit: `src/core/i18n/index.test.ts` must prove Hebrew is a built-in locale with translated settings/app labels.
- Integration: `src/ui/LocaleDirectionBinder.test.tsx` must prove switching to `he` sets `document.documentElement.dir = "rtl"` and adds RTL body classes.
- Integration: `src/ui/views/ReadingView.test.tsx` must prove `dir: rtl` frontmatter adds the `rtl` class and `dir="rtl"` to `.markdown-rendered`, and that notes without direction frontmatter remain explicit LTR under RTL chrome.
- Integration: `src/ui/views/sidebar/PropertiesView.test.tsx` must prove native date and datetime property inputs receive the active locale through `lang`, including YAML-parsed `Date` values.
- Browser: `bun run verify:rtl-browser` must switch locale from English to Hebrew and verify modal/menu/status/tab chrome uses RTL direction while canvas remains spatially LTR.
- Browser: `bun run verify:rtl-browser` must verify Properties sidebar date and datetime pickers use the active locale through `lang`.
- Browser: `bun run verify:rtl-browser` must open one note with `dir: rtl` and one without it, verifying only the opted-in note content flips in Reading and Source modes.
- Browser: `bun run verify:rtl-browser` must mix Hebrew and English paragraphs in an RTL note and verify paragraph direction remains readable because user-content surfaces keep plaintext bidi behavior.

## Current 24.10 Bases Map Severe Tests

For Bases Map behavior, the minimum severe test set is:

- Unit: `src/core/bases/schema.test.ts` must prove `view: map` parses as a first-class Base view type.
- Unit: `src/core/bases/schema.test.ts` must prove `mapLatitude` / `mapLongitude` round-trip through serialize/parse.
- Unit: `src/ui/views/bases/BasesMapView.test.ts` must prove numeric and numeric-string coordinates project to deterministic map percentages.
- Unit: `src/ui/views/bases/BasesMapView.test.ts` must prove missing, non-numeric, and out-of-range coordinates are excluded from pins.
- Browser: `bun run verify:bases-map-browser` must open a `.base` file with `view: map`, `mapLatitude`, and `mapLongitude`, verify rows with valid coordinates appear as clickable pins, and verify out-of-range coordinates do not render pins.
- Browser: `bun run verify:bases-map-browser` must Ctrl-click and normal-click map pins and verify they open the backing note in the expected current/new tab target.
- Browser: `bun run verify:bases-map-browser` must use a map base with no valid coordinates and verify the empty state explains the missing latitude/longitude values.
- Browser: `bun run verify:bases-map-browser` must group a map base and verify each group renders its own coordinate plane with matching pin counts.

## Current 24.20 Screen-Reader Announcement Severe Tests

For tab, modal, and notice screen-reader announcements, the minimum severe test set is:

- Unit: `src/core/a11y/announcer.test.ts` must prove non-empty announcements update the live-region snapshot and notify subscribers.
- Unit: `src/core/notices/notice.test.ts` must prove showing a notice emits an announcement containing the notice kind and message.
- Unit: `src/core/notices/notice.test.ts` must prove notice list snapshots change identity on show and dismiss so UI subscribers cannot miss updates.
- Integration: `src/ui/A11yAnnouncer.test.tsx` must prove the app-level announcer renders a polite, atomic live region with the latest message.
- Integration: `src/ui/A11yAnnouncer.test.tsx` must prove workspace active-tab changes announce `Active tab: <title>` after initial render.
- Integration: `src/ui/overlay/Modal.test.tsx` must prove opening a titled modal announces `Opened dialog: <title>`.
- Integration: `src/ui/overlay/Modal.test.tsx` must prove title-less modals expose an accessible label and announce that label.
- Integration: `src/ui/overlay/NoticeContainer.test.tsx` must prove shown notice content still renders in an alert surface while the announcer receives the spoken message.
- Browser: `bun run verify:a11y-announcements-browser` must switch active tabs and verify the live region announces the active tab title exactly once per change.
- Browser: `bun run verify:a11y-announcements-browser` must open a labeled modal and verify the live region announces a meaningful dialog label while focus lands on a labeled modal control.
- Browser: `bun run verify:a11y-announcements-browser` must trigger success, info, warning, and error notices and verify the notice kind and message are announced without stealing focus.

## Current 24.20 Theme Contrast Severe Tests

For body-text contrast, the minimum severe test set is:

- Unit: `src/styles/contrast.test.ts` must read the real `src/styles/tokens.css` source and fail if it cannot find `--text-normal`.
- Unit: `src/styles/contrast.test.ts` must read the real `src/styles/high-contrast.css` source and fail if the high-contrast selectors are absent.
- Unit: `src/styles/contrast.test.ts` must resolve `--text-normal` and `--background-primary` through CSS variable chains instead of duplicating final colors in the test.
- Unit: `src/styles/contrast.test.ts` must assert light theme body text contrast is at least 4.5:1.
- Unit: `src/styles/contrast.test.ts` must assert dark theme body text contrast is at least 4.5:1.
- Unit: `src/styles/contrast.test.ts` must assert light high-contrast body text contrast is at least 4.5:1.
- Unit: `src/styles/contrast.test.ts` must assert dark high-contrast body text contrast is at least 4.5:1.
- Browser: `bun run verify:theme-contrast-browser` must open the app stylesheet bundle in light and dark themes, inspect `body` computed `color` and `background-color`, and verify the measured contrast is at least 4.5:1.
- Browser: `bun run verify:theme-contrast-browser` must enable high-contrast mode in light and dark themes and verify the body text/background contrast remains at least 4.5:1.

## Current 24.18 External File Drag-and-Drop Severe Tests

For external OS file drag-and-drop behavior, the minimum severe test set is:

- Unit: `src/core/dnd/external-files.test.ts` must prove Ctrl means file-URL drop on Windows/Linux.
- Unit: `src/core/dnd/external-files.test.ts` must prove Option means file-URL drop on macOS and Ctrl alone does not.
- Unit: `src/core/dnd/external-files.test.ts` must prove POSIX, Windows drive-letter, and UNC paths encode to valid `file:///` URLs with spaces escaped.
- Unit: `src/core/dnd/external-files.test.ts` must prove Markdown file URL links are produced only when the host exposes an external path.
- Unit: `src/core/dnd/external-files.test.ts` must prove illegal filename characters and empty names are sanitized before vault import.
- Unit: `src/core/dnd/external-files.test.ts` must prove duplicate dropped filenames get deterministic `-N` suffixes instead of overwriting existing vault files.
- Unit: `src/core/dnd/external-files.test.ts` must prove imported file bytes are written to the selected vault folder and parent folders are created.
- Browser: `bun run verify:external-dnd-browser` must drag an OS-like file into the editor with no modifier and verify supported media/PDF files are copied into the attachments folder and inserted as embeds.
- Browser: `bun run verify:external-dnd-browser` must drag an OS-like file into the editor with Ctrl on Windows/Linux or Option on macOS and verify a `file:///` Markdown link is inserted instead of an imported attachment.
- Browser: `bun run verify:external-dnd-browser` must drag an OS-like file into the editor with the modifier in a host that does not expose file paths and verify Granite prevents browser navigation and shows a warning.
- Browser: `bun run verify:external-dnd-browser` must drag one or more OS-like files onto a File Explorer folder and verify copied files appear in that folder without overwriting collisions.
- Browser: `bun run verify:external-dnd-browser` must drag one or more OS-like files onto the File Explorer empty/root area and verify copied files appear at the vault root.

## Current 24.1 External Edit Detection Severe Tests

For external edit detection, the minimum severe test set is:

- Unit: `src/core/markdown/external-edit.test.ts` must prove the editor sync debounce is no greater than the 500 ms acceptance budget.
- Unit: `src/core/markdown/external-edit.test.ts` must prove the budget constant is exactly 500 ms so the test tracks the product requirement.
- Unit: `src/core/markdown/external-edit.test.ts` must prove create, modify, delete, and rename watcher events touching the open path are recognized.
- Unit: `src/core/markdown/external-edit.test.ts` must prove watcher events for unrelated paths are ignored.
- Unit: `src/core/markdown/external-edit.test.ts` must prove external file content is applied only when the editor has no unsaved local edits.
- Browser: `bun run verify:external-edit-browser` must open a note, write an external file edit through the active browser FileSystem, and verify the open Granite editor updates within 500 ms.
- Browser: `bun run verify:external-edit-browser` must make unsaved local edits in Granite, then save a different external edit to the same file, and verify Granite does not overwrite the unsaved local buffer.
- Browser: `bun run verify:external-edit-browser` must save from Granite and verify the resulting filesystem watcher event does not loop or re-dirty the editor.

## Current 24.22 File Recovery Restore Severe Tests

For file-recovery restore behavior, the minimum severe test set is:

- Unit: `src/core/plugins-core/file-recovery.test.ts` must prove snapshots list newest-first and are scoped to the requested file path.
- Unit: `src/core/plugins-core/file-recovery.test.ts` must prove restoring a selected snapshot writes through the active vault `FileSystem`.
- Unit: `src/core/plugins-core/file-recovery.test.ts` must prove clearing recovery storage removes snapshots for every file.
- Browser: `bun run verify:file-recovery-browser` must run File recovery for an active markdown note and verify a modal opens with the current filename, snapshot list, Show changes toggle, preview textarea, Copy, Restore, and Clear controls.
- Browser: `bun run verify:file-recovery-browser` must create divergent current content and snapshot content, toggle Show changes on/off, and verify the line-level diff compares against the current file instead of an empty baseline.
- Browser: `bun run verify:file-recovery-browser` must restore a snapshot, reopen the note from disk, and verify the file content exactly matches the selected snapshot.
- Browser: `bun run verify:file-recovery-browser` must copy a snapshot and verify clipboard contents match the selected snapshot without restoring.
- Browser: `bun run verify:file-recovery-browser` must clear snapshots, close and reopen the modal, and verify no old snapshots are still listed.

## Current 24.22 Workspace Kill-and-Restart Severe Tests

For workspace crash-restart persistence, the minimum severe test set is:

- Unit: `src/core/fs/orphan-temp.test.ts` must prove startup crash-safety scanning detects both legacy `.tmp~` and current `.granite-tmp~` atomic-write leftovers without matching ordinary note names.
- Unit: `src/core/fs/orphan-temp.test.ts` must prove orphan temp scanning goes through the active `FileSystem` service rather than inspecting a browser-global handle directly.
- Unit: `src/core/workspace/persist.test.ts` must prove a pending debounced workspace snapshot is flushed when persistence is unbound during fast close.
- Unit: `src/core/workspace/persist.test.ts` must prove a pending debounced workspace snapshot is flushed by `beforeunload`.
- Unit: `src/core/workspace/persist.test.ts` must prove 100 consecutive simulated kill-and-restart cycles restore the latest opened note without losing the active snapshot.
- Unit: `src/core/workspace/persist.test.ts` must prove persisted multi-column and stacked layouts still round-trip through restore.
- Unit: `src/core/workspace/persist.test.ts` must prove a transient single-empty-leaf state does not create a workspace snapshot.
- Browser: `bun run verify:workspace-restart-browser` must rapidly open a note, close the app window before the 500 ms debounce elapses, restart, and verify the note and layout are restored.
- Browser: `bun run verify:workspace-restart-browser` must repeat restart testing after creating split panes and stacked tab groups, and verify the restored workspace matches the pre-close layout.

## Current 24.17 Community Plugin Browser Severe Tests

For community plugin browse, install, and enable behavior, the minimum severe test set is:

- Unit: `src/core/plugins/community-registry.test.ts` must parse the official Obsidian `community-plugins.json` shape with `id`, `name`, `author`, `description`, and `repo`.
- Unit: `src/core/plugins/community-registry.test.ts` must reject unsafe plugin ids and non-`owner/repo` registry repos before URLs are derived.
- Unit: `src/core/plugins/community-registry.test.ts` must search by name, id, author, description, and repo so popular plugins remain discoverable by multiple user intents.
- Unit: `src/core/plugins/community-registry.test.ts` must derive the repo manifest URL and version-tagged GitHub release asset URLs from registry entries.
- Unit: `src/core/plugins/community-registry.test.ts` must prove registry fetches are credential-free and fail loudly on non-OK registry responses.
- Browser: `bun run verify:community-plugin-browser` must open Settings → Plugins → Install community plugin and verify a mocked registry loads before any manual URL is pasted.
- Browser: `bun run verify:community-plugin-browser` must search for Advanced Tables, Git, and Calendar, select each result, and verify the preview shows the expected id, version, author, description, and plugin code size.
- Browser: `bun run verify:community-plugin-browser` must install Advanced Tables, Git, and Calendar into a disposable OPFS vault and verify each writes `.granite/plugins/<id>/manifest.json`, `main.js`, and optional `styles.css` without enabling automatically.
- Browser: `bun run verify:community-plugin-browser` must enable each installed plugin from Settings → Plugins and verify it can be disabled again without stale commands, status items, or settings tabs.
- Browser: `bun run verify:community-plugin-browser` must run Check for updates after registry install and verify the persisted `manifestUrl` is used instead of reporting that no plugins have remote manifest URLs configured.

## Current 24.19 Performance Budget Severe Tests

For large-vault performance budgets, the minimum severe test set is:

- Unit: `src/core/search/fuzzy.test.ts` must build a 10k-item Quick Switcher fixture, warm a reusable fuzzy index, and assert a keystroke query completes in under 16 ms.
- Unit: `src/core/search/query.test.ts` must scan 10k synthetic note bodies with a regex query and assert matching completes in under 500 ms.
- Unit: `src/core/metadata/cache.test.ts` must index a 10k Markdown-file fixture in under 3,000 ms, prove cold-start metadata indexing uses `listAll()` file metadata directly, and prove it does not re-stat every listed Markdown file before reading it.
- Browser: `bun run verify:startup-browser` must launch Chromium against a Vite-served 10k-note synthetic vault fixture, bind the real Effect `FileSystem` layer, run `metadataCache.indexVault()`, prove 10,000 reads, 0 stats, 20,000 switcher entries, and assert browser-measured elapsed time remains under 3,000 ms.
- Unit: `src/core/fs/handle-adapter.test.ts` must exercise the real atomic `writeText()` temp-file copy path through a File System Access handle mock and assert content round-trips in under 50 ms.
- Browser: `bun run verify:save-roundtrip-browser` must launch Chromium, open OPFS, write a 120 KB Markdown payload through `handleAdapter.writeText()`, read it back through the adapter, prove byte-for-byte preservation, and assert the browser write round-trip remains under 50 ms.
- Browser: `bun run verify:search-performance-browser` must launch Chromium against a Vite-served fixture, build 20,000 Quick Switcher candidates with the shared fuzzy index used by `Prompt`, assert each progressive query update stays under 16 ms, then scan 10,000 Markdown contexts with `parseQuery()` and `fileMatchesQuery()` for a regex plus property query and assert the scan stays under 500 ms with the expected match count.
- Unit/integration: global search must scan large vaults in chunks big enough to avoid hundreds of state updates while still yielding progressive results.
- Unit/integration: `src/core/graph/pan.test.ts` must prove 10k graph pan transform calculations stay inside a single-frame budget, preserve scale without cloning unrelated viewport state, and that `GraphView` uses the shared transform helper for SVG viewport updates.
- Browser: `bun run verify:graph-pan-browser` must launch Chromium against a Vite-served 10k-node SVG graph fixture, pan continuously for 10 seconds using the same shared viewport transform helpers as `GraphView`, and assert measured frame rate remains at or above 30 fps with frame telemetry reported.
- Unit: `src/core/perf/startup.test.ts` must prove startup timing collection reads navigation timing, first paint, first contentful paint, and current elapsed time from the browser Performance API.
- Unit: `src/core/perf/startup.test.ts` must prove unavailable startup timing entries are reported explicitly instead of being hidden or coerced to zero.
- Unit: `src/core/perf/startup.test.ts` must prove the slow-startup diagnostic is silent below budget, silent when disabled, and sticky-warning visible when enabled startup timing exceeds the 3,000 ms budget.
- Unit: `src/ui/prompts/settings-filter.test.ts` must prove the General settings section is discoverable by startup/profiling search terms.
- Browser: `bun run verify:search-performance-browser` must build the shared Quick Switcher fuzzy index with at least 20,000 candidates, type at least five progressive queries, and verify each keystroke update stays under 16 ms.
- Browser: `bun run verify:startup-browser` plus `src/core/perf/startup.test.ts` must verify startup timing collection and the sticky startup report path, including elapsed, DOMContentLoaded/load, and paint timings where the host exposes them.
- Browser: `bun run verify:search-performance-browser` must run a full-vault regex/property search over 10k note contexts and verify time to complete is under 500 ms with the expected match count.
- Browser: `bun run verify:graph-pan-browser` must open a 10k-node graph fixture, pan continuously for 10 seconds, and verify measured frame rate remains at or above 30 fps.
- Browser: `bun run verify:startup-browser` must cold-open a 10k-note synthetic vault fixture from a fresh browser profile and verify the app reaches an interactive workspace in under 3 seconds.
- Browser: `bun run verify:save-roundtrip-browser` must save a modified 120 KB Markdown payload and verify disk write round-trip completes in under 50 ms without content loss.

## Current Cross-Cutting Error Boundary Severe Tests

For Sentry-style error boundary and Effect error-channel integration, the minimum severe test set is:

- Unit: `src/core/errors/reporter.test.ts` must prove strings, tagged objects, nulls, and native `Error` objects normalize into useful `Error` instances.
- Unit: `src/core/errors/reporter.test.ts` must prove captured reports are stored as the latest report and published to subscribers with the correct source.
- Unit: `src/core/effect/runtime.test.ts` must prove `runFork()` reports fire-and-forget Effect error-channel failures through the shared reporter.
- Integration: `src/ui/overlay/ErrorBoundary.test.tsx` must prove an async/effect report renders the full-screen boundary alert with the report source and message.
- Browser: `bun run verify:error-boundary-browser` must trigger a React render error and verify the boundary shows the real render-error message plus component stack and can reset or reload.
- Browser: `bun run verify:error-boundary-browser` must trigger an unhandled Promise rejection and verify the boundary catches it without leaving the app silently broken.
- Browser: `bun run verify:error-boundary-browser` must trigger a failed fire-and-forget Effect and verify it reaches the same boundary channel instead of disappearing into console-only logging.

## Current Cross-Cutting Debug-Info Severe Tests

For bug-report support diagnostics, the minimum severe test set is:

- Unit: `src/core/plugins-core/debug-info.test.ts` must prove the debug-info collector reports app version, platform/user-agent surface, vault root, total file count, Markdown file count, total byte size, workspace group/leaf counts, command count, plugin state, and metadata tag/property counts.
- Unit: `src/core/plugins-core/debug-info.test.ts` must prove the formatted support dump includes stable localized labels and plugin enablement markers.
- Integration: `src/core/plugins-core/debug-info.test.ts` must prove the `granite:show-debug-info` command writes the same dump to the clipboard when available and shows a sticky notice.
- Browser: `bun run verify:debug-info-browser` must run "Show debug info" from the command palette in an opened vault, verify the sticky notice is readable, verify clipboard contents match the visible dump, and verify no note contents or secrets are included.

## Current 24.20 Icon-Button Accessibility Severe Tests

For icon-only controls using Granite's established `.clickable-icon` pattern, the minimum severe test set is:

- Unit: `src/core/a11y/icon-buttons.test.ts` must scan every UI source file and fail if a `.clickable-icon` occurrence lacks `aria-label`, `ariaLabel`, or an imperative `setAttribute("aria-label", ...)`.
- Unit: `src/core/a11y/icon-buttons.test.ts` must fail if a literal `title` or `data-tooltip` on a clickable icon differs from its literal `aria-label`.
- Unit: `src/core/a11y/icon-buttons.test.ts` must fail if a `ClickableIcon` callsite omits the required `ariaLabel` prop.
- Unit: `src/core/a11y/icon-buttons.test.ts` must read `src/styles/buttons.css` and fail if `.clickable-icon:focus-visible` stops using `--background-modifier-border-focus` for a visible keyboard focus ring.
- Unit: `src/core/a11y/icon-buttons.test.ts` must read `src/styles/tree-item.css` and fail if `.tree-item-self.is-clickable:focus-visible` or `.tree-item-self.mod-collapsible:focus-visible` stops using `--background-modifier-border-focus` for visible keyboard focus on custom tree rows.
- Unit: `src/core/a11y/icon-buttons.test.ts` must read `src/styles/shell.css` and fail if `.status-bar-item.mod-clickable:focus-visible` stops using `--background-modifier-border-focus` for visible keyboard focus on clickable status-bar items.
- Unit: `src/core/a11y/icon-buttons.test.ts` must read `src/styles/view-graph.css` and fail if `.graph-node-interactive:focus-visible circle` stops using `--background-modifier-border-focus` for visible keyboard focus on graph nodes.
- Unit: `src/core/a11y/icon-buttons.test.ts` must read `src/styles/view-bases.css` and fail if `.bases-table-row:focus-visible` stops using `--background-modifier-border-focus` for visible keyboard focus on activatable Bases rows.
- Script: `bun run audit:a11y` must bundle the icon-button audit with live-region, modal-announcement, notice-announcement, and contrast tests.
- Browser: `bun run verify:icon-a11y-browser` must hover representative real titlebar, ribbon, workspace tab, vault-picker, file-explorer toolbar, and canvas toolbar icon controls and verify the tooltip text matches the screen-reader name.
- Browser: `bun run verify:icon-a11y-browser` must reach those same enabled controls with keyboard Tab traversal and verify each receives a visible `:focus-visible` focus ring and exposes a useful accessible name.
- Known limitation: this icon-focused verifier does not by itself prove the broader keyboard-only acceptance item; that coverage lives in `bun run verify:keyboard-browser` and `bun run verify:keyboard-populated-browser`.

## Current 24.20 Keyboard-Only Navigation Severe Tests

For keyboard-only workspace navigation, the minimum severe test set is:

- Unit/integration: `src/ui/workspace/TabStrip.test.tsx` must prove a focused ARIA tab can move within a horizontal tablist with ArrowLeft/ArrowRight and jump with Home/End.
- Unit/integration: `src/ui/workspace/TabStrip.test.tsx` must prove stacked tab groups use ArrowUp/ArrowDown instead of horizontal-only navigation.
- Unit/integration: `src/ui/overlay/Menu.test.tsx` must prove shared menu items use roving focus with ArrowUp/ArrowDown wrapping plus Home/End boundary jumps.
- Unit/integration: `src/ui/overlay/Menu.test.tsx` must prove a focused shared menu item activates with Space as well as Enter.
- Unit/integration: `src/ui/views/sidebar/BookmarksView.test.tsx` must prove the Bookmarks add menu uses roving focus with ArrowUp/ArrowDown wrapping plus Home/End boundary jumps.
- Unit/integration: `src/ui/views/sidebar/BookmarksView.test.tsx` must prove Escape closes the Bookmarks add menu and restores focus to the trigger button.
- Browser: `bun run verify:keyboard-browser` must launch Chromium against the app, tab through the shell to produce a focus-order trace, prove titlebar/ribbon/sidebar controls are keyboard reachable by accessible name, open Vault Picker, Help, Settings, and Command Palette with keyboard focus plus Enter, keep focus trapped while tabbing inside modal dialogs, move Command Palette's active descendant with ArrowDown, and close each surface with Escape.
- Browser: `bun run verify:keyboard-populated-browser` must launch Chromium against a populated fixture vault and use keyboard-only interaction for workspace tabs, File Explorer file rows, the Markdown editor, shared menus, graph controls, canvas controls, notices, and file-backed views; every sampled active control must be reachable, visibly focused, and operable with Enter/Space or the documented arrow keys.

## Current 24.20 Lighthouse Accessibility Severe Tests

For automated page-level accessibility regression coverage, the minimum severe test set is:

- Browser/audit: `bun run audit:lighthouse-a11y` must launch a local Vite app, run Lighthouse's accessibility category, and assert an accessibility score of `1`.
- Browser/audit: `bun run audit:lighthouse-a11y` must fail if the Lighthouse report contains any failed accessibility audit; the 2026-05-13 root-cause failures were `aria-required-parent` for workspace tabs missing a `tablist` and `landmark-one-main` for the workspace shell missing a main landmark.
- Unit: `bun run audit:a11y` must continue to pass the focused icon-button, live-region, modal-announcement, notice-announcement, and contrast checks before the Lighthouse audit is considered valid.
- Unit: `src/core/i18n/externalization.test.ts` must fail if the workspace tablist label stops routing through `workspace.tab.list`.
- Known limitation: Lighthouse does not replace the full keyboard-only audit; it closes the automated page-level Lighthouse item only.

## Current 24.23 Obsidian Compatibility Severe Tests

For opening and round-tripping existing Obsidian vault data, the minimum severe test set is:

- Unit: `src/core/compat/obsidian-roundtrip.test.ts` must index an Obsidian-style fixture containing `.obsidian/` config without writing to any fixture file.
- Unit: `src/core/compat/obsidian-roundtrip.test.ts` must prove Markdown aliases, YAML tags, CSS classes, headings, wikilinks, embeds, block IDs, and footnotes are extracted from the fixture with source-line fidelity.
- Unit: `src/core/compat/obsidian-roundtrip.test.ts` must prove aliases from fixture notes appear in switcher entries without requiring a Granite migration.
- Unit: `src/core/compat/obsidian-roundtrip.test.ts` must parse and serialize the fixture `.canvas` file to an equivalent Canvas structure preserving file nodes, group nodes, arrow edges, and labels.
- Unit: `src/core/compat/obsidian-roundtrip.test.ts` must parse and serialize the fixture `.base` file to an equivalent Base config preserving view type, grouping, summaries, and formulas.
- Unit: `src/core/compat/obsidian-roundtrip.test.ts` must index a generated 200-note Obsidian-style vault with `.obsidian/` config, aliases, YAML/body tags, wikilinks, embeds, callouts, block IDs, canvas, base, and asset files while proving no source or `.obsidian/app.json` writes occur.
- Browser: `bun run verify:obsidian-vault-browser` must launch Chromium against a Vite-served 200-note Obsidian-style vault fixture, bind the real Effect `FileSystem` layer, index the vault with zero writes, render every note through Granite's reading renderer, parse and serialize the fixture canvas/base semantically, save every note back byte-for-byte, and prove no `.obsidian/` file is written.
- Browser: `bun run verify:obsidian-vault-browser` must apply an intentional note edit, re-index it through the Obsidian-compatible metadata parser, and verify frontmatter, aliases, tags, and wikilink/backlink semantics remain intact beyond the intentional body edit.
- Browser: `bun run verify:obsidian-vault-browser` must save-cycle fixture `.canvas` and `.base` files and verify their parsed structures preserve canvas nodes/edges/labels/arrows and base view type/grouping/summaries/formulas without writing `.obsidian/`.
- Known limitation: this does not prove cross-app visual parity for community themes; that remains covered by the dedicated theme cross-render severe tests below.

## Current Public Docs Severe Tests

For the public vault-format and plugin API docs, the minimum severe test set is:

- Unit: `src/core/docs/public-docs.test.ts` must prove `docs/index.html` links to the vault format, plugin API, and contributor guide.
- Unit: `src/core/docs/public-docs.test.ts` must prove `docs/vault-format.md` mentions the Granite config folder, Obsidian compatibility folder, accepted native file extensions, JSON Canvas, plugin data, and atomic writes.
- Unit: `src/core/docs/public-docs.test.ts` must parse `src/core/plugins/types.ts` and fail if `docs/plugin-api.md` omits any top-level `PluginApi` member.
- Unit: `src/core/docs/public-docs.test.ts` must prove `package.json` wires both `docs:check` and `docs:verify-browser` so the drift and browser reachability checks remain discoverable.
- Script: `bun run docs:check` must run the public-docs drift tests directly.
- Browser: `bun run docs:verify-browser` must serve `docs/index.html` in Chromium, verify the docs stylesheet applies, follow the vault-format, plugin-API, and contributor-guide Markdown links, and assert each linked page loads with readable required text.
- Unit/process: `docs/contributor-guide.md` must tell contributors to run `bun run docs:check` before and after public `PluginApi` changes so the existing PluginApi-member drift test is observed as failing before docs updates and passing afterward.

## Current 24.23 Community Theme Severe Tests

For community theme compatibility, the minimum severe test set is:

- Unit: `src/core/themes/loader.test.ts` must prove `.obsidian/themes/<name>/theme.css` themes are discovered.
- Unit: `src/core/themes/loader.test.ts` must prove Granite's `.granite/themes/*.css` layout remains discoverable.
- Unit: `src/core/themes/loader.test.ts` must prove snippet CSS and unrelated CSS files are not listed as themes.
- Unit: `src/core/themes/loader.test.ts` must prove applying an Obsidian-layout theme injects a `style[data-granite-theme]` element with the source CSS.
- Unit: `src/core/themes/loader.test.ts` must prove applying an Obsidian-layout theme writes only `.granite/active-theme.json` and does not rewrite the source `.obsidian/themes/<name>/theme.css`.
- Unit: `src/core/themes/loader.test.ts` must prove editing the active Obsidian-layout `theme.css` through the filesystem watcher refreshes the injected stylesheet and changes the CSS custom property visible to the document.
- Browser: `bun run verify:community-theme-browser` must launch Chromium against a Vite-served fixture, discover two `.obsidian/themes/<name>/theme.css` themes through the real loader, switch each through light and dark mode, render workspace chrome, Markdown, settings, graph, canvas, and bases surfaces, capture four screenshots, prove the screenshots are nonblank and visually distinct by hash, and assert computed theme tokens produce visible text/background values.
- Browser: `bun run verify:community-theme-browser` must edit the active community theme CSS externally and verify Granite reloads it without a restart.
- Known limitation: fixture screenshots prove cross-render mechanics and visual token propagation, not a curated marketplace-wide pixel review of every third-party theme.

## Current Renderer CSS Module Severe Tests

For renderer-spec CSS module extraction, the minimum severe test set is:

- Unit: `src/styles/renderer-modules.test.ts` must prove `src/styles/index.css` imports the dedicated `os-modifiers.css`, `rtl.css`, `typography.css`, `animations.css`, `loading.css`, `progress.css`, `buttons.css`, `inputs.css`, `suggestion-and-prompt.css`, `multi-select.css`, `checkbox.css`, `slider.css`, `flair-and-pill.css`, `notice.css`, `tree-item.css`, `scrollbars.css`, `modal.css`, `drag.css`, `splash.css`, `empty-state.css`, `card.css`, `view-release-notes.css`, `view-history-sync.css`, `settings-community.css`, `view-graph.css`, `view-pdf.css`, and `view-bases.css` renderer modules.
- Unit: `src/styles/renderer-modules.test.ts` must prove `animations.css` defines all renderer animation keyframes: `node-inserted`, `blink`, `sk-cubeGridScaleDelay`, `multi-select-highlight`, `increase`, `decrease`, `pop-down`, `pop-right`, `rotation`, `hmd-file-uploading-ani`, `progress-bar`, `spin`, and `slideIn`.
- Unit: `src/styles/renderer-modules.test.ts` must prove each extracted module starts with the matching `/* SPEC: specs/renderer/<name>.md */` comment and contains representative required selectors from the spec.
- Unit: `src/styles/renderer-modules.test.ts` must fail if the extracted button/input rule blocks return to broad shell or overlay styles instead of their dedicated modules.
- Unit: `src/styles/renderer-modules.test.ts` must fail if toast container/base styles return to `overlays.css` instead of `notice.css`.
- Unit: `src/styles/renderer-modules.test.ts` must fail if tree row, rename, nested-child, icon, or drop-indicator styles return to `views.css` instead of `tree-item.css`.
- Unit: `src/styles/renderer-modules.test.ts` must fail if multi-select pill chrome returns to `flair-and-pill.css` instead of `multi-select.css`.
- Unit: `src/styles/renderer-modules.test.ts` must prove `scrollbars.css` keeps styled scrollbar, Android, screenshot, print, and Firefox fallback rules wired to the renderer spec.
- Unit: `src/styles/renderer-modules.test.ts` must fail if modal container/base, scrollable variants, confirmation state, lightbox, file-browser, rename textarea, or message-box styles return to broad overlay/view styles instead of `modal.css`.
- Unit: `src/styles/renderer-modules.test.ts` must fail if drag body state, drag ghosts, reorder ghosts, hidden-source marker, drop indicator, workspace drop overlay, or fake-target overlay styles return to broad base/shell/tree styles instead of `drag.css`.
- Unit: `src/styles/renderer-modules.test.ts` must prove `splash.css` keeps starter-screen, splash brand/version, and help-options container styles wired to `specs/renderer/splash.md`.
- Unit: `src/styles/renderer-modules.test.ts` must prove `empty-state.css` keeps full-leaf empty-state, side-dock empty-state, action-link, keyboard-hint, and phone feedback-banner styles wired to `specs/renderer/empty-state.md`.
- Unit: `src/styles/renderer-modules.test.ts` must prove `card.css` keeps selectable card containers, hover/selected states, title, description, and horizontal-list rules wired to `specs/renderer/card.md`.
- Unit: `src/styles/renderer-modules.test.ts` must fail if macOS control-token overrides return to `tokens.css` instead of `os-modifiers.css`.
- Unit: `src/styles/renderer-modules.test.ts` must prove `os-modifiers.css` keeps macOS/Windows/Linux titlebar variants, Apple more-menu icon rotation, frameless titlebar behavior, and frame-spacer rules wired to `specs/renderer/os-modifiers.md`.
- Unit: `src/styles/renderer-modules.test.ts` must fail if body RTL direction tokens return to `tokens.css` or markdown RTL direction rules return to `markdown.css` instead of `rtl.css`.
- Unit: `src/styles/renderer-modules.test.ts` must prove `rtl.css` keeps RTL chrome direction, mobile nav padding, icon mirroring, Hebrew help-icon exception, bidi plaintext user-content selectors, callout `:has(:dir(rtl))` handling, and markdown/source direction rules wired to `specs/renderer/rtl.md`.
- Unit: `src/styles/renderer-modules.test.ts` must fail if prompt or suggestion-item chrome returns to `overlays.css` instead of `suggestion-and-prompt.css`.
- Unit: `src/styles/renderer-modules.test.ts` must prove `suggestion-and-prompt.css` keeps prompt input/container/instruction chrome, anchored suggestion container and mobile bounds, CodeMirror autocomplete chrome, suggestion backdrop, selected/dragged/toggle/complex states, prefix/highlight/hotkey/action/flair rules, and secret-key suggestions wired to `specs/renderer/suggestion-and-prompt.md`.
- Unit: `src/styles/renderer-modules.test.ts` must fail if body typography or garbled privacy text returns to `base.css`, reading-mode headings/emphasis/links/inline-title rules return to `markdown.css`, or source-mode heading/emphasis markers return to `cm-livepreview.css` instead of `typography.css`.
- Unit: `src/styles/renderer-modules.test.ts` must prove `typography.css` keeps body and markdown text baselines, reading/source heading scale, formatting markers, paragraph spacing, bold/italic/strike/link rules, keyboard-label styling, inline title behavior, and Flow Circular garbled mode wired to `specs/renderer/typography.md`.
- Unit: `src/styles/renderer-modules.test.ts` must prove `view-release-notes.css` keeps release-notes wrapper padding, readable-line width, markdown-preview overflow, changelog label pseudo-element, and success/failed/highlighted label variants wired to `specs/renderer/view-release-notes.md`.
- Unit: `src/styles/renderer-modules.test.ts` must prove `view-history-sync.css` keeps File Recovery modal/list/text-preview styles, sync-history list/sidebar/avatar/version-group/content styles, sync status spinner reduced-motion handling, and sync sharing/exclude row styles wired to `specs/renderer/view-history-sync.md`.
- Unit: `src/styles/renderer-modules.test.ts` must prove `settings-community.css` keeps Community Plugins/Themes sidebar rows, search-result states, community cards, screenshots, details panes, action buttons, README media constraints, and selected/update states wired to both `specs/renderer/settings-community-plugins.md` and `specs/renderer/settings-community-themes.md`.
- Unit: `src/styles/renderer-modules.test.ts` must prove `view-graph.css` keeps graph color-token hooks, graph canvas/empty/stat overlays, controls panel, collapsed-control state, control sections, group color rows, swatches, and slider value chrome wired to `specs/renderer/view-graph.md` and `specs/renderer/design-tokens.md`.
- Unit: `src/styles/renderer-modules.test.ts` must prove `view-pdf.css` keeps PDF container/theme hooks, markdown PDF embed frame, viewer/page chrome, toolbar, sidebar, find bar, mobile sidebar-resizer hiding, password dialog, presentation mode hiding, and PDF token links wired to `specs/renderer/view-pdf.md` and `specs/renderer/design-tokens.md`.
- Unit: `src/styles/renderer-modules.test.ts` must prove `view-bases.css` keeps Bases shell/header/toolbar/menu hooks, table/list/cards/map views, group headings, summaries, drag-over state, embedded/error/search row hooks, map pins, and Bases token links wired to `specs/renderer/view-bases.md` and `specs/renderer/design-tokens.md`.
- Unit: `src/styles/renderer-modules.test.ts` must fail if thin loading bars, flashing highlights, or button loading spinners return to `base.css` or `buttons.css` instead of `loading.css`.
- Unit: `src/styles/renderer-modules.test.ts` must prove `progress.css` keeps full-screen progress overlay, context, button stack, indicator, line, and animated subline styles wired to `specs/renderer/progress-bar.md`.
- Browser: `bun run verify:renderer-visual-browser` must serve a renderer-state fixture in Chromium, verify representative visible computed styles for typography, inline title, source tokens, garbled text, default/CTA/warning/destructive/loading/icon buttons, loading bars, spinner/cube/progress states, text/search/number/date inputs, textareas, checkboxes, radio buttons, sliders, flairs, multi-select pills, tree rows, drag/drop ghosts/overlays, prompt/suggestion chrome, modal, notices, cards, empty states, macOS/Windows/Linux titlebars, release notes, File Recovery, sync-history, Community Plugins/Themes, graph, PDF, and Bases table/cards/map states.
- Browser: `bun run verify:renderer-visual-browser` must capture the renderer-state fixture in light and dark themes, fail if either screenshot is blank/suspiciously small, and fail if the light/dark screenshots or body backgrounds are not visually distinct.
- Known limitation: this browser fixture plus the renderer module unit audit verifies representative state rendering and wiring, not curated pixel parity for every value in every renderer state table.

## Current 24.21 UI String Externalization Severe Tests

For UI string externalization, the minimum severe test set is:

- Browser: `bun run verify:i18n-browser` must launch Chromium against the real app, switch the runtime locale to Hebrew through the real i18n module, prove `document.dir` and RTL body classes update, verify localized visible/accessible labels on the welcome screen and ribbon, open Settings through the localized settings button, verify localized Settings labels, open Command Palette through the localized ribbon button, and verify its localized placeholder. This must also prove the RTL status bar does not intercept the localized bottom-ribbon controls.
- Unit: `src/core/i18n/index.test.ts` must prove English lookup, fallback, parameter substitution, subscriber notification, locale round-trip, and Hebrew RTL demo strings.
- Unit: `src/core/i18n/externalization.test.ts` must fail if Search view's formerly hard-coded English placeholder, controls, sort options, empty states, or status messages reappear as visible JSX literals.
- Unit: `src/core/i18n/externalization.test.ts` must fail if Tags view's formerly hard-coded empty state, nested-tags toggle, expand/collapse labels, or context-menu labels reappear as visible JSX literals.
- Unit: `src/core/i18n/externalization.test.ts` must fail if Backlinks or Outgoing Links pane empty states, line labels, unlinked-mention labels, scanning state, or match tooltip return as hard-coded English JSX literals.
- Unit: `src/core/i18n/externalization.test.ts` must fail if Recents, Footnotes, or Outline empty states, action labels, placeholders, missing-footnote labels, or footnote reference tooltip text return as hard-coded English JSX literals.
- Unit: `src/core/i18n/externalization.test.ts` must fail if sidebar tab registry labels, sidebar group action labels, Local Graph empty/count/node-open labels, or the unavailable-sidebar fallback return as hard-coded English JSX literals.
- Unit: `src/core/i18n/externalization.test.ts` must fail if Properties or All Properties pane prompts, empty states, add/remove actions, list placeholder, fallback errors, inferred/override titles, usage tooltip, or property type labels return as hard-coded English JSX literals.
- Unit: `src/core/i18n/externalization.test.ts` must fail if Ribbon action labels, Status Bar local-only/word-count/mode labels, or Vault Profile switch/no-vault/settings labels return as hard-coded English JSX literals.
- Unit: `src/core/i18n/externalization.test.ts` must fail if File Explorer prompts, duplicate/invalid-name fallback errors, delete confirmations, delete/import/move notices, context-menu labels, sort labels, toolbar labels, or empty states return as hard-coded English JSX literals.
- Unit: `src/core/i18n/externalization.test.ts` must fail if the shared Prompt no-match state, Quick Switcher placeholder/loading/create/alias/flair/instruction labels, Command Palette placeholder/pin/instruction labels, or Template Picker placeholder/instruction labels return as hard-coded English JSX literals.
- Unit: `src/core/i18n/externalization.test.ts` must fail if Modal fallback/close/announcement labels, Vault Picker prompts/actions/tooltips, Help modal shortcut key labels/descriptions, or Bookmarks prompts/notices/menu/default-group/empty/remove labels return as hard-coded English JSX literals, including the old default-group `"Bookmarks"` constant in the Bookmarks view.
- Unit: `src/core/i18n/externalization.test.ts` must fail if Markdown wikilink autocomplete alias suggestions reintroduce the hard-coded `alias for ${targetStem}` detail instead of `markdown.autocomplete.aliasFor`.
- Unit: `src/core/i18n/index.test.ts` must prove `markdown.autocomplete.aliasFor` has Hebrew coverage with parameter substitution.
- Unit: `src/core/i18n/externalization.test.ts` must fail if the Bases fenced embed header reintroduces the old bare `<code class="bases-fence-filter">` pattern instead of a localized `reading.embed.filterSummary` label.
- Unit: `src/core/i18n/externalization.test.ts` must fail if Vault Context bootstrap, reopen-grant, reopen-failure, plugin-loader, lost-handle, permission-denied, or missing-registry notices/errors return as hard-coded English literals.
- Unit: `src/core/i18n/externalization.test.ts` must fail if File Explorer trash errors returned from `src/core/fs/trash.ts` return as hard-coded English instead of `fs.trash.error.vaultPathUnavailable` and `fs.trash.error.systemUnavailable`.
- Unit: `src/core/i18n/externalization.test.ts` must fail if File System Access adapter `FsAccessDenied.reason` strings return as hard-coded English instead of `fs.error.emptyPath` and `fs.error.directoryRenameUnsupported`.
- Unit: `src/core/i18n/date-format.test.ts` must prove Moment-style numeric tokens still format correctly while month and weekday tokens use the active locale.
- Unit: `src/core/i18n/externalization.test.ts` must fail if Daily Notes or Templates reintroduce hard-coded English month/weekday arrays instead of `formatMomentDate`.
- Unit: `src/core/i18n/externalization.test.ts` must fail if Graph View empty states, SVG label/title, stats chip, controls panel title/section labels, filter/group query placeholders, color-mode options, group controls, display sliders, force sliders, or reset action return as hard-coded English JSX literals.
- Unit: `src/core/i18n/externalization.test.ts` must fail if File Recovery load fallback errors, copy/restore/clear notices, clear confirmation, modal/list labels, filename/filter labels, loading/empty states, byte counts, show-changes toggle, or action labels return as hard-coded English JSX literals.
- Unit: `src/core/i18n/externalization.test.ts` must fail if Install Plugin manifest validation errors, fetch errors, community-registry parse/fetch errors, registry mismatch errors, success notices, modal descriptions, registry search/loading labels, manual manifest label/URL placeholder, fetch/install/cancel actions, author label, or plugin-code size summary return as hard-coded English JSX literals.
- Unit: `src/core/i18n/externalization.test.ts` must fail if Settings modal chrome, search placeholder, sidebar group labels, built-in section labels/descriptions/options, spellcheck-language labels/placeholders, attachments/excluded-files/date/time placeholders, hotkey actions, plugin actions, daily-notes/templates labels, plugin-tab render fallback, or built-in settings-filter rendered titles return as hard-coded English literals.
- Unit: `src/core/settings/spellcheck.test.ts` must prove comma/newline-separated spellcheck language tags are normalized, invalid tags ignored, the first valid tag is selected for editor language, and disabling spellcheck removes the editor `lang` attribute.
- Unit: `src/core/i18n/externalization.test.ts` must fail if Titlebar navigation labels, tab strip actions, tab context-menu actions, pinned-tab controls, tab close controls, leaf-header read/edit/split actions, workspace leaf fallback titles, native asset fallback titles, stale core `leafTitle()` fallbacks, active-tab live-region announcements, welcome empty state, or active-vault no-file hint return as hard-coded English literals.
- Unit: `src/core/i18n/externalization.test.ts` must fail if Markdown source view read errors, attachment-save fallback errors, dropped-file path warnings, save-status text, Web Viewer navigation labels, or Web Viewer URL placeholder return as hard-coded English literals.
- Unit: `src/core/i18n/externalization.test.ts` must fail if Asset View loading text returns as a hard-coded English literal instead of `asset.loading`.
- Unit: `src/core/i18n/externalization.test.ts` must fail if Reading view loading text, canvas/base embed summaries, embed open labels, circular/file-missing markdown embed messages, embedded base loading state, query block labels/status/errors, backlinks labels/counts, or properties-strip count return as hard-coded English literals.
- Unit: `src/core/i18n/externalization.test.ts` must fail if Canvas save fallback errors, text-node prompt, no-path/loading states, toolbar actions, snap-to-grid labels, color controls, delete labels, stats chip, anchor/resize tooltips, file-node fallback/open hint, or link-node label return as hard-coded English literals.
- Unit: `src/core/i18n/externalization.test.ts` must fail if Bases view no-path/loading states, default base names, filter and match summaries, scaffold exists errors, table/list/cards empty states, built-in column labels, stale schema column-label helpers, map labels/helper text, or embedded base loading/empty labels return as hard-coded English literals.
- Unit: `src/core/i18n/externalization.test.ts` must fail if Inline Title rename notices/tooltips/exists errors, Error Boundary fallback labels/actions, Hover Popover loading/missing states, or Notice dismiss labels return as hard-coded English literals.
- Unit: `src/core/i18n/externalization.test.ts` must fail if built-in Command Palette registration names or categories in `CommandsBootstrap` return as hard-coded English literals.
- Unit: `src/core/i18n/externalization.test.ts` must fail if Bases scaffold, Web Viewer, Random Note, or Random Walk core plugin command labels, prompts, success notices, empty-state notices, or fallback errors return as hard-coded English literals.
- Unit: `src/core/i18n/externalization.test.ts` must fail if Copy Link, Plugin Reload, Tour path/body/command labels, Tour success notice, or plugin update-check command labels, success notices, empty-state notices, compatibility notices, update summaries, or fallback errors return as hard-coded English literals.
- Unit: `src/core/i18n/externalization.test.ts` must fail if Format Converter command labels, no-op notices, conversion/migration/copy success notices, or fallback errors return as hard-coded English literals.
- Unit: `src/core/i18n/externalization.test.ts` must fail if Note Composer command labels, selection/no-active-note notices, new-note/merge prompts, duplicate/missing-target errors, extract/merge success notices, or fallback errors return as hard-coded English literals.
- Unit: `src/core/i18n/externalization.test.ts` must fail if Unique Note, Daily Notes, Vault Stats, or Audio Recorder command labels, stats notice rows, recording lifecycle notices, or fallback errors return as hard-coded English literals.
- Unit: `src/core/i18n/externalization.test.ts` must fail if Vault Find/Replace or Tag Rename command labels, prompts, confirmations, no-match notices, success summaries, validation errors, or fallback errors return as hard-coded English literals.
- Unit: `src/core/i18n/externalization.test.ts` must fail if Templates or Workspaces command labels, empty-state notices, save/load/delete prompts, success summaries, or load errors return as hard-coded English literals.
- Unit: `src/core/i18n/externalization.test.ts` must fail if File Recovery command labels, no-active-file/no-snapshot notices, restore picker prompt, overwrite confirmation, restore success, or fallback errors return as hard-coded English literals.
- Unit: `src/core/i18n/externalization.test.ts` must fail if Plugin Loader no-active-vault API errors, read-main notices, or load-failure notices return as hard-coded English literals.
- Unit: `src/core/i18n/externalization.test.ts` must broadly scan non-test `src/ui`, `src/core/plugins-core`, and core plugin registry/update loader sources for hard-coded visible JSX text, labels, placeholders, command names/categories, prompts, confirmations, notices, and direct `textContent` strings, allowing only literal product branding and non-translatable key/chord/code tokens.
- Browser: `bun run verify:i18n-browser` must first switch the no-vault app from English to Hebrew through the real i18n module without reloading, then verify RTL document/body state plus localized welcome, ribbon, Settings, and Command Palette labels.
- Browser: `bun run verify:i18n-browser` must then open a populated OPFS fixture, switch to Hebrew/RTL, and verify localized runtime text or accessible labels for Search, Tags, Backlinks, Recents, Footnotes, Outline, Local Graph, Properties, All Properties, File Explorer, Status Bar, Settings, Vault Picker, Quick Switcher, Command Palette, Template Picker, Canvas, Graph, Bases, modal close controls, and notice dismissal without reloading.
- Integration: `src/ui/views/sidebar/BookmarksView.test.tsx` must seed a legacy saved default group named `"Bookmarks"`, switch locale to Hebrew, and fail if the sidebar renders the English group instead of the localized `bookmarks.defaultGroup` label.
- Unit: `src/core/i18n/externalization.test.ts` must continue to scan Vault Context, plugin loader, Graph, File Recovery, Install Plugin, Settings, workspace/tab/titlebar/leaf strings, Markdown source, Web Viewer, Reading, Canvas, Bases, Inline Title, Error Boundary, Hover Popover, core command registration, and every built-in core plugin source for hard-coded visible English, while preserving dynamic names, ids, filenames, paths, and thrown messages as data.
