# Copilot 2026-05-24

The data `lstosa` processes will suffer a modification soon. Gain reduction will already be performed and Cat-A calibration already performed. How should we modify the program to be able to cope with this?

A good way to handle this is to **separate “what state the input data is already in” from “which steps `lstosa` should run”** and make that explicit in configuration, rather than continuing to add skip flags.

## Recommendation

Introduce a **processing-state / processing-mode model** for the input, for example:

- `legacy_raw` — needs gain reduction + Cat-A calibration
- `gain_reduced` — skip gain reduction, still do Cat-A calibration if needed
- `catA_calibrated` — skip gain reduction and Cat-A calibration
- `custom` — explicit per-step toggles

Then make the sequencer derive its job graph from that state.

## Why this is better than more `--no-*` flags

Right now, options like “no calibration” and “no gain selection” are useful, but they are **negative skip-flags**, which become hard to reason about as upstream formats change. If the new data format bakes in some steps, a positive “input state” model is clearer and safer.

That gives you:

- better readability,
- fewer invalid option combinations,
- easier future extension,
- safer defaults,
- simpler testing.

## Concrete design

Add a config section like:

```ini
[INPUT_STATE]
mode = legacy_raw
# allowed: legacy_raw, gain_reduced, catA_calibrated, custom

[PROCESSING]
do_gain_selection = auto
do_catA_calibration = auto
do_dl1ab = auto
do_dl2 = auto
```

Or, more simply:

```ini
[INPUT_STATE]
gain_reduced = false
catA_calibrated = false
```

I’d prefer **mode + optional overrides**:

```ini
[INPUT_STATE]
mode = catA_calibrated

[OVERRIDES]
do_dl1ab = true
do_dl2 = true
```

## Behavior

The sequencer should translate the mode into a normalized internal execution plan, e.g.:

- `legacy_raw`
  - run gain-related preprocessing if required
  - run Cat-A calibration
  - run downstream steps

- `gain_reduced`
  - skip gain-reduction-dependent preprocessing
  - run Cat-A calibration
  - run downstream steps

- `catA_calibrated`
  - skip gain reduction
  - skip Cat-A calibration
  - start from the first downstream compatible step

## Implementation approach

Refactor around a small internal object, something like:

```python
from dataclasses import dataclass

@dataclass
class ProcessingPlan:
    needs_gain_selection: bool
    needs_catA_calibration: bool
    needs_dl1ab: bool
    needs_dl2: bool
    input_level: str
```

Then create one function that builds this plan from config/CLI:

```python
plan = build_processing_plan(config, cli_args, detected_inputs)
```

And make job creation depend on `plan`, not directly on many unrelated global flags.

## Flexible policy hierarchy

Use this precedence:

1. **Explicit CLI override**
2. **Config file**
3. **Auto-detection from files/metadata**
4. **Backward-compatible default**

That way:

- operations staff can force behavior,
- normal runs can use config defaults,
- future formats can be auto-detected when safe,
- legacy behavior remains unchanged.

## Auto-detection

Auto-detection can help, but it should be **advisory**, not the only mechanism.

For example, if the input files contain metadata indicating:

- gain reduction already applied,
- Cat-A calibration already applied,

then `lstosa` can:

- infer a proposed mode,
- log it,
- optionally require confirmation or allow override.

Best practice:

- **config chooses the mode**
- **auto-detection validates and warns on mismatch**

Example:

- config says `legacy_raw`
- metadata says `catA_calibrated`
- pipeline logs warning or errors, depending on strictness setting

## Suggested strictness option

Add something like:

```ini
[INPUT_VALIDATION]
mode = warn
# warn | strict | off
```

This avoids silent misuse when upstream data changes unexpectedly.

## Minimal code changes strategy

The least disruptive approach is:

1. **Add a new abstraction for input state / processing plan**
2. **Map existing flags into that abstraction**
3. **Keep old flags working**
4. **Gradually move step-selection logic to the plan**

