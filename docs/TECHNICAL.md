# Reward Seeking Versus Instruction Following Transfer — Technical Design

Design and method document for `reward-vs-instruction`. This describes what the repository measures, how
the measurement is constructed, which confounds it controls for, and what it is not
allowed to claim. It is written before the numbers exist, and it is the document to read
first if you want to know whether a result here would mean anything.

**Status:** infrastructure complete; the domain measurement is implemented and unit
tested. No measured result is reported in this repository yet. Any number produced by the
smoke profile is labelled synthetic at the point of production and never reaches a claim.

---

## 1. Question

Induce two behavioural profiles by prompting and prefill, then compare zero-shot transfer across environments that vary in exploitability and goal specification. Because the profiles are induced rather than trained, the induction strength must be verified before any transfer comparison is meaningful.

### 1.1 Goals

| ID | Goal |
|---|---|
| G-01 | Induce the two profiles by prompting and verify the induction independently. |
| G-02 | Build environments varying exploitability and goal specification as crossed factors. |
| G-03 | Compare zero-shot transfer between profiles at matched capability. |
| G-04 | Report whether the difference survives an environment where exploitation is impossible. |

### 1.2 What would falsify the hypothesis

The measurement is designed so a clean negative is reportable. Every headline estimate is
paired with a minimum detectable effect, and a null is only claimed when an equivalence
test places the interval inside a pre-stated margin. An interval that merely contains zero
is reported as uninformative, not as evidence of absence.

---

## 2. Architecture

The repository is two layers. The shared infrastructure spine handles configuration,
caching, model loading, hooks, metrics, ablation, reporting and CI. The domain package
`src/rewinst/rewinst/` holds the science.

### 2.1 Domain modules

| Module | Purpose |
|---|---|
| `organisms.py` | Profile induction by prompt and prefill, with induction-strength verification |
| `envs.py` | Environment family crossing exploitability against goal specification |
| `rollouts.py` | Episode execution with per-episode score and behaviour logging |
| `transfer.py` | Zero-shot transfer comparison between profiles; the core measurement |
| `hackability.py` | Verification that exploitable environments are exploitable and robust ones are not |
| `matched.py` | Capability matching so transfer differences are not capability differences |
| `model_runtime.py` | Fail-closed measured runtime: load pinned weights or raise, never silently synthesize |
| `pipeline.py` | Stage orchestration binding domain calls to the spine's stage graph |
| `enrich.py` | Attach confidence intervals, minimum detectable effect, and claim gates to every metric |

### 2.2 Layer responsibilities

| Layer | Path | Responsibility | Constraints |
|---|---|---|---|
| Spine | `src/rewinst/` | Config, cache, models, metrics, reporting | Print-free, importable, no domain knowledge |
| Domain | `src/rewinst/rewinst/` | The measurement and its controls | Print-free, no argparse, no `__main__` |
| Stages | `src/rewinst/stages.py` | Binds domain calls to the stage graph | Thin adapters only |
| CLI | `scripts/` | Argparse and Hydra entry points | Prints allowed, loose annotations allowed |

### 2.3 Why the harness takes injected callables

The evaluation path never imports a concrete model class. Scorers and model callables
arrive as injected functions with named type aliases, so the measurement can be exercised
against a stub in tests, against a small open-weight model in the pilot, and against
anything else later without touching the harness. A harness that imports a model is a
harness that dies with that model.

---

## 3. Method

### 3.1 Stage graph

| Stage | Input | Output | Fails closed when |
|---|---|---|---|
| `build_dataset` | config | stimuli and splits | construction-validity check fails |
| `collect` | stimuli | model-side measurements | weights absent and `force_synthetic` false |
| `fit` | measurements | fitted estimator | instrument validation below threshold |
| `evaluate` | estimator | core measurement plus controls | inputs unmatched or missing |
| `report` | measurement | headline payload | never — it reports the block reason |

### 3.2 Workstreams

| ID | Workstream | Size | Depends on | Touches |
|---|---|---|---|---|
| WS-01 | Profile induction and verification | M | — | `organisms.py`, `model_runtime.py` |
| WS-02 | Crossed environment family | L | — | `envs.py` |
| WS-03 | Exploitability verification | M | WS-02 | `hackability.py` |
| WS-04 | Zero-shot transfer comparison | L | WS-01, WS-03 | `rollouts.py`, `transfer.py` |
| WS-05 | Capability matching and the unexploitable arm | M | WS-04 | `matched.py`, `enrich.py` |
| WS-06 | Documentation, presets and figures | M | WS-05 | `pipeline.py` |

### 3.3 Statistical treatment

| Quantity | Treatment | Rationale |
|---|---|---|
| Headline effect | Bootstrap interval over the resampling unit that carries the variation | The sampling unit is items or episodes, not concepts or conditions |
| Null claim | Two one-sided equivalence test against a pre-stated margin | An interval containing zero cannot separate a null from an underpowered test |
| Detectability | Minimum detectable effect reported beside every interval | Makes "we saw nothing" and "we could not have seen it" distinguishable |
| Multiple conditions | Per-condition estimates reported; no single aggregate stands alone | An aggregate hides which condition carries the result |

---

## 4. Confounds and controls

Every risk below is either mitigated in code or reported as a limitation. None is left
implicit.

