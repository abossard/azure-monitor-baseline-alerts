---
title: Health model adoption path
weight: 21
---

### In this page

> [Challenges](../Health-Models-Adoption-Path#challenges) </br>
> [Phase 1, inventory and observe](../Health-Models-Adoption-Path#phase-1-inventory-and-observe) </br>
> [Phase 2, model what contributes to health](../Health-Models-Adoption-Path#phase-2-model-what-contributes-to-health) </br>
> [Phase 3, alert on state](../Health-Models-Adoption-Path#phase-3-alert-on-state) </br>
> [Phase 4, broaden and improve](../Health-Models-Adoption-Path#phase-4-broaden-and-improve) </br>
> [Related](../Health-Models-Adoption-Path#related) </br>

## Challenges
- The health model can discover your existing resources, but it won't know the specific flows you care about.
- You can build a full estate and then start to add flows
- At the end, it's important health models are being considered not just on a high level, but with every resource deployment.
- With AMBA you get alerts on every single Azure resource, and you might be used to intepreting them in isolation. Health models alert on health state changes.

## Phase 1, inventory and observe

Goal: become wide and identify involved resources.

**Do.** Deploy the AHM-ALZ-Baseline-Model (Bicep example/policy) to build an estate of your full Azure Landing Zone resources.
- it has customizable discovery rules

**What you get.** An inventory of what the domain is actually made of, and a first read on whether the signals you chose track reality.

## Phase 2, model what contributes to health

**Do.** Add your platform domains, flows and topics you want to get alerted on.

**What you get.** A dependency graph instead of a list, and one number that says how the domain is doing over time.

## Phase 3, alert on state

**Do.**

**What you get.** Notifications that fire on the state of something you care about, and clear again when it recovers.

## Phase 4, broaden and improve

**Do.** Repeat per platform domain and per landing zone. Nest the models into a portfolio view.

**What you get.** Objective attainment over time, which is the measure of whether operations are actually improving rather than just busier.

## Related

- [Adopting Azure Monitor health models](../Health-Models-Adoption)
