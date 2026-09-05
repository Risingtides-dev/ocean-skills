---
name: prove-the-test-can-fail
description: Use when a test suite, mutation gate, differential fixture, or CI check is being trusted as evidence — especially before calling work "done", porting code to another language, or claiming coverage. Catches the failure mode where a check reports success while proving nothing: a red baseline that scores every mutant as caught, a mutant that never compiled, a fixture case whose name promises a bound it never reaches, and an assertion that cannot go red.
license: MIT
---

# Prove the Test Can Fail

## Overview

A passing check is not evidence. It is evidence *only* if you have seen it fail
for the specific reason it claims to guard.

Most bad test suites are not wrong about the code. They are wrong about
themselves — they report on an invariant they stopped exercising, or never
exercised. The suite is green, the dashboard is green, and the invariant is
unguarded. This is worse than no test, because a name like
`asset_id_too_short` actively tells the next person that the bound is covered.

The discipline is one question, asked of every check:

> **What exact change would make this go red? Have I watched it do that?**

If you cannot name the change, the check is decoration. If you can name it but
never ran it, the check is a hypothesis.

## The Five Ways a Green Check Lies

### 1. The baseline was already red

A mutation score means *"the suite rejected this change."* That is only
meaningful if the suite **accepts the unchanged tree**.

If the baseline is failing, every mutant is scored "caught" — the tests failed,
after all — and the harness prints a perfect score while proving nothing.

```python
# Run the UNMUTATED tree first. Refuse to score anything if it is not green.
baseline = run_tests()
if baseline != "passes":
    print("BASELINE is not green — fix that before trusting any mutant score")
    return 1
```

Red-prove the gate itself: break one constant, confirm the harness *refuses to
score* instead of reporting 67/67.

### 2. The mutant never compiled

The subtle one. A harness usually decides "caught" by looking for failure in
the output — and compiler errors are failure output.

```python
# WRONG: a mutant that does not build is scored as detection.
return "test result: FAILED" in output or "error[" in output
```

A mutant that does not compile proves **nothing**. It never ran. Worse, the
inverse bug hides in the same line: a macro syntax error emits a bare `error:`
with no code, so that mutant gets scored **survived** — a phantom coverage gap
that sends you hunting for a hole that does not exist.

Use three verdicts, not two:

| verdict | meaning | action |
|---|---|---|
| `caught` | tree built, suite rejected it | the invariant is guarded |
| `survived` | tree built, suite accepted it | **real gap** — write the test |
| `invalid` | tree did not build | **broken mutant** — fix it, it proved nothing |

Make `invalid` fail the run. Otherwise it silently converts to a false "caught".

### 3. The mutant was a no-op

A mutant has to actually change behavior. It is easy to write one that changes
only syntax:

```rust
let keys: Vec<&str> = ...;        // original
let mut keys: Vec<&str> = ...;    // "mutant": adds mut, sorts nothing
```

That survives every suite, forever, and looks exactly like a coverage gap.
When a mutant survives, first ask whether it changed anything at all.

### 4. The case name promises a bound it never reaches

The most dangerous kind, because it reads as coverage in review.

```python
# named "too short" — and it is 76 characters, inside the 72..=134 bound.
# It tests nothing. Both implementations accept it.
"short-asset-" + "b" * 64
```

Compute your boundaries, do not eyeball them. For a bound `72..=134` over
`prefix + "-asset-"(7) + digest(64)`, the real cases are prefix 1 (=72, min),
prefix 63 (=134, max), prefix 64 (=135, must refuse).

### 5. An earlier bound shadows the one you meant to test

A case can be refused for the *wrong reason* and still look like it passed.

Testing a 100-device cap with 101 devices proves nothing if each device also
appears in a summary list that is itself capped at 100 — the list bound refuses
first, and the device cap is never reached. The test goes green whether or not
the device cap exists.

Construct the case so **only** the target bound can reject it: 101 devices whose
summary lists stay empty. Then the cap is the sole remaining explanation.

## How to Find These

Mutation testing is the cheapest way, and it does not need a framework. A
~60-line script that patches an anchor string, runs the suite, and restores the
file will find real holes today:

```python
MUTANTS = [
    ("worker exit code 0 accepted as a failure",
     "src/results.rs",
     "if !(1..=255).contains(&exit_code) {",
     "if !(0..=255).contains(&exit_code) {"),
]
```

Anchor on the **invariant**, not on formatting. When a formatter moves the
code, the anchor stops matching — and a skipped mutant is a failure too, because
the harness silently stopped testing that invariant. Report `skipped` and exit
non-zero.

Run the gate serially (`--test-threads=1`). If the wider suite has a
process-wide race — shared environment variables are the classic one — a verdict
can otherwise be a scheduling artifact rather than a fact about the code.

## Checklist

Before calling a check evidence:

- [ ] The baseline passes, and the harness verifies that before scoring.
- [ ] Every mutant compiles; non-compiling ones are reported, not counted.
- [ ] Every surviving mutant actually changes behavior.
- [ ] Boundary cases are computed, and each sits exactly on/around its bound.
- [ ] Each case can only be rejected by the bound it names.
- [ ] A moved anchor fails the run instead of silently skipping.
- [ ] You have watched the check go red for the reason it claims to guard.

## Why This Matters

These are not hypothetical. In one session porting a device-automation runtime
between languages, this discipline found: three mutants scored "caught" that had
only ever produced compiler errors; one scored "survived" that was a no-op; a
fixture case named for a length bound that sat comfortably inside it; and a
device-cap case shadowed by a list cap. Every one of them had been reported —
by me — as proof. The suite was green the whole time.

The cost of the discipline is minutes. The cost of skipping it is a green
dashboard over an unguarded invariant, which is the exact condition under which
people ship confidently and break production.
