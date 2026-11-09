# vibe-coding-surgery

English | [简体中文](README.md)

A skill for reviewing and repairing code produced by AI.

It targets a specific state. A project has been modified by AI over and over, features have piled up, and the structure has drifted. Such projects usually still run and still ship, but every change costs more than the last one.

Two workflows are provided. `surgery` repairs code that is already tangled. `grow` adds functionality to existing code without damaging it. Both share the same hard rules. Produce a plan, wait for authorization, then act, and handle one independently revertible change at a time.

No dependencies. Language agnostic. Examples use TypeScript.

---

## The problem

AI writes code differently from people. It carries no memory across sessions, and each round it answers only the current error or request. The result is a small set of recurring shapes.

The same function written twice under different names. Existing logic left untouched while another conditional layer is wrapped around it. An interface extracted in advance for future flexibility, with exactly one implementation. Exceptions caught and discarded so the test goes green. A dependency that does not exist. A test named for the error path, with no assertion for it. Documentation frozen at the previous round, out of step with the code.

None of these is severe on its own. Once they accumulate, the project reaches a state where it runs but nobody wants to touch it.

Existing tools generally do not address this. Linters check syntax and style. Type checkers check type correctness. Coverage tools count tests. None of them answers whether two implementations are the same thing.

---

## Core features

**Systematic review.** Seven cross-check methods covering tool reports against source, multiple copies of one fact, documentation against code, similarly named functions, adjacent implementations in one file, callers against callees, and the project's own records against its code.

**Problem classification.** Ten recurring shapes found in AI-generated code, each with cause, symptoms, handling, and the boundary at which the classification stops applying.

**Sliced repair.** Six structural actions applied in the order delete, merge, flatten, split, move, rename. Each slice is capped at 5 files or 300 lines and can be verified and reverted on its own.

**Safe feature addition.** Two tactics. Sprout writes the new logic entirely in new files, with a single call added to the existing file. Wrap builds a same-named wrapper around the original function, performing pre-validation and post-audit while keeping the signature unchanged.

**Constraint file generation.** Templates for seven files that keep the same problems from reappearing after the code is repaired.

**Verification.** Three equivalence techniques, destructive spot checks, diff purity auditing, and commit tagging.

---

## Characteristics

Plan before acting. The first round produces only a report and a plan. No file is modified before explicit authorization.

Every change reverts on its own. Each slice is committed immediately, with structural changes and behavior changes kept apart, so reverting means undoing one commit.

Verification works without tests. For projects with no test suite, equivalence is established by copying the original implementation to a temporary file and comparing output on identical input, or by keeping a temporary export and running both versions side by side.

The output states its coverage. A report must say what was read, what was not read, what was run, and what was not run. A finding of no problems comes with its scope.

Code repair and process constraints run together. Repairing code does not stop the problem from returning, so constraint files and hooks change the conditions under which the next round is generated.

Scale adapts. Under 150 source files is scanned in one pass, 150 to 800 is split into sections, and above 800 starts with skeleton distillation.

Degradation paths are explicit. Missing git, missing tests, and an environment that cannot run commands each have a permitted fallback, and any fallback is declared on the first line of the report.

Cost is tiered. Quick scan, standard scan, and deep scan, chosen by need.

---

## Advantages

The output is located contradictions rather than a count of problems. File size rankings, complexity numbers, and empty catch counts describe scale. They cannot tell you whether two implementations are equivalent or whether documentation matches the code, which is what drives the cost of change upward.

Repair can be interrupted. Each slice is a complete step, so stopping midway leaves no half-finished state.

Conclusions are reproducible. Every finding states which two sources were compared and how.

Misclassification has a defined exit. Each of the ten shapes carries a boundary beyond which it stops applying, and anything that cannot be classified is treated as genuine business complexity and kept out of the repair queue.

Rules can be enforced by tooling. Beyond the constraint files, git hooks can check commit tags, line limits, and diff purity at commit time.

---

## Who it is for

Projects that AI has modified repeatedly. Projects where feature accumulation has outpaced structure. Projects with missing tests and stale documentation. Projects that must keep growing on top of old code. Codebases inherited from another person or from an AI session.

Not for greenfield projects, formatting-only work, full security audits, performance tuning, or cases where unfamiliarity with a domain is mistaken for a code problem.

Exemptions include files under 30 lines, style and copy and configuration changes, small changes that follow an existing pattern, and explicit requests to write code directly. Any of these skips the workflow.

---

## Quick start

```bash
git clone https://github.com/Colin-Fumina/vibe-coding-surgery.git
cp -r vibe-coding-surgery ~/.agents/skills/
```

Trigger it with a natural language request.

```
This codebase is a mess, help me sort it out
Review what the AI wrote here
Add an export feature without breaking the existing code
Is this project still salvageable
```

The first round returns a report and a plan, then stops for authorization.

---

## Workflow

```
environment probe → scan → diagnosis → report and plan → wait for authorization
→ execute one slice → verify → return to planning
```

