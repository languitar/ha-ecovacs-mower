# Scheduled runs that send no state message (issue #120)

## Problem

On a GOAT G1-800 (`77atlz`, firmware 1.36.208, integration 0.10.0) a run started by the Ecovacs app's schedule stays **Docked** in Home Assistant for its whole length. The reporter's log covers two schedule windows (13:00–13:30 and 14:00–15:30) of a job continued from the day before.

The mower sends no state message at all in those two hours: no `onCleanInfo_V2`, no `onScheduleTaskInfo`, and no `getCleanInfo` is ever asked. What it does send is task bury points and `onStats`:

```
13:00:23.762  onFwBuryPoint-bd_task-mow-schedule-resume   triggerType: schedule
13:00:26.410  onFwBuryPoint-bd_task-charge-stop
13:00:26.501  onChargeState                               isCharging: 0
13:30:23.452  onFwBuryPoint-bd_task-mow-schedule-pause    cuttedArea: 199.29
13:30:23.801  onFwBuryPoint-bd_task-return-normal-start   triggerType: schedule
13:31:35.849  onFwBuryPoint-bd_task-charge-start
13:31:35.850  onChargeState                               isCharging: 1   -> DOCKED (dropped as a repeat)
13:31:38.394  onFwBuryPoint-bd_task-return-normal-stop    triggerType: workcomplete
14:00:23.824  onFwBuryPoint-bd_task-mow-schedule-resume   triggerType: schedule
14:00:27.022  onChargeState                               isCharging: 0
15:01:59.695  onFwBuryPoint-bd_task-mow-schedule-stop     cuttedArea: 280.20
15:01:59.802  onFwBuryPoint-bd_task-return-normal-start   triggerType: schedule
```

`onStats` arrives about every two seconds inside both windows and not between them.

Two things combine:

1. **Nothing moves the state off DOCKED.** `mow-schedule-resume` and `-pause` are not registered (`Unknown message`). `mow-schedule-start` and `-stop` are, but `_OnMowJobEdge` only publishes `MowerJobEdgeEvent`, which feeds the progress sensor and the last-job event entity, never `StateEvent`. `return-normal-start` is not registered.
2. **The poll is off while docked.** `EcovacsController._setup_polling` stops polling on DOCKED, on the premise that leaving the dock is the one transition the mower has not been seen to drop. On this firmware a scheduled resume is exactly that transition, dropped.

The O1200 LiDAR Pro (`2i0fns`, firmware 1.13.10) is not affected: on 2026-10-12 a scheduled edge job pushed `onScheduleTaskInfo` with `state: clean` 13 seconds before its `mow-schedule-start` bury point, and Home Assistant went to *mowing* on that push.

The reporter's use case is to keep the schedule in the app and let Home Assistant add a weather check when a scheduled run starts. That needs the state to change and a way to tell a scheduled start from another one.

## Goals

- A scheduled start or resume on a mower that sends only bury points shows as *mowing* once the mower has left its charger, never on the announcement alone; the trip home after a pause or stop shows as *returning*, and docking shows as *docked*.
- Leaving DOCKED starts the poll, as it does for every other start.
- A new entity says what started or resumed the current job and which job it is, so an automation can tell the schedule's job from any other (the *Job started* event entity, §2).
- No change on a mower that already pushes its state: a bury point that arrives after the push has moved the state on writes nothing (see the ordering note in §1).

## Non-goals

- The brief *Unavailable* the reporter sees at the end of a run. The excerpt has no `AvailabilityEvent(available=False)` at all, so it needs its own log; it stays open on #120.
- Mapping trigger words to a single "scheduled" flag. The same bury point carries different words on different firmware (see §2), so the entity reports the words as they are and carries the job type beside them.
- `onStats` as a mowing signal. It is reliable on this G1-800 but unconfirmed on the O1200, and the bury points are already there.

## Design

### 1. State from the task bury points (`deebot_patch/messages.py`, `deebot_patch/state_precedence.py`)

**New job edges.** Four more `_OnMowJobEdge` subclasses, one per bury point, each a `NAME`, a `PHASE` and a `JOB_TYPE` like the existing eight:

