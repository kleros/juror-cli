# juror-cli

Headless TypeScript CLI that commits and reveals Kleros v2 juror votes on **Arbitrum One**
(chain 42161). One-shot commands, no daemon. Binary: `kleros-juror`.

**This tool casts a vote; it does not decide one.** The choice is always an input. Nothing here
reads evidence, resolves a dispute template, or discovers draws — see `CONTEXT.md` for the
casting/deciding line and `docs/adr/0001` for why it falls there.

**The primary consumer is an autonomous LLM juror agent, not a human at a terminal.** Several
defaults a human operator would expect invert under an agent consumer; where they do, `docs/adr/`
records the decision. A human is a debug surface only, so the CLI must be self-documenting.

There is no external specification. The seeding spec this was built from is retired and outdated;
the invariants below, the ADRs and the tests — the salt and commitment vectors above all — are the
source of truth.

Status: Slice A shipped — `status`, `salt`, `commit`, `reveal`. `vote` and `recover` are next.
Published to npm as `kleros-juror-cli@0.1.0`, renamed to `@kleros/juror-cli`; no transaction has
been broadcast in production yet. A published version makes a salt-derivation change breaking for
installs beyond this machine. **The `kleros-juror-cli/v1/…` seed and salt tags are not the package
name and MUST NOT follow it** — never find-and-replace the old name.

```
pnpm test             # unit + fork tests (each fork suite skips when its prerequisite is absent)
pnpm test:fork        # spawn an Arbitrum One fork on :8546 and run only the fork tests
pnpm test:acceptance  # commit -> passPeriod -> reveal on a pinned fork; needs KLEROS_ARCHIVE_RPC
pnpm typecheck
pnpm lint             # biome check .   (`pnpm exec biome check --write .` to fix)
pnpm dev status --dispute 154 --round 0 --votes 0 --address 0x...   # no `--`: it reaches the CLI
```

## Invariants

Guard rails that hold before you've read anything else. Where an ADR records the reasoning, it is
named.

- **The salt is recomputed, never stored.** `reveal` re-derives it. It MUST NOT read a salt from
  `commits.jsonl`, which is a non-authoritative audit record.
- **The seed is derived from a wallet signature and never persisted** — `keccak256(sign(...))`,
  proved deterministic by signing twice at startup. The **signer address and seed source are locked
  for the life of any in-flight commitment**; changing either yields an unrevealable vote.
  `ADR-0003`
- **Vote IDs canonicalise identically in every command** — dedupe, **numeric** ascending sort,
  decimal, comma-joined; the same canonical array goes on chain. Lexicographic sort puts `10`
  before `9` and yields an unrevealable commitment.
- **`commit`, `reveal` and `vote` are never substituted for one another** — not by the CLI, not by
  an agent, and not by an upstream field. `kleros juror draws` supplies `actionRequired` as a
  **hint**; pre-flight MUST independently read `hiddenVotes` and the period and refuse a mismatch.
- **Evidence never enters this process.** Everything an agent reads from a dispute is authored by
  parties with an interest in the outcome. The separation is structural, not procedural: attacker-
  authored text has no path to the signing key because this tool never reads any.
- **Never print the seed. Never print the salt during `commit`** — logging it defeats the hiding.
- **Chain 42161 only, Classic and Gated kits only.** Refuse anything else; refuse Shutter by name.
- **Simulate every state-changing call, and broadcast only on explicit `--broadcast`.** The default
  is plan → simulate → stop. There is no human confirmation gate and nothing upstream provides one.
  `ADR-0004`
- **Failure semantics live in the JSON payload, not the exit code.** The consuming agent sees
  stdout and stderr merged into one middle-out-truncated buffer and an effectively binary exit
  status. So: JSON on stdout, stderr silent unless `--verbose`, output kept small, and a stable
  `code` field on every error. Exit codes stay stable for shell callers, but they are not the machine
  contract.

