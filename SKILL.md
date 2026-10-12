---
name: lifecycle-feature
description: Use when a feature changes the state of something two or more parties share (an invitation, application, order, approval, contract, claim, payout, request) with named statuses and moves like accept, decline, withdraw, approve, counter, cancel, expire; when screens disagree about a status; when "who can do what right now" is decided in more than one place; or when a bug is a wrong or out-of-turn transition. Also use to decide that a feature is NOT a lifecycle and should stay plain CRUD.
---

# Lifecycle Feature

## Overview

A lifecycle is a small contract between parties: a few named stages, a few moves, and rules about who may make which move when. Build it as **one registry table** that declares every move, **one procedure** that is the only way to apply a move, and **folds** that derive everything else (permissions, allowed actions, read model, frontend vocabulary, tests, docs, activity, notifications). Nothing outside the table may decide a rule; nothing outside the procedure may change the state.

This skill is the build recipe. It plugs into `feature-cycle` at the Build step; the loop (audit, issue, worktree, verify, review, prove, document, close) still runs around it. A project profile (`lifecycle-feature-<project>` or the project's feature-cycle profile) supplies file paths and reference implementations; this file never names them.

## The gate: is this a lifecycle?

Answer all five before designing anything. Write the answers into the issue.

| Question | Yes looks like |
|---|---|
| Two or more parties act on the same thing? | owner and professional; buyer and seller; requester and approver |
| A small set of named states? | pending, negotiating, agreed, hired, declined, expired |
| Rules that must agree on every screen? | a list row, a detail page, a mobile view, a notification all say the same thing |
| Money, a contract, or trust on a wrong move? | hire without payment, accept twice, act out of turn |
| Someone could act when it is not their turn? | the sender accepting their own offer; a stranger changing a status |

**Three or more yes: use this skill.** Fewer: it is CRUD or a single-owner form. Say so explicitly, build it plainly, and do not add a registry. A registry over CRUD is an inner platform.

## The recipe

Every phase produces the artifact named in bold. A phase without its artifact is not done.

### 1. Audit: **the today table**

Before design, enumerate what exists, with file and line: every stored status value; every place that writes the status (views, serializers, signals, tasks, webhooks); every screen or helper that derives "what stage are we in" or "which buttons show"; every notification and activity write for the domain; every permission check on the domain's endpoints. Mark each live-and-working, live-and-wrong, or dead. The design must cite this table; a defect not in it is a guess.

### 2. Registry: **one table, one file**

One file per lifecycle declaring, for each `(stage, command)`: the actor (a role relative to the subject, never a user id), the target stage, an optional guard predicate, named effects, the notification recipient rule, the activity kind, whether the UI must confirm, and whether the command does external I/O. Stage is **derived** from stored fields, not a new stored column, unless a real fact must be stored. Load `reference.md` for the field list and defaults that have already been decided once.

Decide and write down: which party is "the actor" for turn-based moves (receiver of the standing offer, not "either party"); what happens on the terminal moves to any open sub-object (a negotiation, a hold); whether siblings settle on completion.

### 3. Procedure: **one way in, shared across lifecycles**

One `apply_command(subject, command, actor)` per project, parameterised by the registry, not one per lifecycle. Fixed pipeline: lock the row; resolve the party; derive the stage; find the row or refuse with 409; check the actor or refuse with 403; check the guard or refuse with 409; validate the payload; run effects; write the state naming every updated field; emit one domain event. Mount HTTP endpoints from the registry through a factory that refuses at boot to mount a command the table does not declare. External I/O (payments) runs untransacted before the command; the command then runs in its own short transaction.

If a second lifecycle arrives and the first builder is lifecycle-specific, generalise the builder first. Two builders is the mistake the recipe exists to prevent.

### 4. Folds: **read model, generated frontend registry, tests, docs, activity**

- Read model fields on every payload of the subject: `stage`, `awaiting`, `allowed_actions` for the viewer, `counterparty`, plus whatever the lifecycle agrees on (an agreed amount, who accepted). Computed by the same functions the procedure uses.
- A management command exports the registry to the frontend as a generated, committed file. CI diffs it.
- One parametrised test per registry row: named actor succeeds, other party 403, non-party 404, wrong stage 409, plus one per absent pair. Snapshot the exported registry.
- Each command names its activity kind; the domain event is the only source for activity and notifications (see the project's activity design if one exists).
- Docs and the state diagram are exports, not hand drawings.

### 5. Frontend: **render from stage, send commands**

- `selectStage(subject, viewer)`: server read model first, a legacy derivation as fallback only while old API responses can still arrive.
- Copy tables keyed by stage and viewer role, and by command: label, variant, toast, confirm dialog. Copy never lives in a component.
- One command hook: one mutation key per subject, optimistic target from the registry row, one invalidation set that includes lists and activity.
- One actions component that renders exactly `allowed_actions`, and one row model for lists that both parties' pages use. Delete the normalisers and status cards it replaces in the same PR.

### 6. Guardrails: **two ratcheting checks**

Add, with the project's guardrail harness: a backend check that no code outside the procedure writes the subject's status, and a frontend check that no stage or command string literal exists outside the generated registry. Both start at today's count and only go down. Boot-time checks in the registry module refuse a row missing an effect, an activity kind, or copy for a viewer role.

### 7. Rollout: **stacked PRs in this order**

1. **Stop the bleeding.** Party-only permission on the subject's endpoints, status read-only except the transitions the current frontend sends with actor checks, previous-status snapshot so signals fire on transitions only. No frontend change. Ships first.
2. **Registry and read model, additive.** Nothing existing changes shape.
3. **Frontend renders from stage.** One surface per PR, deleting a normaliser each time.
4. **Procedure and endpoints.** Old write path routes through the procedure until step 5.
5. **Frontend sends commands.** Delete the status maps and optimistic status writes.
6. **Close the loop.** Remove the old write path; guardrail counts reach zero; drop dead status values.

Each PR is stacked on the previous branch and retargeted to the base when its parent merges. Every scheduled task the rollout adds ships **off by default** behind a setting, with a one-off quiet backfill command, so a first tick cannot mass-notify a backlog.

## Common mistakes

| Mistake | What to do instead |
|---|---|
| Clone the first lifecycle's builder, mixin and actions component for the second | Generalise the shared pieces; only the registry table and effects are per lifecycle |
| Add a stored status for a new stage | Derive it; store only facts (an accepted offer, a timestamp) |
| Fix the domain first, permissions second | Permissions are PR 1; the hole is open until they land |
| Keep a post_save writer "for now" | Move it behind the domain event in the same rollout, or list it as debt in the close-the-loop step |
| Skip the frontend fallback | The new frontend meets old API responses during deploy; `selectStage` must survive both |
| Turn on a new beat task on deploy | Default off, quiet backfill, then enable |
| Save with `update_fields=['status']` | Name the timestamp too, or "last updated" never moves |
| Write the audit as prose | The today table with file:line, or the design cites nothing |

## Red flags

Any of these means stop and return to the recipe: a second `apply_command`; a status literal in a component; a button whose visibility is computed from `status` in a page; a new `if status == ...` in a signal; a migration for a state that could be derived; a PR that changes both permissions and behaviour.