The first step is the environment probe. Whether git is available determines whether rollback is possible. Whether tests run determines whether equivalence can be verified. Whether build and typecheck run determines how deep the analysis can go. Source file count determines whether to split into sections, above 150, or start with skeleton distillation, above 800. Any missing capability is declared on the first line of the report.

Both workflows share this sequence and differ only in how changes are made.

`surgery` handles code that is already tangled and provides six actions in the order delete, merge, flatten, split, move, rename. The default order matters. Deleting first reduces the surface. Merging exposes real duplication. Flattening makes room for later splitting. Splitting comes late because doing it first scatters concentrated complexity across many small files. Moving stabilizes the structure. Renaming comes last because it must answer to the final structure.

Three cases may jump the queue. A defect that triggers immediately. Code that cannot currently be verified and needs a seam opened first. Naming that blocks comprehension. Any jump is justified in the plan.

`grow` handles adding functionality to existing code and provides two tactics.

Sprout implements the new feature entirely in new files, with its tests in the same new files, and adds a single call to the existing file without expanding any new business logic there.

Wrap renames the original function to `xxxLegacyInternal` and creates a same-named function around it that performs pre-validation, calls the original, then post-audits, keeping the public signature and return value unchanged.

Wrap changes stack depth and function identity. If the original function is reached through reflection, decorators, serialization, or `Function.name`, use Sprout instead. Before deciding, search the repository for the original name together with `name`, `bind`, and `caller`.

The main risk in `grow` is drifting into opportunistic refactoring. Problems noticed while adding a feature go into `FINDINGS.md` and are handled later.

---

## Hard rules

### The first round produces a plan only

Before explicit authorization, no file is modified, including Markdown, JSON, comments, and formatting. No state-changing command runs. No directory is created or removed. Nothing is committed.

The threshold for authorization is defined. The following count as authorization: go ahead, start, do it, proceed, execute. The following do not: you decide, whatever, change it if you think it should change, I just want to look, help me analyze it, don't be too conservative. Supplying a target path does not constitute authorization.

Authorization for one slice is not authorization for the next. Confirmation is requested after every slice.

### Boundaries of a slice

A slice must be independently verifiable and independently revertible. The cap is 5 files or 300 lines, and anything larger is split.

Each slice is committed immediately, with a message beginning `[surgery]` or `[behavior]`. The first means structure only, the second means behavior changed and the acceptance criteria are stated. The two are never mixed in one commit.

Without commits there is no slice history, rollback becomes a whole-tree `restore`, and the ability to undo one slice is lost.

### Verification criteria come first

If it cannot be stated how a broken change would be detected, the change is not made.

Projects without tests use equivalence verification, in increasing cost. Copy the original implementation to a temporary file and compare output on identical input. Keep a temporary export and run both versions against the same cases. Sample empty, single, maximum, and just-out-of-range values by hand.

If none of the three is feasible, the slice does not touch code. Instead, one line of synchronization comment is added at each of the two locations.

---

## Core method

The core action is comparison, placing two sources that should agree side by side.

| | Comparison | What it surfaces |
|---|---|---|
| F1 | Linter report ↔ source | A linter reported a `return` above seven hooks. Reading the source showed an early return when the goal was empty. Tracing the caller showed empty arrays were already filtered, so severity dropped from immediate to conditional |
| F2 | Multiple copies of one fact | Version number was 0.1.59 in `build.gradle`, 0.1.8 in `AGENTS.md`, 0.1.31 in the README, 0.1.37 in the backlog |
| F3 | Documentation ↔ code | `findings.md` claimed zod validation. `git grep -E "z\.object|safeParse"` returned nothing. Validation was hand-written in `schema.ts` |
| F4 | Similarly named functions | Two `toMinutes` implementations with byte-identical bodies |
| F5 | Adjacent functions in one file | Same meaning under different names, findable only by reading in order |
| F6 | Caller ↔ callee | Determines whether a finding is immediate or conditional |
| F7 | Project records ↔ code | `backlog.md` noted that `react-wheel-picker` was left behind after a custom WheelPad was written. The dependency had no references |

F2, F3, and F7 are the cheapest and should be done first.

Seven related methods exist as well, covering literal values, environment variables, entry points, test names against assertions, types against runtime, dual environments, and shapes across layers.

A conclusion must name both sides of the comparison. Saying that something is duplicated is not a finding. A location and the comparison itself are required.

---

## Common problem patterns

Read this table before scanning so findings can be classified. The full version in `references/slop-patterns.md` covers causes and the boundaries at which each classification stops applying.

