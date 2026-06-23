# Technical approach

## Why physics-first?

Most predictive maintenance approaches are data-driven: train a model on historical sensor readings and failure events, then predict future failures. This works well when you have years of labeled failure data from a large fleet.

It breaks down in exactly the situations manufacturers actually face: a new machine with no failure history, a one-off production line, or a facility that can't afford unplanned downtime while waiting years to accumulate training data.

The physics-first approach inverts this. OEM manuals already contain the degradation physics — service intervals, tolerance thresholds, failure mode descriptions — expressed as engineering knowledge. Kaful extracts that knowledge and uses it to construct a model that can produce meaningful RUL forecasts from day one.

## Pipeline design decisions

### RAG over manuals (not fine-tuning)

Kaful uses retrieval-augmented generation to extract component specs and failure modes from OEM PDFs. Fine-tuning was considered and rejected: the relevant information is sparse and machine-specific, retrieval keeps the source document as the authority, and it generalizes to new machine types without retraining.

### Forward Monte Carlo, not a point estimate

The prognosis stage propagates the physics model forward rather than producing a single RUL number. A single number implies false precision; forward Monte Carlo over the model yields a distribution over the time each component crosses its failure threshold, which gives a calibrated uncertainty range — actionable for a maintenance engineer in a way a point estimate isn't.

A classical particle filter was considered, and it's the natural choice when a machine is streaming live sensor telemetry to condition on. That isn't the setting here: Kaful generates the twin from documentation alone, with no observation stream to weight and resample against, so the prognosis is forward uncertainty propagation rather than Bayesian filtering. (An early particle-filter implementation also hit degeneracy — the ensemble collapsing to a single trajectory under heavy censoring — which reinforced the move to direct forward propagation.)

Under a fixed operating profile the composite model is deterministic, so that distribution turns out to be a narrow band around a single degradation trajectory. The demo exploits this: one forward pass to the horizon yields every component's threshold-crossing time, and the uncertainty band is reconstructed as ±2σ from a per-event coefficient of variation calibrated on the model's process noise — which is what lets the live forecast run with no backend.

### Agentic pipeline (not a monolithic model)

The seven-stage pipeline is structured as an agentic system where each stage's output becomes the next stage's input, with LLM reasoning driving the comprehension, triage, and codegen stages. This decomposition makes it possible to inspect and debug each stage independently, which matters when the input is an OEM PDF and the output is a physics model.

## Known limitations

- RUL forecasts are only as good as the physics encoded in the OEM manual. Underdocumented failure modes produce wider uncertainty ranges.
- The model requires calibration per machine type (handled by the phase 6.5 calibration module) to keep degradation timescales from triggering too early.
- Current support targets industrial equipment with documented degradation physics; HVAC, hydraulic, and purely electrical systems require additional physics primitives.