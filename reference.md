# Lifecycle Feature: shapes and decided defaults

Load this when implementing, not when deciding. Shapes only; the project profile names the real files.

## Registry row

```python
@dataclass(frozen=True)
class CommandRule:
    actor: Actor            # PRO | OWNER | PARTY | RECEIVER (turn-based) | PAYMENT | SYSTEM
    to: Stage               # target stage; the procedure maps it to storage
    when: Guard = always    # predicate over the subject, e.g. negotiable, not_expired
    effects: tuple[str, ...] = ()   # names resolved in an EFFECTS map; boot check refuses unknown names
    notify: Notify | None = None    # recipient rule (OTHER_PARTY | BOTH) + event key
    activity: str = ""              # activity kind; copy per viewer relation lives elsewhere
    confirm: bool = False           # frontend shows a confirm dialog
    io: Literal["db", "external"] = "db"   # external = run outside the transaction, then apply

COMMANDS: dict[tuple[Stage, str], CommandRule]
```

Derived: `derive_stage(subject)`, `allowed_actions_for(subject, viewer)`, `resolve_party(subject, viewer)`, `read_model(subject, viewer)`, `export_registry()`.

### Defaults decided once, reuse them

- Stage is derived from stored fields; the storage column keeps its old values. No migration for a stage.
- Turn-based moves name `RECEIVER` (of the standing offer), never `PARTY`.
- Terminal moves (decline, withdraw, expire) close any open sub-object as an effect.
- Direct accept on a negotiable subject records the posted amount as the agreed amount.
- Agreed subjects never expire on read. Expiry is INVITED and NEGOTIATING only.
- Sibling settlement on completion sits behind a setting, default off, until the stakeholder decides.
- Every status write names the auto-updating timestamp in `update_fields`.

## Procedure pipeline

```
lock(subject, of=self)            # 1
party = resolve_party(...)        # 2  none -> 404 (the key is not confirmed to a stranger)
expire lazily in its own txn      # 3  so a rolled-back refusal keeps the expiry
stage = derive_stage(subject)     # 4
rule = COMMANDS.get((stage, cmd)) # 5  none -> 409 with the stage's refusal copy
rule.check_actor(party, actor)    # 6  wrong party -> 403, before the stage check
rule.when(subject)                # 7  false -> 409
payload = SCHEMAS[cmd](data)      # 8  invalid -> 400
run effects; write status(+timestamp); refresh; emit transitioned(subject, cmd, from, to, actor, payload)
```

Person-callable commands mount as `POST /<subjects>/<key>/<command>`; PAYMENT and SYSTEM rows never mount. A refused command writes nothing.

## Folds checklist

- [ ] Read model mixin on every serializer of the subject (cached per viewer per request).
- [ ] `export_<subject>_registry` management command; generated TS file committed; snapshot JSON test.
- [ ] Parametrised HTTP tests: per row (actor ok, other 403, stranger 404, wrong stage 409) and per absent pair (409).
- [ ] Domain event has receivers for notification and activity; no post_save decides a transition.
- [ ] Mermaid export of the state diagram.

## Frontend pieces

- `shared/lib/<subject>-registry.ts` generated; `<subject>-stage.ts` with `selectStage` (server first, legacy fallback) and `resolveViewerRole`; `<subject>-stage-view.ts` with `STAGE_VIEW[stage][role]` and `COMMAND_VIEW[command]`.
- `use<Subject>Stage(subject)` bound to the signed-in user; `use<Subject>Command(subject, info)` with one mutation key per subject key, cancel-then-optimistic, one invalidation set (details, lists, activity).
- `<SubjectActions>` renders `allowed_actions` with the registry's confirm dialogs; `<StageView>` renders the copy; `<StageChip>`; one row model `subjectRowFor(subject, info, viewerRole)` feeding both parties' list items, desktop and mobile.
- Copy: labels Title Case, sentences plain, no em dashes, never defeating, role-relative ("You accepted" vs "Pro accepted").

## Guardrails

- Backend: `GR-<n> no <subject> status write outside apply_command` (grep `.status =` and `update(status=` on the model; baseline-ratcheted).
- Frontend: `W-<n> no stage/command literal outside the generated registry` (string literals matching the stage or command unions, outside `shared/lib/<subject>-*.ts`).
- Boot: registry module raises `ImproperlyConfigured` for unknown effects, missing activity kinds, missing copy, or a mounted command absent from the table.

## Deploy gates

- New beat or cron: setting default off; `manage.py <task> --quiet` one-off backfill; enable after backfill.
- Backend before frontend when the frontend reads new fields; frontend keeps the legacy fallback until the backend is confirmed live.
- Stacked PRs: base each on the previous branch; when the parent merges, retarget to the base and merge with a merge commit.
