<p align="center">
  <h1 align="center">Reward Seeking Versus Instruction Following Transfer</h1>
  <p align="center"><strong>Compare how reward-seeking and instruction-following behaviors transfer across small task shifts.</strong></p>
</p>

---

## Overview

This repository implements experimental profiles for **Reward Seeking Versus Instruction Following Transfer**. Config, caching, hooks, metrics, ablations, reporting, and CI support local pilots on small open-weight models.

Hypothesis (one line): Compare how reward-seeking and instruction-following behaviors transfer across small task shifts.

## Status

Shared infrastructure is in place; domain stages must pass harness validation before any measured claim.

| Command | Purpose |
|---|---|
| `make install-dev` | editable install + pinned requirements |
| `make test` | full unit suite |
| `make ci` | lint + test + typecheck |
| `make pilot` | end-to-end pilot profile |
