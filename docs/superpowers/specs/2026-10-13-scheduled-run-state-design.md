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
- A new entity says what started or resumed the current job, so an automation can tell a scheduled start from an app or Home Assistant start (its form is an open decision, see §2).
- No change on a mower that already pushes its state: a bury point that arrives after the push has moved the state on writes nothing (see the ordering note in §1).

## Non-goals

- The brief *Unavailable* the reporter sees at the end of a run. The excerpt has no `AvailabilityEvent(available=False)` at all, so it needs its own log; it stays open on #120.
- Mapping trigger words to a single "scheduled" flag. The same bury point carries different words on different firmware (see below), so the sensor reports the words as they are.
- A start trigger restored across a Home Assistant restart.
- `onStats` as a mowing signal. It is reliable on this G1-800 but unconfirmed on the O1200, and the bury points are already there.

## Design

### 1. State from the task bury points (`deebot_patch/messages.py`)

**New job edges.** Four more `_OnMowJobEdge` subclasses, one per bury point, each a `NAME` and a `PHASE` like the existing eight:

| Class | `NAME` | `PHASE` |
|---|---|---|
| `OnMowScheduleResume` | `onFwBuryPoint-bd_task-mow-schedule-resume` | `resume` |
| `OnMowSchedulePause` | `onFwBuryPoint-bd_task-mow-schedule-pause` | `pause` |
| `OnMowSpotAreaResume` | `onFwBuryPoint-bd_task-mow-spotarea-resume` | `resume` |
| `OnMowSpotAreaPause` | `onFwBuryPoint-bd_task-mow-spotarea-pause` | `pause` |

