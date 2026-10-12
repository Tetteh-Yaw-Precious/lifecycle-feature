# Lifecycle Feature

A skill for the features that break in production: the ones where two parties act
on the same thing, and the screens stop agreeing about what state it is in.

An invitation, an application, an order, an approval, a contract, a claim, a
payout. A few named stages, a few moves (accept, decline, withdraw, counter,
cancel, expire), and rules about who may make which move when. That sounds
small, which is why it is usually built by hand in six places and then disagrees
with itself.

## The shape it insists on

**One registry table** declaring every move. **One procedure** that is the only
way to apply a move. **Folds** that derive everything else: permissions, allowed
actions, read model, frontend vocabulary, tests, docs, activity, notifications.

> Nothing outside the table may decide a rule. Nothing outside the procedure may
> change the state.

Stage is **derived** from stored facts, not kept as its own column. You store that
an offer was accepted and when; you do not store "negotiating", because a stored
stage is a second source of truth that drifts from the facts underneath it.

## First, the gate: is this even a lifecycle?

The skill opens by trying to talk you out of it. Five questions, answered before
any design:

| Question | Yes looks like |
|---|---|
| Two or more parties act on the same thing? | owner and professional; buyer and seller; requester and approver |
| A small set of named states? | pending, negotiating, agreed, hired, declined, expired |
| Rules that must agree on every screen? | a list row, a detail page, a mobile view and a notification all say the same thing |
| Money, a contract, or trust on a wrong move? | hire without payment, accept twice, act out of turn |
| Someone could act when it is not their turn? | the sender accepting their own offer; a stranger changing a status |

Three or more yes: use the recipe. Fewer: it is CRUD or a single-owner form, so
say that out loud and build it plainly. **A registry over CRUD is an inner
platform**, and the skill would rather you skipped it than built one.

## The two phases people skip

**The audit produces a table, not prose.** Every stored status value, every place
that writes the status (views, serializers, signals, tasks, webhooks), every
helper that derives "which buttons show", every notification, every permission
check. With file and line. Each marked live-and-working, live-and-wrong, or dead.
The design has to cite that table, because a defect that is not in it is a guess.

**Permissions ship first, alone.** Rollout is seven stacked PRs and the first one
is party-only permission checks, status made read-only except the transitions the
current frontend already sends, and a previous-status snapshot so signals fire on
transitions only. No frontend change at all. Fix the domain first and permissions
second and the hole stays open for the length of the project.

The other rollout rule worth stealing: every scheduled task the rollout adds
ships **off by default** behind a setting, with a one-off quiet backfill, so the
first tick cannot mass-notify a backlog of historical rows.

## Install

```bash
git clone https://github.com/Tetteh-Yaw-Precious/lifecycle-feature \
  ~/.claude/skills/lifecycle-feature        # personal, every project

git clone https://github.com/Tetteh-Yaw-Precious/lifecycle-feature \
  .claude/skills/lifecycle-feature          # project scope, shared with the team
```

For [Claude Code](https://claude.com/claude-code) and any harness reading the
same format. Stack-agnostic: the recipe names shapes (registry row, procedure
pipeline, folds) and never a framework. `reference.md` carries the registry field
list, the procedure's fixed pipeline order and the defaults that have already
been decided once, so you are not re-litigating 409 against 403 every time.

## Red flags it tells an agent to stop on

A second `apply_command`. A status literal in a component. A button whose
visibility is computed from `status` inside a page. A new `if status == ...` in a
signal. A migration for a state that could be derived. A PR that changes both
permissions and behaviour.

## Pairs with

- [feature-cycle](https://github.com/Tetteh-Yaw-Precious/feature-cycle): this
  plugs into its Build step. The surrounding loop (audit, issue, worktree,
  verify, review, prove, document, close) still runs around it.
- [architecture-guardrails](https://github.com/Tetteh-Yaw-Precious/architecture-guardrails):
  phase 6 asks for two ratcheting checks, one that no code outside the procedure
  writes the status and one that no stage literal exists outside the generated
  registry. That skill owns the ratchet mechanics.
- [vertical-slice-matrix](https://github.com/Tetteh-Yaw-Precious/vertical-slice-matrix):
  a stage list is the textbook **fact**, so route it there before adding a value.

Project-specific file paths and reference implementations belong in a profile
(`lifecycle-feature-<project>`), never in this file. That separation is the point:
the recipe travels, the paths do not.

## License

MIT.