So instead of deleting old options immediately:

- `--no-calib` becomes an override on the plan
- `--no-gainsel` becomes an override on the plan
- later, you can deprecate them in favor of `--input-mode`

## Backward compatibility

Keep current behavior as default:

- if no new config is provided, behave exactly as today
- `legacy_raw` should be the default mode

Then add:

- `--input-mode legacy_raw|gain_reduced|catA_calibrated|custom`

This lets operators adopt the new mode gradually.

## Testing strategy

You’ll want tests for at least these cases:

1. `legacy_raw`
   - calibration jobs included
   - downstream dependencies unchanged

2. `gain_reduced`
   - gain-reduction-related work skipped
   - Cat-A calibration still runs

3. `catA_calibrated`
   - Cat-A calibration skipped
   - downstream jobs still generated correctly

4. CLI override beats config

5. auto-detection mismatch
   - warning or failure depending on validation mode

6. backward compatibility
   - old configs and old CLI flags still produce current behavior

## High-risk areas

The main risks are:

- places where step ordering is implied rather than explicit
- hidden assumptions that calibration artifacts always exist locally
- dependency wiring between calibration and downstream jobs
- provenance/history reporting that assumes skipped steps ran inside `lstosa`
- file naming / discovery logic that infers state from legacy products

## Best overall design choice

If you want the most flexible long-term solution, I recommend:

- **explicit input mode**
- **internal processing plan**
- **optional auto-detection for validation**
- **backward-compatible old flags as overrides**

In short:

> Don’t model this as “skip more steps.”
> Model it as “`lstosa` can start from different upstream processing states.”

If you want, I can next draft a **concrete refactoring plan** with proposed config keys, CLI syntax, and the Python interfaces to add.

## Follow-up: should gain-selection be connected to the sequencer?

Right now, gain-selection is a separate step not linked to the sequencer. Should they be connected?

Yes — **conceptually they should be connected, but not necessarily merged into one script**.

Right now the repo treats gain selection as an **external upstream stage** and the sequencer as a **downstream consumer**:

- The launchers/cron run gain selection separately.
- The launcher docs explicitly say Sequencer 1 runs only when `GainSelFinished.txt` exists.
- The crontab also shows gain selection and sequencer as distinct scheduled stages.

So today they are **operationally connected by cron/flag files**, not by a shared internal pipeline model.

## What should change?

**They should be connected at the orchestration/model level, not tightly coupled at the implementation level.**

### Good target architecture

Keep:

- `gain_selection` as its own executable/module
- `sequencer` as its own executable/module

But add a **shared processing-state contract** between them.

For example:

- `input_mode = legacy_raw`
- `input_mode = gain_selected`
- `input_mode = catA_calibrated`
- `input_mode = fully_precalibrated`

Then sequencer decides:

- whether it must wait for gain selection,
- whether it should launch jobs assuming gain-selected input,
- whether Cat-A calibration is needed,
- what file locations/products to expect.

## Why not fully merge them?

Because gain selection is still a distinct operational activity:

- separate schedule,
- separate logs/history,
- separate monitoring web page,
- separate retry logic.

So I would **not** fold gain selection logic into the sequencer job-submission code. That would make the sequencer too responsible for upstream mechanics.

## What to connect, exactly

You should connect them through three things:

### 1. Shared data-state model

Introduce a common abstraction like:

```python
from enum import Enum

class InputState(Enum):
    LEGACY_RAW = "legacy_raw"
    GAIN_SELECTED = "gain_selected"
    CATA_CALIBRATED = "catA_calibrated"
```

Then sequencer no longer asks:

- “did gain selection finish?” as a special case

It asks:

- “is the required input state for this stage available?”

### 2. Shared readiness check

Replace hard-coded special cases like `--no-gainsel` / `GainSelFinished.txt` gating with a general readiness API:

- `is_input_state_available(date, state)`
- `get_available_input_state(date)`
- `validate_input_state(required, available)`

