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

- A scheduled start or resume on a mower that sends only bury points shows as *mowing*, the trip home after a pause or stop shows as *returning*, and docking shows as *docked*.
- Leaving DOCKED starts the poll, as it does for every other start.
- A new sensor says what started or resumed the current job, so an automation can tell a scheduled start from an app or Home Assistant start.
- No change on a mower that already pushes its state: the bury point's state equals the push's and the bus drops it.

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

**A start or resume means the mower is cutting.** In `_OnMowJobEdge._handle_body`, after the trigger is parsed and only on a registered bus (`record_for(event_bus) is not None`): for `PHASE in ("start", "resume")` call `record.move()` and notify `StateEvent(State.CLEANING)`. Written in the handler, before it returns, per the rule in CLAUDE.md that state derived from the stream is never written by a subscription. `move()` is what `handle_clean_info` calls on CLEANING too, so the charge gate (#67) sees the mower as off its charger.

A `pause` or `stop` sets no state. What follows it already says what the mower does: `return-normal-start` (below) and then `onChargeState isCharging: 1`, which `OnChargeState` turns into DOCKED and `record.dock()`. Giving `pause` a PAUSED of its own would flash *paused* for 0.35 s before *returning*.

**The trip home.** A new handler `OnReturnNormalStart` for `onFwBuryPoint-bd_task-return-normal-start`: on a registered bus, `record.move()` and notify `StateEvent(State.RETURNING)`. It is not a job edge: it publishes no `MowerJobEdgeEvent`, and it is not an `_OnMowJobEdge` subclass, because a return is not a job boundary and the progress sensor must not see it as one. Same `MessageBody` base as `_OnMowJobEdge`, since the fields sit directly under `body`.

**Why the poll then starts.** `_setup_polling`'s `on_status` subscription starts the poll on any state other than DOCKED, so the CLEANING above is what turns it on. No change to the controller.

**Self-correction.** If a resume is announced and the mower does not actually leave (for example a rain delay), the state reads *mowing* while it sits on the charger. The poll that CLEANING started sends `MowerStateRefresh`, whose charge half answers DOCKED within one `POLL_INTERVAL` (5 minutes), and the poll stops again.

**What cannot go wrong on a pushing mower.** The polled `getCleanInfo` answer `{"state": "idle"}` is already discarded for every command family (`_MowerCleanInfoHandling`, issue #48), so a poll cannot undo the CLEANING. On a mower that pushes `onScheduleTaskInfo` or `onCleanInfo` first, the bury point's CLEANING equals the bus's last `StateEvent` and `EventBus.notify` drops it before any subscriber runs.

**Registration.** The five new classes are added to the tuple `apply()` registers in `deebot_patch/__init__.py`, next to the existing job edges.

### 2. The *Started by* sensor (`sensor.py`)

A new entity class, `EcovacsJobTriggerSensor`, added per mower in `async_setup_entry` next to `EcovacsActivitySensor` and `EcovacsMowingProgressSensor`. Like those two it listens to events no library capability describes, so it is a class of its own rather than an `ENTITY_DESCRIPTIONS` entry. It subscribes to `MowerJobEdgeEvent` and, on `phase in ("start", "resume")`, sets its value to the event's `trigger`.

- **Key and translation key:** `job_trigger`; name *Started by*.
- **Device class:** `SensorDeviceClass.ENUM`, `entity_category` none (it is meant for automations).
- **Options:** the trigger words observed on a start or resume, verbatim: `schedule`, `app`, `continue`, `reborn`. Each has a translated state name in `strings.json` and `translations/en.json` (kept identical) and an icon in `icons.json`.
- **Unknown word:** logged once at debug level and dropped, leaving the value as it was, the same pattern as `event.py`'s `_logged_triggers`. HA would reject a value outside `options`.
- **After a restart:** *unknown* until the next start or resume. No `RestoreSensor`.
- **Pause and stop:** ignored. The value describes what started the job that is running or was last running.

Why the words stay raw: the G1-800's `mow-schedule-resume` says `triggerType: schedule`, while the O1200's `mow-schedule-resume` in issue #73 said `trigger: continue`. A schedule continuing after a charge break and a schedule window opening are not told apart by one word across firmware, so translating both to "scheduled" would be a guess. The sensor reports what the mower said.

`MowerJobEdgeEvent._seq` already makes every edge unequal to the last, so two equal resumes both reach the sensor.

## Testing

All new behaviour is in the patch layer or in `sensor.py`. The patch-layer tests run locally (`tests/deebot_patch/`, `-p no:homeassistant`); the sensor tests are `requires_ha` and CI is the source of truth for them.

- **`tests/deebot_patch/test_messages.py`**
  - The #120 sequence replayed through the real handlers on a registered bus, the bury-point payloads copied from the reporter's log: CLEANING at the 13:00 resume, RETURNING at `return-normal-start`, DOCKED at `onChargeState isCharging: 1`, CLEANING at the 14:00 resume, RETURNING after the stop. The record's `docked` flag follows.
  - A start or resume on an unregistered bus (a vacuum on the same account) notifies no `StateEvent`.
  - `pause` and `stop` notify no `StateEvent`.
  - `return-normal-start` notifies no `MowerJobEdgeEvent`.
  - The O1200 sequence from 2026-10-12: `onScheduleTaskInfo` CLEANING followed by `mow-schedule-start` produces exactly one CLEANING notification to subscribers.
- **`tests/deebot_patch/test_contract.py`:** the five new `NAME`s are registered by `apply()`.
- **`tests/test_sensor.py`:** *Started by* takes the trigger of a start and of a resume, ignores pause and stop, keeps its value on an unknown word, and the progress sensor's existing tests still pass with `pause`/`resume` edges in the stream.
- **`tests/test_event.py`:** a `pause` or `resume` edge fires no event.
- **`tests/test_translations.py` and the platform tests:** the new translation key and icon belong to a real entity, both directions.

## Documentation

- `CLAUDE.md`, the `messages.py` bullet: the job edges now include pause and resume, they write CLEANING on start and resume, and `return-normal-start` writes RETURNING.
- README, the platform table: the `sensor` row's count goes from 16 to 17 and lists *Started by*, plus a short section on the sensor's values and the firmware difference in trigger words, with the weather-check automation from #120 as the example.