| ID | Confound | Control |
|---|---|---|
| C-01 | Prompt-induced profiles are too weak to differ, producing an uninformative null | Induction strength is verified first with a stated threshold; below it, the transfer comparison is not run |
| C-02 | The reward-seeking profile wins only where exploitation is possible, which is trivial | The unexploitable arm is reported separately and is the discriminating result |
| C-03 | Profile induction changes general capability, confounding transfer | Capability matching is a required arm and the match is reported |
| C-04 | Synthetic smoke output is mistaken for a measured result | `is_synthetic` is set at production and survives aggregation; `claim_ok` is false whenever any input was synthetic |
| C-05 | Pilot n too small to separate a true null from an underpowered test | Report minimum detectable effect beside every interval and run an equivalence test (TOST) before claiming a null (X12) |
| C-06 | Small open-weight models do not exhibit the phenomenon at all | State a falsification threshold before running; a clean negative with adequate power is a reportable result |

---

## 5. Honesty mechanisms

| Mechanism | Where | Effect |
|---|---|---|
| `is_synthetic` flag | set at production in the collection path | survives aggregation into the final report |
| `claim_ok` gate | `enrich.py` | a claim is suppressed rather than softened when a gate fails |
| `blocked_by` list | report payload | names which precondition failed, so the reason is visible |
| Fail-closed measured path | `model_runtime.py` | a measured stage raises rather than substituting synthetic data |
| Instrument validation | `fit` stage | the measurement is tested against planted structure before it is trusted |
| Provenance in every payload | `_util.stage_result` | `task`, `seed`, `git_sha`, `n` travel with the number |

---

## 6. Reproducibility

| Element | Implementation |
|---|---|
| Seeding | `set_seed` covers `random`, numpy, torch, CUDA and `PYTHONHASHSEED` |
| Commit provenance | `git_sha()` in every result payload, `"unknown"` outside a repository |
| Run directories | timestamped, holding the resolved config and `run_metadata.json` |
| Model pinning | revision recorded per model in the registry and resolved into the manifest |
| Artifact cache | one file per record, append-only manifest, atomic writes, hard error on version mismatch |
| Device | MPS preferred with `PYTORCH_ENABLE_MPS_FALLBACK` set explicitly and recorded |

Numeric work in numpy, pandas and scikit-learn runs on CPU. Only the model forward pass
uses the accelerator, and the documentation does not describe the rest as accelerated.

---

## 7. Expected artifacts

Every stage writes into a timestamped run directory. The contract is fixed before the
code produces anything, so a missing artifact is a failure rather than an omission.

| Stage | Path | Contents |
|---|---|---|
| `build_dataset` | `runs/<timestamp>/artifacts/dataset/` | Stimuli, splits and construction-validity checks |
| `collect` | `runs/<timestamp>/artifacts/features/` | Model-side collection keyed by revision and layer |
| `fit` | `runs/<timestamp>/artifacts/basis/` | Fitted estimators plus instrument validation |
| `evaluate` | `runs/<timestamp>/artifacts/coverage/` | Core measurement, controls, intervals and claim gate |
| `report` | `runs/<timestamp>/artifacts/report/` | Headline payload with the claim verdict |

Each payload carries `task`, `seed`, `git_sha`, `n`, `is_synthetic`, and — for stages that
support a claim — `claim_ok` with its `blocked_by` list.

---

## 8. Compute envelope

| Profile | Items | Model | Wall clock | Notes |
|---|---|---|---|---|
| `smoke` | 32 | none | seconds | `force_synthetic: true`, no weights required |
| `pilot` | 256–512 | small open-weight | minutes on an Apple M4 | measured, fails closed without weights |
| `full` | 1,024+ | small open-weight | hours | not run locally; the profile exists so the scale is explicit |

---

## 9. Scope boundaries

| Excluded | Reason |
|---|---|
| Reinforcement-learning training of the organisms | Out of scope for this phase |
| Synthetic document fine-tuning | Out of scope for this phase |
| Training-speed comparisons requiring real RL | Out of scope for this phase |

---

## 10. Success criteria

Each criterion is verified by a test or by an artifact in a run directory. None is
verified by reading the code.

| ID | Criterion | Verified by |
|---|---|---|
| SC-01 | Induction strength is verified against a stated threshold before transfer is measured. | test or run artifact |
| SC-02 | Every cell of the exploitability-by-specification cross is populated. | test or run artifact |
| SC-03 | Exploitability is demonstrated rather than assumed for each environment. | test or run artifact |
| SC-04 | Transfer is compared at matched capability, with the match reported. | test or run artifact |
| SC-05 | The unexploitable-environment arm is reported separately from the aggregate. | test or run artifact |

---

## 11. Open technical decisions

- [ ] Prompt-induced profiles are too weak to differ, producing an uninformative null — current mitigation: Induction strength is verified first with a stated threshold; below it, the transfer comparison is not run
- [ ] The reward-seeking profile wins only where exploitation is possible, which is trivial — current mitigation: The unexploitable arm is reported separately and is the discriminating result
- [ ] Profile induction changes general capability, confounding transfer — current mitigation: Capability matching is a required arm and the match is reported
- [ ] Synthetic smoke output is mistaken for a measured result — current mitigation: `is_synthetic` is set at production and survives aggregation; `claim_ok` is false whenever any input was synthetic
- [ ] Pilot n too small to separate a true null from an underpowered test — current mitigation: Report minimum detectable effect beside every interval and run an equivalence test (TOST) before claiming a null (X12)
- [ ] Small open-weight models do not exhibit the phenomenon at all — current mitigation: State a falsification threshold before running; a clean negative with adequate power is a reportable result

---

## 12. What this repository does not claim

- No measured result is reported. The smoke profile produces synthetic output, labelled
  as such, and the claim gate refuses it.
- Findings on small open-weight models are not evidence about frontier models. The scale
  is a constraint of the compute envelope in section 8, and it is stated wherever a result
  would otherwise imply generality.
- A passing test suite is evidence the measurement is implemented as described. It is not
  evidence that the measurement is valid; that is what the instrument validation in the
  `fit` stage is for, and its result is reported separately.
