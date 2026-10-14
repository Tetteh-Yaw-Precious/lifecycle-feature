# Worked example: an invitation with negotiation

One complete registry, filled in. Read this before writing your own table. The lifecycle is an
owner inviting a professional to a job, where either side may counter on price.

Nothing here is a template to copy literally. It is here so you can see a whole table at once,
including the parts that only make sense in a whole table: the actor that flips, the pairs left
absent on purpose, and the rows that never get an endpoint.

## The parties and the stages

**Parties**, as roles relative to the subject, never user ids: `OWNER` (sent the invitation) and
`PRO` (received it).

**Stages**, all derived from stored facts. There is no `stage` column:

| Stage | Derived from | Terminal |
|---|---|---|
| `INVITED` | exists, no counter, no acceptance, not past expiry | no |
| `NEGOTIATING` | has an open counter-offer | no |
| `AGREED` | has an accepted amount, no payment | no |
| `HIRED` | has a confirmed payment | yes |
| `DECLINED` | declined at | yes |
| `WITHDRAWN` | withdrawn at | yes |
| `EXPIRED` | past expiry, and was `INVITED` or `NEGOTIATING` | yes |

## The table

| Stage | Command | Actor | To | Guard | Effects | Notify | Activity | Confirm | IO |
|---|---|---|---|---|---|---|---|---|---|
| `INVITED` | `accept` | `RECEIVER` | `AGREED` | `not_expired` | `record_agreed_amount` | `OTHER_PARTY` | `invitation_accepted` | yes | db |
| `INVITED` | `counter` | `RECEIVER` | `NEGOTIATING` | `negotiable`, `not_expired` | `open_counter` | `OTHER_PARTY` | `counter_sent` | no | db |
| `INVITED` | `decline` | `RECEIVER` | `DECLINED` | `not_expired` | `close_open_counter` | `OTHER_PARTY` | `invitation_declined` | yes | db |
| `INVITED` | `withdraw` | `OWNER` | `WITHDRAWN` | | `close_open_counter` | `OTHER_PARTY` | `invitation_withdrawn` | yes | db |
| `INVITED` | `expire` | `SYSTEM` | `EXPIRED` | `past_expiry` | `close_open_counter` | `BOTH` | `invitation_expired` | no | db |
| `NEGOTIATING` | `accept` | `RECEIVER` | `AGREED` | `not_expired` | `record_agreed_amount`, `close_open_counter` | `OTHER_PARTY` | `counter_accepted` | yes | db |
| `NEGOTIATING` | `counter` | `RECEIVER` | `NEGOTIATING` | `not_expired`, `under_counter_limit` | `close_open_counter`, `open_counter` | `OTHER_PARTY` | `counter_sent` | no | db |
| `NEGOTIATING` | `decline` | `RECEIVER` | `DECLINED` | `not_expired` | `close_open_counter` | `OTHER_PARTY` | `negotiation_declined` | yes | db |
| `NEGOTIATING` | `withdraw` | `OWNER` | `WITHDRAWN` | | `close_open_counter` | `OTHER_PARTY` | `invitation_withdrawn` | yes | db |
| `NEGOTIATING` | `expire` | `SYSTEM` | `EXPIRED` | `past_expiry` | `close_open_counter` | `BOTH` | `invitation_expired` | no | db |
| `AGREED` | `confirm_payment` | `PAYMENT` | `HIRED` | | `settle_siblings` | `BOTH` | `pro_hired` | no | external |

Eleven rows. Four terminal stages have none, which is the correct amount of work for a terminal
stage.

## The three things only a whole table shows you

### 1. `RECEIVER` flips, which is why the actor is never `PARTY`

In `INVITED` the standing offer is the owner's, so the receiver is the `PRO` and only the `PRO` may
accept. The moment the `PRO` counters, the standing offer is theirs, so in `NEGOTIATING` the
receiver is the `OWNER`. Same command, same row, opposite person.