Then the launchers can still exist, but they become thinner and more generic.

### 3. Shared configuration

The config should say what upstream guarantees are expected for this deployment:

- old mode: gain selection must happen before sequencer
- new mode: gain selection already done outside `lstosa`
- future mode: gain selection absent entirely because upstream writes post-gain-selected data

## Direct answer to the question

**Yes, they should be connected logically. No, they do not need to be physically combined into one program.**

That is the flexible design.

---

## What this means for the upcoming data change

If future data arrives already:

- gain-reduced
- Cat-A calibrated

then sequencer should be able to start from a later state **without caring whether gain selection was run by `lstosa`, by another service, or offline upstream**.

That means sequencer should stop using “gain selection finished” as a pipeline truth, and instead use “required input products/state are available”.

## Practical migration path

I’d suggest:

### Phase 1

Keep current scripts, but introduce config like:

```ini
[INPUT]
state = legacy_raw
# legacy_raw | gain_selected | catA_calibrated

[ORCHESTRATION]
require_internal_gainsel = auto
```

### Phase 2

Refactor sequencer launcher conditions:

- today: wait for `GainSelFinished.txt`
- future: call a shared “state ready?” check

### Phase 3

Deprecate `--no-gainsel` as a special-purpose flag and replace it with:

- `--input-state gain_selected`
- or `--input-state catA_calibrated`

## One important current smell

The current `--no-gainsel` help text suggests the sequencer is not really modeling input state; it is just skipping a check. That’s exactly what you want to evolve away from.

## Recommendation

Implement this rule:

> Sequencer should depend on declared/validated input state, not on the internal existence of a separate gain-selection step.

So:

- **connect the concepts**
- **do not tightly couple the executables**
- **replace step-specific flags with input-state semantics**

If you want, I can next outline a **concrete refactor in `lstosa` terms**: which modules to add/change, what config keys to introduce, and how to migrate the existing `GainSelFinished.txt` logic cleanly.

## Concrete refactor outline

Here’s a concrete refactor outline for `cta-observatory/lstosa` that keeps gain selection separate but makes the pipeline flexible enough for future input states.

### Goal

Make the pipeline start from different **declared input states** instead of hard-coding assumptions like:

- gain selection must have run,
- Cat-A calibration must be produced here,
- special flags are needed to skip checks.

### Design principle

Keep these as separate concerns:

1. **Upstream stages**
   gain selection, Cat-A calibration, upstream preprocessing
2. **Pipeline state model**
   what state the data is in
3. **Sequencer execution plan**
   what `lstosa` should run next

The sequencer should consume a **state description**, not infer behavior from legacy flags alone.

### 1. Add a new shared state module

Create a module like:

```python
from dataclasses import dataclass
from enum import Enum

class InputState(str, Enum):
    LEGACY_RAW = "legacy_raw"
    GAIN_SELECTED = "gain_selected"
    CATA_CALIBRATED = "catA_calibrated"

@dataclass(frozen=True)
class ProcessingPlan:
    input_state: InputState
    require_gain_selection_check: bool
    run_catA_calibration: bool
    run_dl1: bool
    run_catB_tailcuts: bool
    run_dl1ab: bool
    run_dl2: bool
```

This becomes the central abstraction.

### 2. Add plan-building logic

Create:

```python
from osa.pipeline_state import InputState, ProcessingPlan

def build_processing_plan(options, cfg) -> ProcessingPlan:
    ...
```

Suggested mapping:

- `legacy_raw`
  - require gain-selection readiness
  - run Cat-A calibration
  - run downstream steps
- `gain_selected`
  - do not require gain-selection step inside `lstosa`
  - still run Cat-A calibration
  - run downstream steps
- `catA_calibrated`
  - do not require gain-selection readiness
  - do not run Cat-A calibration
  - start from the next compatible stage

This is where current flags should be translated too:

- `--no-gainsel` → override `require_gain_selection_check = False`
- `--no-calib` → override `run_catA_calibration = False`
- `--no-dl1ab` → override `run_dl1ab = False`