| Class | `NAME` | `PHASE` | `JOB_TYPE` |
|---|---|---|---|
| `OnMowScheduleResume` | `onFwBuryPoint-bd_task-mow-schedule-resume` | `resume` | `schedule` |
| `OnMowSchedulePause` | `onFwBuryPoint-bd_task-mow-schedule-pause` | `pause` | `schedule` |
| `OnMowSpotAreaResume` | `onFwBuryPoint-bd_task-mow-spotarea-resume` | `resume` | `spotarea` |
| `OnMowSpotAreaPause` | `onFwBuryPoint-bd_task-mow-spotarea-pause` | `pause` | `spotarea` |

All four are observed on real hardware (`MowerJobEdgeEvent`'s docstring, issue #73, the #120 log, and Geten's log of 2026-10-12). `MowerJobEdgeEvent.phase` widens from `"start"`/`"stop"` to those four values, and its docstring says so. The existing consumers already name the phases they act on explicitly (`sensor.py`'s `_on_job_edge` and `event.py`'s `_on_job_edge`), so a `pause` or `resume` reaches them and changes nothing there; tests pin that.

**The job type travels with the edge.** `JOB_TYPE` is a new `ClassVar[str]` on `_OnMowJobEdge`, set on all twelve subclasses to the bury point's middle segment (`schedule`, `spotarea`, `border`, `auto`), and `MowerJobEdgeEvent` gains a field `job_type: str` that the handler fills from it. The middle segment is the job type, not the trigger (`MowerJobEdgeEvent`'s docstring), and it is the one signal that says "this is the schedule's job" the same way on every firmware: the O1200's `trigger: continue` and the G1-800's `triggerType: schedule` both arrive on `mow-schedule-resume`. It is not the `cleanState.content.type` that `MowerStateRecord.job_type` holds for the mow command (`spotArea` there, `spotarea` here); the two are not merged.