Write `PARTY` instead and you have just allowed somebody to accept their own offer. That is the
single most common bug in this shape of feature, and it is invisible in any one screen. Note that
`withdraw` is always `OWNER` regardless, because withdrawing is about who created the thing, not
about whose turn it is.

### 2. The absent pairs are decisions, and each needs refusal copy

A missing `(stage, command)` is a 409, and the message comes from the stage rather than the command.
Absent on purpose here:

| Asked for | Refused because |
|---|---|
| `(AGREED, withdraw)` | Once both sides agree, withdrawal is a cancellation with money attached. Different feature, different permissions. |
| `(AGREED, expire)` | Agreed subjects never expire. This is the default in `reference.md`, and it exists because an expiry sweep that catches agreed work loses somebody a job. |
| `(HIRED, anything)` | Terminal. Changes after hiring belong to the job, not the invitation. |
| `(DECLINED, accept)` | No undo. If the product wants one, it is a new command (`reopen`) with its own row, not a relaxed guard. |

The last one is the pattern to internalise: **a new allowance is a new row, never a loosened
guard.** Loosening `when` changes the meaning of a move that already shipped.

### 3. Two rows never get an endpoint

`expire` is `SYSTEM` and `confirm_payment` is `PAYMENT`. Both are real transitions and both go
through the same procedure, so the state can only change one way. Neither mounts as
`POST /invitations/<key>/expire`, because nothing outside the system should be able to call them.

`confirm_payment` is also the only `external` row. The payment runs outside the transaction and
the command applies after it returns, so a slow provider cannot hold a row lock and a failed
charge leaves no partial state.

## In code, with the shape from `reference.md`

```python
COMMANDS: dict[tuple[Stage, str], CommandRule] = {
    (Stage.INVITED, "accept"): CommandRule(
        actor=Actor.RECEIVER,
        to=Stage.AGREED,
        when=not_expired,
        effects=("record_agreed_amount",),
        notify=Notify(Recipient.OTHER_PARTY, "invitation.accepted"),
        activity="invitation_accepted",
        confirm=True,
    ),
    (Stage.NEGOTIATING, "accept"): CommandRule(
        actor=Actor.RECEIVER,          # now the OWNER: the counter is the standing offer
        to=Stage.AGREED,
        when=not_expired,
        effects=("record_agreed_amount", "close_open_counter"),
        notify=Notify(Recipient.OTHER_PARTY, "counter.accepted"),
        activity="counter_accepted",
        confirm=True,
    ),
    (Stage.AGREED, "confirm_payment"): CommandRule(
        actor=Actor.PAYMENT,
        to=Stage.HIRED,
        effects=("settle_siblings",),
        notify=Notify(Recipient.BOTH, "pro.hired"),
        activity="pro_hired",
        io="external",
    ),
    # ...the remaining eight rows
}
```

## What the folds give you for free

**The read model**, for one subject in `NEGOTIATING` where the `PRO` countered:

```json
{ "stage": "NEGOTIATING",
  "awaiting": "OWNER",
  "allowed_actions": ["accept", "counter", "decline", "withdraw"],
  "counterparty": { "name": "Ama K.", "role": "PRO" },
  "agreed_amount": null,
  "posted_amount": "450.00",
  "counter_amount": "600.00" }
```

The same subject fetched by the `PRO` returns `"allowed_actions": []`, because it is not their turn.
No screen computes that, and no screen can get it wrong.

**The test count**, generated rather than written: 11 rows times 4 cases each (right actor
succeeds, other party 403, stranger 404, wrong stage 409) is 44, plus one 409 per absent pair, plus
one that runs `accept` twice in parallel and asserts one success and one 409. Nobody writes 44
tests by hand, which is exactly why a hand-written lifecycle has about six.

**The state diagram** is a Mermaid export of this table. If the drawing and the table disagree, the
drawing is wrong, because it was generated from an older table.