Vote windows are short (court 34: 1800s) and **not guaranteed** — both periods end early once every
juror has acted, and `passPeriod` is permissionless. Fail loudly and fast; never retry quietly.

## Stack

`incur` (pinned as `@kleros/agentkit` pins it) · `viem` · Node >=22. Runtime truth: `package.json`.

Addresses and ABIs are **imported** from `@kleros/kleros-v2-contracts` in `src/core/deployment.ts`,
never hand-copied. It stays a **devDependency** because tsup bundles it (`ADR-0006`); importing its
root throws, so use the `cjs/deployments` subpath. `deployment.test.ts` still asserts both
deployed-versus-`master` fingerprints (`require` strings, a single `getDegreeOfCoherence`) — that is
what makes the import safe, and it must keep passing. It also pins the five addresses against
historical literals: **salt derivation folds in the kit address**, so a moved address strands
in-flight commitments. Do **not** add `@kleros/kleros-sdk`; it is a higher-level layer this scope
does not need and it drags a conflicting zod major.

RPC only — no subgraph. Every pre-flight read is a plain `eth_call`, with one exception:
`identifyDisputeKit` reads the `DisputeKitCreated` log to map kit addresses to their IDs. Its
**results** are bounded — four events on Arbitrum One — but the query itself is full-range and
**unfiltered** (`fromBlock: 0n`, no topic filter), so it falls back to `getDisputeKitsLength` /
`disputeKits` when a fork or a range-capping provider returns nothing. Do not read "bounded" as a
bounded *query*: a topic-filtered log read is cheaper than what this already does.

Layout mirrors `@kleros/agentkit` so the eventual port is close to a file move: framework-free
`src/core/` returning `KlerosResult<T>`, thin `src/commands/` owning incur and the CTA blocks.
`ADR-0001`

## Domain docs

`CONTEXT.md` is the glossary — use its terms, avoid the synonyms it lists. `docs/adr/` records the
six decisions that a reader would otherwise question. Convention: `docs/agents/domain.md`.

The **CLI surface is machine-checked** against that glossary: `vocabulary.test.ts` renders `--help`
and `--llms-full` and fails on a term no command or option description may ever use. Its list is
narrower than the `_Avoid_` lines on purpose — read the comment before widening it. Prose is not
checked, so `README.md` and `CONTEXT.md` may still name a term to contrast it.

`README.md` is the only doc written for a stranger. It restates the command surface, the option
defaults and the JSON envelope — so a change to any of those is a change to it too.

## Reference material

Read-only, outside this repo, via a **gitignored symlink** — absent in a fresh clone. It is not a
build dependency; it exists to be read.

| Symlink              | Recreate with                                                        |
| -------------------- | -------------------------------------------------------------------- |
| `coinbase-agentkit/` | `ln -s ../../kleros-playgrounds/coinbase-agentkit coinbase-agentkit` |

`coinbase-agentkit/` is upstream `coinbase/agentkit` @ 0.11.0. **This tool does not ship as an
action provider** — `ADR-0002`. The symlink stays for reading source if that is ever revisited.

Other Kleros research artifacts exist outside this repo (a PRD, an agent-authored best-practices
document). The PRD is obsolete and anything else is advisory only; this repo governs.

## Agent skills

- **Issue tracker** — GitHub Issues on `kleros/juror-cli`, which is **public**: an exploitable
  finding goes in a draft security advisory, never an issue. See `docs/agents/issue-tracker.md`.
- **`.scratch/` is history, not a tracker** — the features filed before GitHub Issues. Migrated
  issues link back to those files by path, so never move, migrate or delete them, and file nothing
  new there.
- **Triage labels** — the five canonical roles, each label string equal to its name. See
  `docs/agents/triage-labels.md`.
- **Domain docs** — single-context: `CONTEXT.md` is the glossary (skills that say `GLOSSARY.md`
  mean it) and `docs/adr/` the decisions. See `docs/agents/domain.md`.