That lets you preserve backward compatibility.

### 3. Add config keys for input state

Today config is mostly procedural. Add declarative config, e.g.:

```ini
[INPUT]
state = legacy_raw
# allowed:
# legacy_raw
# gain_selected
# catA_calibrated

[VALIDATION]
strict_input_state = true
```

Optional future extension:

```ini
[INPUT]
state = auto

[OVERRIDES]
run_catA_calibration = true
```

But for now I’d keep it simple and explicit.

### 4. Extend CLI in `cliopts.py`

Current sequencer CLI has step-skipping flags in `src/osa/utils/cliopts.py`. Add:

```python
parser.add_argument(
    "--input-state",
    choices=["legacy_raw", "gain_selected", "catA_calibrated"],
    default=None,
    help="Declared preprocessing state of the input data",
)
```

Then in `sequencer_cli_parsing()`:

```python
options.input_state = opts.input_state
```

Also add this field to shared options:

```python
input_state = None
```

#### Backward compatibility rule

Precedence should be:

1. CLI `--input-state`
2. config `[INPUT] state`
3. default `legacy_raw`

Old flags still work and override plan pieces.

### 5. Replace “GainSel finished?” checks with state readiness checks

Right now the repo is launcher/cron driven, and gain selection is gated by `GainSelFinished.txt`.

Instead of letting the launcher directly encode pipeline semantics, centralize it in a function like:

```python
from osa.pipeline_state import InputState

def is_required_input_ready(date, input_state, cfg) -> tuple[bool, str]:
    """
    Return (ready, reason).
    """
    ...
```

Example logic:

- `legacy_raw`
  - require `GainSelFinished.txt` or equivalent gain-selected products
- `gain_selected`
  - require presence of gain-selected input products
- `catA_calibrated`
  - require presence of Cat-A-calibrated products

This avoids tying the system forever to the existence of a gain-selection script.

### 6. Keep launchers, but make them thin

Do **not** remove `gain_selection` as a separate tool yet.

Instead, change launcher logic from:

- “run sequencer if `GainSelFinished.txt` exists”

to:

- “run sequencer if input state is ready”

### 7. Make sequencer build only the needed job graph

The important refactor is not just submission-time checks. It’s also sequence construction.

Wherever the sequencer currently:

- assumes calibration sequence exists,
- assumes data sequences depend on calibration,
- assumes gain selection was external,

change that logic to depend on `ProcessingPlan`.

Pseudo-logic:

```python
plan = build_processing_plan(options, cfg)

if plan.run_catA_calibration:
    include_pedcalib_sequence()

if plan.run_dl1:
    include_data_sequences()

if plan.run_catB_tailcuts:
    include_catB_sequences()

if plan.run_dl1ab:
    include_dl1ab_sequences()

if plan.run_dl2:
    include_dl2_sequences()
```

In submission logic, dependencies should come from the plan, not from scattered flags.

### 8. Refactor current flags into plan overrides

Current flags:

- `--no-gainsel`
- `--no-calib`
- `--no-dl1ab`

Keep them, but interpret them as plan modifiers.

That lets old operational workflows keep running during migration.

### 9. Add input validation

You don’t want mismatches between declared state and actual files.

Add:

```python
def validate_input_state(date, input_state, cfg) -> list[str]:
    ...
```

Examples:

- declared `catA_calibrated`, but expected calibrated files are missing
- declared `gain_selected`, but only raw R0 files exist
- declared `legacy_raw`, but upstream-calibrated files already exist

Then:

- in strict mode: abort
- in warn mode: log warning

Suggested config:

```ini
[VALIDATION]
mode = warn
# warn | strict | off
```

### 10. Introduce a “capabilities” view for future extensibility

If you expect more upstream changes, a simple enum may later become too coarse.

A future-proof version is:

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class InputCapabilities:
    gain_reduced: bool
    catA_calibrated: bool
    catB_calibrated: bool = False