All four are observed on real hardware (`MowerJobEdgeEvent`'s docstring, issue #73, and the #120 log). `MowerJobEdgeEvent.phase` widens from `"start"`/`"stop"` to those four values, and its docstring says so. The existing consumers already name the phases they act on explicitly (`sensor.py`'s `_on_job_edge` and `event.py`'s `_on_job_edge`), so a `pause` or `resume` reaches them and changes nothing there; tests pin that.

**A start or resume from the dock arms a departure; the charger confirms it.** A bury point alone is not proof that the mower left: a resume announced while the mower stays on its charger (a rain delay, a fault) would read *mowing* with nothing to correct it. The charge half of the poll publishes DOCKED only on `isCharging: 1`, a mower parked on a full battery answers `isCharging: 0` (see `GetChargeStateMower`'s docstring), and the polled `idle` is discarded (#48), so a false CLEANING would stand and the poll would never stop again. A false CLEANING also reaches every `StateEvent` consumer: it clears the fault latch (`fault.py`, CLEANING is in `_RECOVERY_STATES`), clears the activity sensor's rain flag (`EcovacsActivitySensor._on_state`), and makes the progress sensor trust a stats payload that firmware 1.36.208 does not zero between jobs (`_STATS_TRUSTED_STATES`). So the state is written in two steps, both in handlers:

1. **Arm.** In `_OnMowJobEdge._handle_body`, for `PHASE in ("start", "resume")`, on a registered bus whose `record.docked` is True: `record.arm_departure(now)`, which stores `time.monotonic()` in a new field `MowerStateRecord.departure_armed_at: float | None`. Nothing is notified yet. This runs independently of the trigger parsing below it: an edge with neither `trigger` nor `triggerType` still arms, and still returns `analyse()` for its `MowerJobEdgeEvent` as today. On a bus whose `record.docked` is False, nothing is armed: the mower is already off its charger, the poll is already running, and the state it has is the newer one (see the ordering note below).
2. **Confirm.** In `GetChargeStateMower._handle_body_data_dict`, which both the `onChargeState` push and the `getChargeState` answer go through: on `isCharging: 0`, if `record.departure_armed_at` is set and no more than `DEPARTURE_WINDOW` (60 s, a module constant in `state_precedence.py`) old, clear it, call `record.move()` and notify `StateEvent(State.CLEANING)`, then hand over to the library's handling as today. `record.dock()` clears `departure_armed_at` as well, so a resume answered by a fresh `isCharging: 1` cannot confirm later. An armed time older than the window is cleared and ignored.

In the #120 log the confirmation arrives 3 s after the edge each time (`13:00:23.762` resume → `13:00:26.501` `isCharging: 0`; `14:00:23.824` → `14:00:27.022`). `move()` is what `handle_clean_info` calls on CLEANING, so the charge gate (#67) then sees the mower as off its charger.

Accepted gap: a resume while the mower is out on the lawn (not docked) writes no state. #120 is about leaving the dock, which is the only time the poll is off; out on the lawn the poll is running already.

A `pause` or `stop` sets no state. What follows it already says what the mower does: `return-normal-start` (below) and then `onChargeState isCharging: 1`, which `GetChargeStateMower` turns into `record.dock()` and the library into DOCKED. Giving `pause` a PAUSED of its own would flash *paused* for 0.35 s before *returning*.

**The trip home.** A new handler `OnReturnNormalStart` for `onFwBuryPoint-bd_task-return-normal-start`, on the same `MessageBody` base as `_OnMowJobEdge` (the fields sit directly under `body`). It reads no field of the body. On a registered bus whose `record.docked` is False it calls `record.move()` and notifies `StateEvent(State.RETURNING)`; on a docked or unregistered bus it notifies nothing. Either way it returns `success()`, since it understood the message. It is not a job edge: it publishes no `MowerJobEdgeEvent` and is not an `_OnMowJobEdge` subclass, because a return is not a job boundary and the progress sensor must not see it as one.

**Ordering on a mower that pushes its state.** Both gates exist because a bury point can arrive after the state has moved on. The O1200's `mow-schedule-start` lands 13 s after its `onScheduleTaskInfo` CLEANING push; a pause or a rain return in those 13 s must not be overwritten. That is covered because the push already called `move()`, so the mower is no longer docked and the late edge arms nothing. When `return-normal-start` arrives relative to `onChargeInfo goCharging` and docking on a pushing mower is unobserved; a late one after docking finds `record.docked` True and writes nothing, so it cannot flip DOCKED back to RETURNING or clear the `suppressed` plan state #67 and `CleanMower` rely on.

**Why the poll then starts.** `_setup_polling`'s `on_status` subscription starts the poll on any state other than DOCKED, so the confirmed CLEANING is what turns it on. The controller's code does not change; its comment that leaving the dock "is the one transition the mower has not been seen to drop" does (see Documentation).

**A poll cannot undo it.** The polled `getCleanInfo` answer `{"state": "idle"}` is already discarded for every command family (`_MowerCleanInfoHandling`, issue #48). On a mower that pushes its state before the charge confirmation, the confirmed CLEANING equals the bus's last `StateEvent` and `EventBus.notify` drops it before any subscriber runs.

**Registration.** The five new classes are added to the tuple `apply()` registers in `deebot_patch/__init__.py`, next to the existing job edges.

### 2. The *Started by* sensor (`sensor.py`)

**⚠ OPEN DECISION — pending Rickard:** Rickard chose this sensor on 2026-10-12 (over an attribute on `lawn_mower` and new event types on `event.<device>_last_mowing_job`). Review round 1 argues it cannot carry the use case: Home Assistant writes no `state_changed` when a sensor writes the value it already has, so after the first scheduled start a trigger on *Started by* never fires again; and a condition on it read when `lawn_mower` turns *mowing* sees the previous job's word on the O1200, whose CLEANING push lands 13 s before the start edge. Options: (a) replace the sensor with an event entity *Job started* (`event_types` = the trigger words) that fires once per start or resume edge; (b) keep the sensor, set it to *unknown* on an unrecognised word, and document the trigger-on-*mowing* plus condition-on-sensor pattern with its O1200 caveat (on the G1-800 the edge is notified before the confirmed CLEANING, 3 s later, so the condition is current there); (c) both. The rest of this section describes the sensor as first designed and is rewritten once this is decided.

A new entity class, `EcovacsJobTriggerSensor`, added per mower in `async_setup_entry` next to `EcovacsActivitySensor` and `EcovacsMowingProgressSensor`, with the same gate as the activity sensor (`device.capabilities.device_type is DeviceType.MOWER`) and `device.capabilities.state` as the capability passed to `EcovacsEntity`. Like those two it listens to events no library capability describes, so it is a class of its own rather than an `ENTITY_DESCRIPTIONS` entry. It subscribes to `MowerJobEdgeEvent` and, on `phase in ("start", "resume")`, sets its value to the event's `trigger`.

- **Key and translation key:** `job_trigger`; name *Started by*.
- **Device class:** `SensorDeviceClass.ENUM`, `entity_category` none (it is meant for automations).
- **Options:** the trigger words observed on a start or resume, verbatim: `schedule`, `app`, `continue`, `reborn`. Each has a translated state name in `strings.json` and `translations/en.json` (kept identical) and an icon in `icons.json`.
- **Unknown word:** logged once at debug level and dropped, leaving the value as it was, the same pattern as `event.py`'s `_logged_triggers`. HA would reject a value outside `options`.
- **After a restart:** *unknown* until the next start or resume. No `RestoreSensor`.
- **Pause and stop:** ignored. The value is the word of the last start or resume edge, which is not always a new job: `reborn` arrives repeatedly within one run on the O800 RTK (the reason `sensor.py`'s `_NEW_JOB_TRIGGERS` excludes it), and `continue` is a resume after a charge break.

Why the words stay raw: the G1-800's `mow-schedule-resume` says `triggerType: schedule`, while the O1200's `mow-schedule-resume` in issue #73 said `trigger: continue`. A schedule continuing after a charge break and a schedule window opening are not told apart by one word across firmware, so translating both to "scheduled" would be a guess. The sensor reports what the mower said.

`MowerJobEdgeEvent._seq` already makes every edge unequal to the last, so two equal resumes both reach the sensor.

## Testing

All new behaviour is in the patch layer or in `sensor.py`. The patch-layer tests run locally (`tests/deebot_patch/`, `-p no:homeassistant`); the sensor tests are `requires_ha` and CI is the source of truth for them.

- **`tests/deebot_patch/test_messages.py`**
  - The #120 sequence replayed through the real handlers on a registered bus that starts docked, the payloads copied from the reporter's log: nothing at the 13:00 resume, CLEANING at the `onChargeState isCharging: 0` 3 s later, RETURNING at `return-normal-start`, DOCKED at `onChargeState isCharging: 1`, the same again from the 14:00 resume, RETURNING after the stop. The record's `docked` and `departure_armed_at` follow.
  - A resume on a docked bus followed by no `isCharging: 0`, or by one more than 60 s later, notifies no CLEANING; a resume followed by `isCharging: 1` and then `isCharging: 0` notifies no CLEANING (the dock cleared the arm).
  - A `getChargeState` answer with `isCharging: 0` confirms an armed departure the same way the push does.
  - A start edge with neither `trigger` nor `triggerType` still arms.
  - A start or resume on an unregistered bus (a vacuum on the same account) arms nothing and notifies no `StateEvent`.
  - Adverse orders: a resume edge after a pushed PAUSED (bus not docked) arms nothing and the state stays PAUSED; a `return-normal-start` after docking notifies nothing and leaves `record.docked` and `record.suppressed` as they were.
  - `pause` and `stop` notify no `StateEvent`.
  - `return-normal-start` notifies no `MowerJobEdgeEvent`.
  - The O1200 sequence from 2026-10-12: `onScheduleTaskInfo` CLEANING followed by `mow-schedule-start` produces exactly one CLEANING notification to subscribers.
- **`tests/deebot_patch/test_state_precedence.py`:** `arm_departure()`, the window, and `dock()` clearing the arm.
- **`tests/deebot_patch/test_contract.py`:** the five new `NAME`s are registered by `apply()`.
- **`tests/test_controller.py`:** a confirmed bury-point departure on a docked mower starts the poll, and the DOCKED at the next charge stops it.
- **`tests/test_sensor.py`:** *Started by* takes the trigger of a start and of a resume, ignores pause and stop, keeps its value on an unknown word, and the progress sensor's existing tests still pass with `pause`/`resume` edges in the stream.
- **`tests/test_event.py`:** a `pause` or `resume` edge fires no event.
- **`tests/test_translations.py` and the platform tests:** the new translation key and icon belong to a real entity, both directions.

## Documentation

- `CLAUDE.md`: the `messages.py` bullet does not mention the job edges today, so it gains a sentence: the task bury points (`mow-*-start/resume/pause/stop`) publish `MowerJobEdgeEvent`, a start or resume from the dock arms a departure that `isCharging: 0` confirms as CLEANING, and `return-normal-start` writes RETURNING. The `state_precedence.py` bullet gains the armed departure.
- `controller.py`, `on_status`: the comment "leaving is the one transition the mower has not been seen to drop" is refuted by #120; it says instead that a departure the mower does not push is written from the bury points (`messages.py`).
- `messages.py` module docstring: the list of task bury points ("`{start,stop}`", "eight more messages") grows by the four edges and `return-normal-start`.
- `MowerJobEdgeEvent` docstring: "Only the two edges this integration acts on are registered" and the `phase` description cover all four phases.
- `sensor.py` module docstring: "the two additions" counts the new entity.
- `GetChargeStateMower` docstring: it now also confirms an armed departure on `isCharging: 0`.
- README, the platform table: the `sensor` row's count goes from 16 to 17 and lists *Started by*, plus a short section on the sensor's values and the firmware difference in trigger words, with the weather-check automation from #120 as the example.
