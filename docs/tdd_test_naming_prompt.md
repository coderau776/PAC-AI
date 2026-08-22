# TDD Test-Generation Prompt (Clean Code Compliant)

## How to use this
Copy the block below into your AI agent whenever you want it to write or review tests. Fill in the two bracketed lines at the top, then leave the rest as-is — it's written as instructions *to* the agent.

---

```
You are writing unit tests using Test-Driven Development. Every test must satisfy the rules below, drawn from Robert C. Martin's Clean Code (Ch. 2 "Meaningful Names" and Ch. 9 "Unit Tests").

LANGUAGE / FRAMEWORK: [e.g. Java + JUnit5 / Python + pytest / TypeScript + Jest]
EXISTING CONVENTION (paste a sample test method if this codebase has one, else write "none"): [ ]

── TDD PROCESS ──
Follow the Three Laws of TDD:
1. Do not write production code until a failing test exists for it.
2. Do not write more of a test than is needed to fail (including "does not compile").
3. Do not write more production code than is needed to pass the current failing test.
Work one behavior at a time, red -> green -> refactor, rather than dumping a full suite before any implementation exists — unless I explicitly ask for the full suite upfront for review.

── TEST NAMING (get this exactly right) ──
Every test name has three parts:
  [behaviorUnderTest]_[scenario]_[expectedOutcome]

- behaviorUnderTest: the capability being verified, in language a domain expert would recognize — not an internal method or class name.
- scenario: the one condition that makes this test different from its siblings.
- expectedOutcome: the observable result, in plain language — never the mechanism (no mock names, no internal flags, no variable names).

Avoid both failure modes:
  BAD, too technical (leaks implementation detail):
    test_calcDiscount_mockRepoReturnsNull_npeThrown
  BAD, too business (a run-on sentence, not a name):
    test_whenALongTimeLoyaltyMemberPlacesAnOrderOverOneHundredDollarsTheSystemAutomaticallyAppliesTheirEarnedDiscountToTheFinalTotal
  GOOD, balanced — names the behavior, not the code or the essay:
    calculateDiscount_premiumCustomerOrderOver100_appliesTenPercentDiscount

Test it like this: could a non-programmer domain expert read the name and understand what's guaranteed, without knowing how it's coded? Could a developer locate the exact code path from the name alone, without opening the test body? If either answer is no, rename it.

If EXISTING CONVENTION above isn't "none," match it instead of the shape given here — a consistent lexicon across the codebase matters more than any single "correct" format.

── TEST STRUCTURE ──
Build-Operate-Check, and only those three parts, per test:
1. Build — set up just the state this one scenario needs.
2. Operate — call the one behavior under test.
3. Check — assert the outcome.
Push setup noise into small, well-named helper functions so the test body reads as those three steps, not framework plumbing.

── ASSERTIONS ──
- One concept per test. If you're tempted to describe the test with "and," split it into two tests.
- Minimize assertions; multiple asserts are fine only when they all confirm the same single concept.

── F.I.R.S.T ──
Every test must be:
- Fast: no real I/O, sleeps, or network calls.
- Independent: passes alone and in any run order; no shared state between tests.
- Repeatable: identical result on any machine or environment.
- Self-validating: pass/fail only — no reading logs or diffing output by eye.
- Timely: written for the behavior you're about to implement, not bolted on after.

── COVERAGE ──
- Cover boundary conditions explicitly, not just the happy path.
- Don't skip trivial-looking cases — they're cheap and they document behavior.
- If a requirement is ambiguous, write the test anyway, mark it skipped, and say in a comment what's unclear — don't silently guess.

── OUTPUT FORMAT ──
For each behavior, give me:
1. The test name (per the rule above)
2. The test body (Build-Operate-Check)
3. One line naming the single concept it verifies

── SELF-CHECK BEFORE YOU SHOW ME THE TESTS ──
Review every test you just wrote against this list, and rewrite any that fail before presenting the final output:
[ ] Name describes a behavior, not an implementation (no mock/internal-method/variable names in it)
[ ] Name is not a run-on business sentence — you could say it out loud in one breath
[ ] Name follows the same shape as the other tests in this set (or matches EXISTING CONVENTION)
[ ] Test verifies exactly one concept
[ ] Test would still make sense if the production method were renamed — i.e. it describes behavior, not internal structure
```

---

## Grounded in
- Ch. 2, *Meaningful Names* — "Use Solution Domain Names," "Use Problem Domain Names," "Avoid Encodings," "Add Meaningful Context"/"Don't Add Gratuitous Context"
- Ch. 9, *Unit Tests* — Three Laws of TDD, Build-Operate-Check, Domain-Specific Testing Language, One Assert per Test, Single Concept per Test, F.I.R.S.T.
- Ch. 17, *Smells and Heuristics* — N2 "Choose Names at the Appropriate Level of Abstraction," N3 "Use Standard Nomenclature," T5 "Test Boundary Conditions"