```

Recommendation: start with the enum now for simplicity and move to capabilities later if needed.

### 11. Specific repo areas to touch

Based on the current structure, the main changes would likely be in:

- `src/osa/utils/cliopts.py`
  - add `--input-state`
  - persist it in `options`
- `src/osa/configs/options.py`
  - add `input_state`
- new modules:
  - `src/osa/pipeline_state.py`
  - `src/osa/plan.py`
  - `src/osa/readiness.py`
  - `src/osa/validation.py`
- `src/osa/job.py`
  - use `ProcessingPlan` for dependency wiring and conditional submission
- sequence-building code
  - wherever build logic decides what steps exist
- launcher scripts under `src/osa/crontab/launchers/`
  - replace file-flag-specific gating with a shared state-based check
- tests in `src/osa/conftest.py` and relevant test modules
  - add fixtures for new input states

### 12. Suggested migration path

#### Phase 1: non-invasive

- add `input_state`
- add plan builder
- map old flags to plan
- do not change launcher behavior yet

#### Phase 2: internal cleanup

- replace direct flag checks in sequencer/job code with plan-based logic
- add readiness/validation functions

#### Phase 3: operational cleanup

- update launchers to call shared readiness checks
- reduce dependence on `GainSelFinished.txt`

#### Phase 4: deprecation

- deprecate `--no-gainsel` and maybe `--no-calib`
- recommend `--input-state ...`

### 13. Recommended final operational model

For the upcoming change, aim for this:

#### Old nights / legacy format

```bash
sequencer --input-state legacy_raw ...
```

#### New nights with gain reduction already done

```bash
sequencer --input-state gain_selected ...
```

#### Newer nights with gain reduction + Cat-A already done

```bash
sequencer --input-state catA_calibrated ...
```

Same sequencer, same codebase, different plan.

### 14. Practical recommendation

If you want the simplest robust implementation:

1. **Do not merge gain selection into sequencer**
2. **Add `input_state`**
3. **Create `ProcessingPlan`**
4. **Drive sequencer dependencies and sequence generation from the plan**
5. **Move launcher gating to a shared readiness concept**
6. **Keep old flags temporarily as overrides**

That gives flexibility without a disruptive rewrite.

## Constraint: keep shell launchers cheap

You noted that the launchers are kept as `.sh` files to avoid starting the Python ecosystem like conda and python so often.

That means the best fit is:

- keep the `.sh` launchers,
- keep cheap file/path checks in shell,
- move only the sequencer’s internal execution logic to `input_state` / `ProcessingPlan`,
- optionally use tiny shell-readable readiness artifacts instead of Python checks on every cron tick.

## Best fit for that constraint

### Do **not** do this on every launcher invocation

Avoid:

```bash
python -m osa.scripts.input_ready ...
```

inside a cron-polled launcher every 15 or 20 minutes.

That would indeed add Python/conda startup overhead.

### Do this instead

Keep shell conditions like today, but make them depend on **state-specific artifacts**.

Examples:

- legacy mode:
  - wait for `GainSelFinished.txt`
- gain-selected mode:
  - wait for a directory, sentinel file, or expected first product indicating gain-selected inputs exist
- catA-calibrated mode:
  - wait for a directory, sentinel file, or expected calibrated products

Then the launcher remains cheap:

```bash
if [ "$INPUT_STATE" = "legacy_raw" ]; then
    [ -e "$GAINSEL_FLAG" ] || exit 0
elif [ "$INPUT_STATE" = "gain_selected" ]; then
    [ -e "$GAIN_SELECTED_READY_FLAG" ] || exit 0
elif [ "$INPUT_STATE" = "catA_calibrated" ]; then
    [ -e "$CATA_READY_FLAG" ] || exit 0