**A start or resume from the dock is written only once the charger confirms the departure.** A bury point alone is not proof that the mower left: a resume announced while the mower stays on its charger (a rain delay, a fault) would read *mowing* with nothing to correct it. The charge half of the poll publishes DOCKED only on `isCharging: 1`, and the polled `idle` is discarded (#48), so a false CLEANING on a mower answering `isCharging: 0` would stand and the poll would never stop again. A false CLEANING also reaches every `StateEvent` consumer: it clears the fault latch (`fault.py`, CLEANING is in `_RECOVERY_STATES`), clears the activity sensor's rain flag (`EcovacsActivitySensor._on_state`), and makes the progress sensor trust a stats payload that firmware 1.36.208 does not zero between jobs (`_STATS_TRUSTED_STATES`).

So the CLEANING needs two facts that arrive within `DEPARTURE_WINDOW` (60 s, a module constant in `state_precedence.py`) of each other, **in either order**: the start or resume edge, and the mower's own `onChargeState` push saying it stopped charging. Observed on hardware:

| Mower | Departure | First | Second |
|---|---|---|---|
| G1-800 `77atlz` (#120) | schedule window opens, 13:00 and 14:00 | resume edge | `isCharging: 0`, 3 s later |
| O1200 `2i0fns` (2026-10-12) | scheduled start after a night on the dock, battery 100 | start edge, 09:01:06 | `isCharging: 0`, 7 s later |
| O1200 `2i0fns` (2026-10-12) | app start, battery 100 | start edge, 18:14:15 | `isCharging: 0`, 7 s later |
| O1200 `2i0fns` (2026-10-12) | schedule resume after a charge break, 11:47 and 14:29 | `isCharging: 0` | resume edge, 8 s later |

The two O1200 starts at battery 100 also show that a mower parked on a full battery still reports itself charging until it leaves, so a full battery does not take the confirmation away. `GetChargeStateMower`'s docstring says a mower parked on a full battery stops charging without moving; neither log shows that, and the design does not rely on it either way.

The record keeps both facts in two new fields of `MowerStateRecord`, `departure_edge_at: float | None` and `charge_left_at: float | None` (monotonic seconds), through two new methods that take the clock as an argument so tests can inject it:

- `note_departure_edge(now: float) -> bool` stores `now` as `departure_edge_at`. If `charge_left_at` is set and `now - charge_left_at <= DEPARTURE_WINDOW`, it clears both and returns True; otherwise it returns False. A stamp older than the window is cleared, not kept.
- `note_charge_left(now: float) -> bool` does the same with the roles swapped.
- `dock()` and `move()` clear both fields. A dock means a pending departure did not happen; a pushed state that moved the mower is better evidence than either stamp, and a stale stamp must not re-assert CLEANING over a PAUSED pushed in between.

The two callers, both in handlers, per the rule in CLAUDE.md that state derived from the stream is never written by a subscription:

1. **The edge.** In `_OnMowJobEdge._handle_body`, for `PHASE in ("start", "resume")`, on a registered bus whose mower may be on its charger, call `record.note_departure_edge(time.monotonic())`. "May be on its charger" means `record.docked` is True, **or** no `StateEvent` has been published on the bus yet (`event_bus.get_last_event(StateEvent) is None`): after a restart the record starts undocked, and a mower that has said nothing since must still be able to arm. Arming while that is unknown is safe because only a real push can complete the pair. If the call returns True, call `record.move()` and notify `StateEvent(State.CLEANING)`. This runs before and independently of the trigger parsing: an edge with neither `trigger` nor `triggerType` still counts, and still returns `analyse()` for its `MowerJobEdgeEvent` as today.
2. **The push.** In `OnChargeState._handle_body`, before it forwards to `GetChargeStateMower._handle_body`: if the body's `data.isCharging` is `0` and the bus is registered, call `record.note_charge_left(time.monotonic())`; if it returns True, call `record.move()` and notify `StateEvent(State.CLEANING)`; then forward as today. Only the push counts. A `getChargeState` answer is a snapshot, not a change, and it is sent while the mower is docked by `update_entity`, by the refresh after an availability flap and by the first subscriber; letting it pair would bring the false CLEANING back. `GetChargeStateMower` therefore does not change.

`move()` is what `handle_clean_info` calls on CLEANING, so the charge gate (#67) then sees the mower as off its charger.

A `pause` or `stop` sets no state. What follows it already says what the mower does: `return-normal-start` (below) and then `onChargeState isCharging: 1`, which `GetChargeStateMower` turns into `record.dock()` and the library into DOCKED. Giving `pause` a PAUSED of its own would flash *paused* for 0.35 s before *returning*.

**The trip home.** A new handler `OnReturnNormalStart` for `onFwBuryPoint-bd_task-return-normal-start`, on the same `MessageBody` base as `_OnMowJobEdge` (the fields sit directly under `body`). It reads no field of the body. On a registered bus whose `record.docked` is False it calls `record.move()` and notifies `StateEvent(State.RETURNING)`; on a docked or unregistered bus it notifies nothing. Either way it returns `success()`, since it understood the message. It is not a job edge: it publishes no `MowerJobEdgeEvent` and is not an `_OnMowJobEdge` subclass, because a return is not a job boundary and the progress sensor must not see it as one.

**Ordering on a mower that pushes its state.** Both gates exist because a bury point can arrive after the state has moved on. The O1200's `mow-schedule-start` lands 13 s after its `onScheduleTaskInfo` CLEANING push; a pause or a rain return in those 13 s must not be overwritten. That is covered because the push already called `move()`: the mower is no longer docked, a `StateEvent` exists, and the late edge does not count. When `return-normal-start` arrives relative to `onChargeInfo goCharging` and docking on a pushing mower is unobserved; a late one after docking finds `record.docked` True and writes nothing, so it cannot flip DOCKED back to RETURNING or clear the `suppressed` plan state #67 and `CleanMower` rely on.

**Accepted gaps.** (1) A resume while the mower is out on the lawn writes no state: #120 is about leaving the dock, the only time the poll is off. (2) Only the observed pause and resume edges are registered (`schedule`, `spotarea`), so a charge-break resume of an `auto` or `border` job on a mower that sends only bury points still reads *docked* until a push or the next start; registering those is one subclass each once they are seen. (3) A `pause` out on the lawn that is not followed by `return-normal-start` leaves CLEANING standing until the next push or docking. (4) The O1200's `mow-border-rotate-*` edges are not registered here; they belong to #127.

**Why the poll then starts.** `_setup_polling`'s `on_status` subscription starts the poll on any state other than DOCKED, so the paired CLEANING is what turns it on. The controller's code does not change; its comment that leaving the dock "is the one transition the mower has not been seen to drop" does (see Documentation).

**A poll cannot undo it.** The polled `getCleanInfo` answer `{"state": "idle"}` is already discarded for every command family (`_MowerCleanInfoHandling`, issue #48). On a mower that pushes its state before the pair completes, `move()` has already cleared the stamps and nothing is written; if the pair completes first, the push's CLEANING equals the bus's last `StateEvent` and `EventBus.notify` drops it before any subscriber runs.

**Registration.** The five new classes are added to the tuple `apply()` registers in `deebot_patch/__init__.py`, next to the existing job edges. `OnChargeState` is already registered.

### 2. The *Job started* event entity (`event.py`)

Decided by Rickard on 2026-10-13, after review round 1: an event entity, not a sensor. Rickard first chose a *Started by* enum sensor (2026-10-12); the review showed it cannot carry the use case, because Home Assistant writes no `state_changed` when a sensor writes the value it already has, so after the first scheduled start a trigger on it never fires again, and a condition on it read when `lawn_mower` turns *mowing* sees the previous job's word on the O1200, whose CLEANING push lands 13 s before the start edge. An event entity fires once per edge, whatever the previous one was.

A new entity class, `EcovacsJobStartedEventEntity`, in `event.py` next to `EcovacsLastJobEventEntity`, added in `async_setup_entry` for every device whose `device.capabilities.device_type is DeviceType.MOWER` (the gate `EcovacsActivitySensor` uses), with `device.capabilities.state` as the capability passed to `EcovacsEntity`. It subscribes to `MowerJobEdgeEvent` and, on `phase in ("start", "resume")`, calls `_trigger_event(event.trigger, {"phase": event.phase, "job_type": event.job_type})` and `async_write_ha_state()`.

- **Description:** `EventEntityDescription(key="job_started", translation_key="job_started", event_types=["schedule", "app", "continue", "reborn"])`, no `entity_category` (it is meant for automations).
- **Event types:** the trigger words observed on a start or resume, verbatim. Translations in `strings.json` and `translations/en.json` (kept identical): the entity name *Job started* under `entity.event.job_started.name`, and one state name per type under `entity.event.job_started.state_attributes.event_type.state`, proposed *Schedule*, *App*, *Continued*, *Restarted*. One `default` icon for the entity in `icons.json` (`mdi:play-circle-outline`), the same shape as `last_job`.
- **Event data:** `phase` (`"start"` or `"resume"`), so an automation can tell a new job from one continuing, and `job_type` (`schedule`, `spotarea`, `border`, `auto`), so it can tell the schedule's job from any other whatever the trigger word.
- **Unknown word:** logged once at debug level and dropped, the same pattern as `EcovacsLastJobEventEntity._logged_triggers`. `EventEntity._trigger_event` raises on a type outside `event_types`.
- **Replay guard:** the edge already fired is held by identity (`self._reported_start`), exactly as `_reported_ending` does in `EcovacsLastJobEventEntity` and for the same reason: `EventBus.subscribe` replays the last event to a new subscriber, and an entity-ID change re-adds the entity. `MowerJobEdgeEvent._seq` makes two genuine, equal-looking edges distinct objects.
- **Pause and stop:** ignored; the stop already feeds *Last mowing job*.
- **A start from Home Assistant:** its trigger word is unobserved (`mow-auto-start` is inferred, not captured), so it may fire no event, only a debug line, until the word is seen and added.

What an event means: the mower announced that something started or resumed it. It is not always a new job: `reborn` arrives repeatedly within one run on the O800 RTK (the reason `sensor.py`'s `_NEW_JOB_TRIGGERS` excludes it), and `continue` is a resume after a charge break. The event fires at the announcement, before the departure is confirmed (§1): on the G1-800 it precedes the CLEANING by about 3 s, on the O1200 it follows the push by 13 s. An automation that must act only once the mower is out conditions on `lawn_mower` being *mowing*.

Why the words stay raw: the G1-800's `mow-schedule-resume` says `triggerType: schedule`, while the O1200's says `trigger: continue` (Geten's log of 2026-10-12, and issue #73). A schedule continuing after a charge break and a schedule window opening are not told apart by one word across firmware, so translating both to "scheduled" would be a guess. The entity reports what the mower said, and `job_type` answers "is this the schedule's job"; the README shows the #120 weather check as an automation triggered on the entity with the condition `job_type == "schedule"`.

## Testing

All new behaviour is in the patch layer or in `event.py`. The patch-layer tests run locally (`tests/deebot_patch/`, `-p no:homeassistant`); the entity and controller tests are `requires_ha` and CI is the source of truth for them.

- **`tests/deebot_patch/test_messages.py`**
  - The #120 sequence replayed through the real handlers on a registered bus that starts docked, the payloads copied from the reporter's log: nothing at the 13:00 resume, CLEANING at the `onChargeState isCharging: 0` 3 s later, RETURNING at `return-normal-start`, DOCKED at `onChargeState isCharging: 1`, the same again from the 14:00 resume, RETURNING after the stop.
  - The O1200 charge-break order: `onChargeState isCharging: 0` first, the resume edge 8 s later, CLEANING at the edge.
  - Neither order pairs across more than 60 s; a dock between the two facts cancels the pairing; a pushed CLEANING (`move()`) between them cancels it too, so a later fact does not re-assert CLEANING over a PAUSED.
  - A `getChargeState` answer with `isCharging: 0` does **not** complete a pair.
  - An edge on a bus with no `StateEvent` yet and `record.docked` False counts; an edge on a bus that is not docked and has a `StateEvent` does not.
  - A start edge with neither `trigger` nor `triggerType` still counts.
  - A start or resume, or a `return-normal-start`, on an unregistered bus (a vacuum on the same account) notifies no `StateEvent`; `return-normal-start` returns success there.
  - Adverse orders: a resume edge after a pushed PAUSED (bus not docked) does not count and the state stays PAUSED; a `return-normal-start` after docking notifies nothing and leaves `record.docked` and `record.suppressed` as they were.
  - `pause` and `stop` notify no `StateEvent`; `return-normal-start` notifies no `MowerJobEdgeEvent`.
  - Every edge carries its class's `job_type`.
  - The O1200 sequence from 2026-10-12: `onScheduleTaskInfo` CLEANING, then `mow-schedule-start`, then `onChargeState isCharging: 0`, produces exactly one CLEANING notification to subscribers.
  - Registration: the five new names are registered by `apply()`, next to `test_the_schedule_and_spotarea_job_edge_names_are_registered`.
- **`tests/deebot_patch/test_state_precedence.py`:** `note_departure_edge()` and `note_charge_left()` in both orders, at and past the window, and `dock()` and `move()` clearing both stamps.
- **`tests/test_controller.py`:** a paired departure on a docked mower starts the poll, and the DOCKED at the next charge stops it.
- **`tests/test_event.py`:** *Job started* fires the trigger word with `phase` and `job_type` on a start and on a resume, fires nothing on pause and stop or on an unknown word (logged once), does not re-fire on a subscribe replay of the same edge object, does fire for a second equal-looking edge, and exists only on a MOWER device. *Last mowing job* fires nothing on a `pause` or `resume` edge. `test_no_stale_event_translations_or_icons` widens from `last_job` to both keys.
- **`tests/test_sensor.py`:** the progress sensor ignores `pause` and `resume` edges.
- **`tests/test_translations.py` and the platform tests:** the new translation keys and icon belong to a real entity, both directions.

## Documentation

- `CLAUDE.md`: the `messages.py` bullet does not mention the job edges today, so it gains a sentence: the task bury points (`mow-*-start/resume/pause/stop`) publish `MowerJobEdgeEvent` with their job type, a start or resume edge paired with an `onChargeState isCharging: 0` push within 60 s, in either order, writes CLEANING, and `return-normal-start` writes RETURNING. The `state_precedence.py` bullet gains the two departure stamps.
- `controller.py`, `on_status`: the comment "leaving is the one transition the mower has not been seen to drop" is refuted by #120; it says instead that a departure the mower does not push is written from the bury points (`messages.py`).
- `messages.py` module docstring: the opening paragraph ("Without the latter two the entity never leaves 'docked' during a scheduled run") gains the task bury points as a third way out of DOCKED, and the list of task bury points ("`{start,stop}`", "eight more messages") grows by the four edges and `return-normal-start`.
- `MowerJobEdgeEvent` docstring: "Only the two edges this integration acts on are registered", the `phase` description and the new `job_type` field.
- `OnChargeState` docstring: the push now also pairs with a start or resume edge. `GetChargeStateMower` docstring: its full-battery sentence gains that the two O1200 logs of 2026-10-12 show a mower at battery 100 reporting itself charging until it leaves.
- `event.py` module docstring: the entity gains a sibling, *Job started*.
- README, the platform table: the `event` row's count goes from 1 to 2 and lists *Job started*, plus a short section on its event types and event data, the firmware difference in trigger words, and the weather-check automation from #120 as the example.
