# Reward Seeking Versus Instruction Following Transfer

## Question

Induce two behavioural profiles by prompting and prefill, then compare zero-shot transfer across environments that vary in exploitability and goal specification. Because the profiles are induced rather than trained, the induction strength must be verified before any transfer comparison is meaningful.

## Why it is worth measuring

The question is answerable at small scale with a locally runnable pilot, and it has a
clean falsification condition: the measurement is built so a negative result with adequate
power is reportable rather than a dead end. Most of the design effort goes into the
controls, because the easy version of this measurement would produce a number that looks
like an answer and is really an artifact of how the stimuli were built.

## Objectives

1. Induce the two profiles by prompting and verify the induction independently.
2. Build environments varying exploitability and goal specification as crossed factors.
3. Compare zero-shot transfer between profiles at matched capability.
4. Report whether the difference survives an environment where exploitation is impossible.

## Method

The repository implements a five-stage pipeline. Stimuli are constructed locally so their
ground truth is known rather than assumed. Model-side collection is measured against a
revision-pinned small open-weight model and fails closed when weights are absent. The core
measurement runs with its controls in the same pass, so a result and the arm that would
undermine it are produced together rather than in separate sessions.

Domain code lives in `src/rewinst/rewinst/`. The shared infrastructure — typed Hydra
configuration, versioned artifact cache, hooks and generation, metrics, ablation,
reporting and CI — is separate from it, so the science is reviewable without reading the
plumbing.

## Plan

| ID | Workstream | Size | Description |
|---|---|---|---|
| WS-01 | Profile induction and verification | M | Induce both profiles and verify induction with a behavioural check independent of the transfer measurement. |
| WS-02 | Crossed environment family | L | Environments crossing exploitability against goal specification, with each cell populated. |
| WS-03 | Exploitability verification | M | Demonstrate the exploit in exploitable environments and its absence in robust ones, so the factor is measured not assumed. |
| WS-04 | Zero-shot transfer comparison | L | Compare profiles across the crossed grid with paired intervals. Carries the headline claim. |
| WS-05 | Capability matching and the unexploitable arm | M | Match profiles on capability and report the unexploitable-environment arm separately, since it is the discriminating case. |
| WS-06 | Documentation, presets and figures | M | Transfer-by-cell figure, domain presets, and documentation to the standard's floor. |

## Confounds

| Risk | Control |
|---|---|
| Prompt-induced profiles are too weak to differ, producing an uninformative null | Induction strength is verified first with a stated threshold; below it, the transfer comparison is not run |
| The reward-seeking profile wins only where exploitation is possible, which is trivial | The unexploitable arm is reported separately and is the discriminating result |
| Profile induction changes general capability, confounding transfer | Capability matching is a required arm and the match is reported |
| Synthetic smoke output is mistaken for a measured result | `is_synthetic` is set at production and survives aggregation; `claim_ok` is false whenever any input was synthetic |
| Pilot n too small to separate a true null from an underpowered test | Report minimum detectable effect beside every interval and run an equivalence test (TOST) before claiming a null (X12) |
| Small open-weight models do not exhibit the phenomenon at all | State a falsification threshold before running; a clean negative with adequate power is a reportable result |

## What would make this credible

- Induction strength is verified against a stated threshold before transfer is measured.
- Every cell of the exploitability-by-specification cross is populated.
- Exploitability is demonstrated rather than assumed for each environment.
- Transfer is compared at matched capability, with the match reported.
- The unexploitable-environment arm is reported separately from the aggregate.

## Honesty commitments

Synthetic output is labelled where it is produced and the label survives into the report.
A claim whose gate fails is suppressed and its block reason named, rather than restated
with hedging. No number in this repository is presented as measured unless it came from a
run against real weights, and the run directory carries the seed, the commit and the model
revision that produced it.

## Compute

The pilot runs on an Apple M4 with no CUDA and no API keys. Model forward passes use MPS
where available; the statistics run on CPU and are documented as such.

## Current status

Infrastructure and the domain measurement are implemented and unit tested. No measured
result is reported yet. The design document at [`TECHNICAL.md`](TECHNICAL.md) states the
artifact contract and the open technical decisions; the program plan under
[`programs/`](programs/) carries the workstream detail and acceptance criteria.