fi
```

Only after that does it activate conda and run the sequencer.

## Revised architecture

### 1. Launchers stay in shell

They remain responsible for:

- cron integration
- cheap readiness gating
- environment activation once per actual run
- file existence checks
- orchestration

### 2. Sequencer gets `input_state`

Once started, the sequencer uses the new state model to:

- skip Cat-A calibration if already done
- avoid depending on gain selection if upstream already handled it
- build the correct job graph

### 3. Shell and Python share the same declared state

For example from the same config or environment variable:

```bash
INPUT_STATE=catA_calibrated
```

Then:

- the launcher uses it for cheap gating,
- the sequencer uses it for job planning.

That keeps behavior consistent without repeated Python startups.

## Concrete practical approach

### Option A: state in environment variable

Set in `osa-env.sh`:

```bash
export INPUT_STATE=legacy_raw
```

or later:

```bash
export INPUT_STATE=catA_calibrated
```

Launcher reads it directly.

Sequencer also receives it:

```bash
sequencer --input-state "$INPUT_STATE" ...
```

This is simple and cheap.

### Option B: state in cfg, but shell reads a mirrored env var

If parsing the cfg in shell is awkward, don’t parse it there. Just mirror the value in `osa-env.sh`.

That is often the most practical operations choice.

### Option C: readiness sentinel files

If upstream changes are handled by external systems, have those systems write a cheap sentinel file such as:

- `GainSelFinished.txt`
- `InputState_gain_selected.txt`
- `InputState_catA_calibrated.txt`

Then launchers only test for one file.

This is extremely cheap and works well with cron.

## Minimal-change recommendation

Given that constraint, I recommend:

1. keep `.sh` launchers,
2. keep current cron structure,
3. make launchers branch on declared `INPUT_STATE`,
4. use cheap shell file checks for readiness,
5. pass `--input-state` into the sequencer,
6. make the sequencer internally derive its `ProcessingPlan` from that state.

That gets you GitHub-renderable markup text while preserving the substance of the note.

Yes — **run-level markers are the right design** if a single date can contain mixed input types.

A date-level marker like `gain_selected.ready` for the whole night becomes ambiguous as soon as some runs are:
- legacy raw,
- some already gain-reduced,
- some already Cat-A calibrated.

So the state model should move from:

- **night-level readiness**

to:

- **run-level capability/state**

## Recommendation
Use:

- **night-level markers** only for global artifacts
  - run summary ready
  - night finished
  - web updated

and use:

- **run-level markers** for processing state
  - gain selected
  - Cat-A calibrated
  - DL1 ready
  - Cat-B ready
  - DL1AB ready
  - DL2 ready

That gives you mixed-mode flexibility.

---

## Good filesystem layout
Something like:

```text
$STATE_ROOT/
  20260523/
    night/
      run_summary.ready
      night_finished.ready
    runs/
      01807/
        input/
          legacy_raw.ready
        products/
          gain_selected.ready
          catA_calibrated.ready
          dl1a.ready
      01808/
        input/
          gain_selected.ready
        products/
          catA_calibrated.ready
          dl1a.ready
      01809/
        input/
          catA_calibrated.ready
        products/
          dl1a.ready
          dl1ab.ready