| | Pattern | Symptoms | Handling |
|---|---|---|---|
| P1 | Duplicate twins | Two byte-identical function bodies under slightly different names | Merge |
| P2 | Patch stacking | Five to eight nested conditionals, each with a one-line justification | Flatten |
| P3 | Premature abstraction | One interface, one implementation, one factory, four files touched per change | Delete |
| P4 | Hallucinated dependency | Installed with zero references, or `npm view` finds nothing | Delete |
| P5 | Swallowed errors | `catch { return [] }`, with no way for the caller to learn of failure | Throw, log, or declare a fallback |
| P6 | Fake tests | Test name promises an error path, assertion has no `toThrow` | Add the assertion or delete the test |
| P7 | Documentation drift | Documented dependency not installed, documented command missing | Fix the documentation |
| P8 | State creep | Derived data stored separately, effects writing to another store | Collapse into one-directional flow |
| P9 | God files | One file of 2000 lines carrying seven responsibilities | Split by intent |
| P10 | Type escapes | `any` density near the top, or chains of `as unknown as` | Fix the data shape |

Anything that cannot be classified is likely business complexity and should be re-evaluated against the health criteria.

When one pattern appears three times or more, the problem is in the process. Fixing instances one by one lets it reappear elsewhere, so constraints are added first.

---

## Output

The first round contains a report, a plan, and a request for authorization.

The report follows this structure.

```
0. Environment and boundaries   baseline commit · test availability · fallbacks · assessment
1. Scale and scan mode          source file count · full or sectioned · skeleton distillation
2. Findings                     location · description · evidence · severity · disposition
3. Overall conclusion           counts per disposition · health criteria results
4. Suggested order              slice 1 through N, goals only
5. Plans                        one page per slice
6. Deferred items               three lines each, why not now, what changes if done, unlock condition
7. Coverage statement           read, not read, run, not run
8. Uncertainties
```

Sections 6 and 7 are the easiest to trim and the most indicative of review quality. A conclusion of no problems must carry its scope.

One plan per slice.

```
[Slice N of M]
where       <file and line> (about X lines)
why         <one sentence with evidence>
how         <delete / merge / flatten / split / move / rename / Sprout / Wrap>
related     <callers to update, at most 5>
not doing   <explicitly out of scope for this slice>
break check <what to run, what to observe>
rollback    <command; whether it reverts alone>
impact      <file count, data or interface changes>
```

The "not doing" line bounds the change. Without it, work tends to spread into adjacent code during execution.

---

## Constraint files

Repairing code does not stop the problem from returning. The next AI session entering the project will follow the same patterns.

`references/templates.md` provides templates for seven files.

| File | Responsibility | Limit |
|---|---|---|
| `AGENTS.md` (root) | Hard rules and index | 200 lines / 2000 tokens |
| `<module>/AGENTS.md` | Responsibilities, non-responsibilities, dependency boundaries | 8 to 15 lines |
| `CONTEXT.md` | Glossary, one name per concept | Grows with the project |
| `docs/adr/NNNN-*.md` | Decisions and rationale | One page each |
| `docs/FINDINGS.md` | Known problems and why they remain unfixed | Grows with the project |
| `docs/PLAN-<date>.md` | Changes, order, and rollback for this round | One per round |
| `TASK.md` | Contract for a single task | Half a page |

One principle deserves separate mention. A rule that tooling can enforce does not belong in documentation. Documentation carries only what tooling cannot cover, which is boundaries, responsibilities, prohibitions, and rationale.

Rules should be phrased as prohibitions, for example "never modify core computation logic without a passing existing test suite". Advisory phrasing invites the model to skip the rule and infer instead.

Two git hooks are optional. `commit-msg` enforces the commit tag, and `pre-commit` enforces the 300-line cap and diff purity. Installing hooks changes repository behavior and requires confirmation first.

---

## Installation and structure

The repository root is the skill root, so it can be copied into a skills directory directly after cloning.

```
vibe-coding-surgery/
├── SKILL.md                      main file, always loaded, 610 lines
├── README.md                     Chinese documentation
├── README.en.md                  English documentation
├── LICENSE
└── references/
    ├── finding-methods.md        methods F1 to F14, read before scanning
    ├── slop-patterns.md          ten problem patterns with causes
    ├── commands.md               symptom search commands, used to rule out
    ├── templates.md              constraint file templates and hook scripts
    └── roles.md                  red and blue role split
```

The main file is always loaded and `references/` is read by phase. A quick scan loads roughly 9 to 11K tokens.

---

## References

Google eng-practices provides the review standard, whose goal is steadily improving overall code health rather than perfection. Michael Feathers' Sprout and Wrap are the basis for `grow`. Martin Fowler's refactoring catalog is the source of the problem classification. Microsoft's documentation on workspace instructions anti-patterns explains why linter-enforced rules should stay out of prose documentation.

Observations on AI-generated code include commits over 500 insertions, dependency hallucination, tests covering only the happy path, premature abstraction, architectural drift from context decay, and test suites with a mutation kill rate around 40 percent.

Designs deliberately not adopted include multi-layer governance boards, mandatory read-only gating, fifteen-field plans, rule count caps, and phased roadmaps. Each of these turns the skill itself into something that needs maintenance.

---

## License

MIT
