# lifecycle-feature

> **In plain terms:** Some features involve two people taking turns on the same thing. An
> invitation someone sends and someone else accepts. A request that gets approved. An offer that
> gets countered. These break in a particular way: one screen says it's still pending while another
> says it's agreed, a button appears for somebody who shouldn't have it, or a person manages to
> accept their own offer. This is a recipe an AI assistant follows to build that kind of feature
> once, properly. It's the difference between a board game with a rule card in the box and one
> where four people each remember the rules differently.

The rest of this page explains the shape it insists on and the questions it asks first. You don't
need to already know the tools to follow along.

## What am I looking at?

This is a **skill for AI coding agents** like Claude Code, Cursor, Codex, and Gemini CLI, the tools
that write code alongside you. A skill is a set of written instructions the agent reads and
follows. You install it once, and when you start building something two parties share, the agent
recognizes the shape and follows this.

## The problem it fixes

The feature sounds small. A few named stages, a few actions, and some rules about who can do what
and when. Because it sounds small, it gets built by hand in each place it appears: once in the list
view, once on the detail page, once on mobile, once in the email, once in the permission check.

Then it drifts. Six places that each have their own opinion about whether this invitation can still
be withdrawn. One of them is wrong, usually the one you're not looking at, and if the thing involves
money or a contract then being wrong is expensive rather than untidy.

## The shape it insists on

**One table** that lists every possible move: from which stage, by which party, to which stage, and
what happens as a result. **One piece of code** that is the only way a move can ever be applied.
And then **everything else is derived from those two**: the permissions, which buttons you can see
right now, what the screens display, the wording, the tests, the notifications, the documentation.

> Nothing outside the table decides a rule. Nothing outside that one piece of code changes the
> state.

There's a subtler rule underneath that one. The current stage is **worked out from the facts**
rather than stored as its own field. You store that an offer was accepted and when; you don't store
the word "negotiating", because the moment you do, you have two sources of truth and they will drift
apart. Store what happened, calculate what that means.

## First, it tries to talk you out of it

This is the part that makes the skill worth having, because the machinery above is genuinely
overkill for most features. Five questions, answered before anything gets designed:

| Question | A yes looks like |
|---|---|
| Do two or more parties act on the same thing? | an owner and a contractor; a buyer and a seller; someone requesting and someone approving |
| Is there a small set of named stages? | pending, negotiating, agreed, hired, declined, expired |
| Must the rules agree across several screens? | a list row, a detail page, a phone, and an email all saying the same thing |
| Is money, a contract, or trust at stake if someone moves wrongly? | hiring without payment, accepting twice, acting out of turn |
| Could somebody act when it isn't their turn? | the sender accepting their own offer; a stranger changing a status |

**Three or more yes: use the recipe.** Fewer, and it's an ordinary form with a single owner, so the
skill tells the agent to say that plainly and build it the simple way. Building the machinery
anyway is how you end up maintaining a tiny, undocumented rules engine that only ever had one
customer.

## The two phases everybody skips

**The survey comes before the design, and it's a table, not a paragraph.** Every stored status
value, every place in the code that writes one, every helper that works out which buttons to show,
every notification, every permission check. With the file and the line number. Each one marked
working, wrong, or dead. The design then has to point at that table, because a problem that isn't
in it was a guess.

**The permissions ship first, on their own, before anything else changes.** The rollout is six
pieces of work stacked in order, and the first is nothing but locking down who is allowed to touch
the thing, with no change to what anyone sees. Fix the interesting part first and the hole stays
open for the length of the project. There's a related rule worth stealing whatever you're building:
any new scheduled job the work introduces ships **switched off**, with a one-off quiet catch-up run,
so its first tick can't email a decade of old records.

**And there's a floor for when six is not affordable.** A recipe that needs six pieces of work gets
abandoned in exactly the week it would have helped, and abandoning it means somebody writes the
rules by hand in six places instead. So the skill names the two that still help on their own: the
permissions, and then the rules table plus what the screens read, added without anything reading
from them yet. Stop there and say so in writing. That's honest debt with a named next step rather
than a half-built pattern.

## What's in the box

- **`SKILL.md`** is the recipe: the gate, the seven phases, the common mistakes and the red flags.
- **`EXAMPLE-invitation.md`** is one complete table filled in, for an owner inviting a professional
  to a job where either side can counter on price. Read it before writing your own, because three
  things only make sense seen whole: the actor that **flips** between stages (in the first stage
  only the professional may accept; once they counter, the offer is theirs, so only the owner may),
  the combinations left out **on purpose** and what each refusal says, and the two moves that never
  get an address of their own because nothing outside the system should be able to call them.
- **`reference.md`** holds the shapes and the decisions already made once: what a row in the table
  needs, the fixed order of operations when a move is applied, and which error to return when.
  That exists so you aren't re-arguing the same small decisions on every project.

**On languages, honestly.** The recipe itself is about shapes and travels anywhere. The code
examples in `reference.md` are Python and React, because that's where this was first built, so
`reference.md` opens with a table translating the five things that need it into Go, Rails and Node.
Four are renames. One isn't: Python lets a status change happen as a side effect of saving, which is
the disease this whole recipe treats, so a language without that feature starts a step ahead rather
than a step behind.

## Signs to stop, from the skill itself

A second copy of the one piece of code that applies a move. A status name typed loose inside a
screen. A button whose visibility is worked out from the status inside a page. A new "if the status
is X" inside a background job. A database change for a stage that could have been worked out. A
single piece of work that changes both the permissions and the behaviour.

## Requirements

- An AI coding agent that supports the Agent Skills standard: Claude Code, Cursor (2.4+), Codex,
  Gemini CLI, or GitHub Copilot CLI. They all read the same `SKILL.md` format, so one copy works in
  any of them.

## Install

| Agent | Where skills live |
|-------|-------------------|
| Claude Code | `~/.claude/skills/` (your home folder, used by every project) |
| Codex | `~/.agents/skills/` |
| Gemini CLI | `~/.agents/skills/` |
| GitHub Copilot CLI | `~/.agents/skills/` |
| Cursor (2.4+) | `.cursor/skills/` **inside the project** you want to use it in |

```bash
# Claude Code
git clone https://github.com/Tetteh-Yaw-Precious/lifecycle-feature.git \
  ~/.claude/skills/lifecycle-feature

# Codex, Gemini CLI, or Copilot CLI
git clone https://github.com/Tetteh-Yaw-Precious/lifecycle-feature.git \
  ~/.agents/skills/lifecycle-feature
```

Project-specific file paths and reference implementations belong in a separate profile of your own,
never in this skill. That separation is deliberate: the recipe travels between projects, the paths
don't.

## Companion skills

| Skill | How it relates |
|---|---|
| [`feature-cycle`](https://github.com/Tetteh-Yaw-Precious/feature-cycle) | The loop around this. It calls this skill at its build step, and the rest of the loop (survey, issue, isolated branch, checking, review, proving it runs, documenting, closing) still happens around it. |
| [`architecture-guardrails`](https://github.com/Tetteh-Yaw-Precious/architecture-guardrails) | Phase six asks for two automatic checks: nothing outside the approved path writes a status, and no status name appears loose in a screen. That skill is how you build checks that hold. |
| [`vertical-slice-matrix`](https://github.com/Tetteh-Yaw-Precious/vertical-slice-matrix) | A list of stages is the textbook case of a fact that leaks into many files. Read that before adding a new stage. |

## License

MIT.