```

You can simplify the names, but the important part is:
- state is per run,
- monotonic progression is visible,
- shell can test it cheaply.

---

## Even better: capabilities, not exclusive modes
At run level, I would slightly shift from “one mode per run” to “capabilities per run”.

Because a run may have:
- gain reduction done
- Cat-A done
- but not later products

So markers like these are better than a single mutually exclusive mode file:

```text
runs/01807/capabilities/gain_selected.ready
runs/01807/capabilities/catA_calibrated.ready
```

That is more flexible than forcing one enum in shell.

### Why
If you use only one state file like:
- `legacy_raw`
- `gain_selected`
- `catA_calibrated`

you have to decide whether state means:
- current input origin, or
- furthest completed step.

That gets messy.

Marker capabilities avoid that confusion.

---

## Suggested semantics

### Input markers
Describe what the run already has when lstosa sees it:

- `has_raw.ready`
- `has_gain_selected.ready`
- `has_catA_calibrated.ready`

### Output markers
Describe what lstosa has produced:

- `dl1a.ready`
- `catB.ready`
- `dl1ab.ready`
- `dl2.ready`

### Optional provenance marker
You may also want:
- `source_external.ready`
- `source_lstosa.ready`

to distinguish upstream-provided vs locally-produced state.

But that is optional.

---

## How launchers would use this
Shell launchers shouldn’t require one single night-wide condition anymore.

Instead they should:
1. iterate over runs for the night,
2. test each run’s markers,
3. submit only eligible runs.

For example, Sequencer 1 launcher logic becomes conceptually:

- for each run in the run summary:
  - if run has `has_catA_calibrated.ready`, skip calibration-dependent setup and submit downstream-compatible processing
  - else if run has `has_gain_selected.ready`, submit Cat-A calibration + downstream
  - else if run has `has_raw.ready`, either wait for gain selection or skip, depending on configured policy

That does mean launchers get a bit more logic, but not necessarily much if the per-run decision mostly stays in Python.

---

## Important design choice
Since you want to keep shell lightweight, I would **not** make shell fully decide per-run processing logic.

Instead:

### Shell should do only coarse gating
Example:
- “is run summary ready?”
- “is there at least one runnable run?”
- “has the night already finished?”

### Python sequencer should do fine-grained run-level decisions
Because once runs can differ in state, encoding all that branching in shell gets ugly fast.

So the best split is:

- **shell**
  - triggers the sequencer periodically
  - passes the date/config
  - maybe checks night-level prerequisites

- **sequencer**
  - inspects all runs for the date
  - determines each run’s input state
  - builds the right jobs for each run

That is probably the cleanest architecture for mixed-mode nights.

---

## So should markers be run-level?
**Yes. Absolutely, if mixed data states can coexist within a night.**

## Should launchers consume run-level markers directly?
**Only a little.** Mostly no.

If you push too much per-run logic into `.sh`, it becomes fragile.

## Best hybrid design
### Shell launchers check only night-level prerequisites:
- run summary exists
- not night finished
- maybe state directory exists

### Sequencer handles run-level markers internally:
- each run can be legacy raw, gain-selected, or Cat-A-calibrated
- sequencer builds per-run execution plan

That gives:
- minimal shell complexity,
- full flexibility,
- mixed-run support.

---

## Practical structure I’d recommend

### Night level
```text
state/YYYYMMDD/night/run_summary.ready
state/YYYYMMDD/night/night_finished.ready
```

### Run level
```text
state/YYYYMMDD/runs/RUNID/capabilities/has_gain_selected.ready
state/YYYYMMDD/runs/RUNID/capabilities/has_catA_calibrated.ready
state/YYYYMMDD/runs/RUNID/products/dl1a.ready
state/YYYYMMDD/runs/RUNID/products/catB.ready
state/YYYYMMDD/runs/RUNID/products/dl1ab.ready
state/YYYYMMDD/runs/RUNID/products/dl2.ready
```

Optional:
```text
state/YYYYMMDD/runs/RUNID/metadata/input_origin.txt
```

---

## One more useful idea
You may want one small machine-readable file per run, for example:

```json
{
  "run": 1807,
  "date": "2026-05-23",
  "has_gain_selected": true,
  "has_catA_calibrated": false,
  "dl1a_ready": false,
  "source": "external"
}
```

But if your main goal is cheap shell checks, marker files are simpler and probably better.

---

## Final recommendation
For mixed run types in one date:

- **yes, use run-level markers**
- keep **night-level markers** only for global coordination
- keep **launchers in shell**
- let **sequencer make per-run decisions**
- model run state as **capabilities/markers**, not just one date-wide mode

If you want, I can next propose:
1. a **minimal run-level marker scheme**, and  
2. how the sequencer’s per-run decision tree should work for:
   - raw runs,
   - gain-selected runs,
   - Cat-A-calibrated runs.


