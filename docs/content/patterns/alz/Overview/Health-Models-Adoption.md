---
title: Adopting Azure Monitor health models
weight: 20
---

> [!important]
> [Azure Monitor health models](https://learn.microsoft.com/azure/azure-monitor/health-models/overview) are in public preview. This guidance moves with the product, so expect it to change as health models do.

### In this page

> [Overview](../Health-Models-Adoption#overview) </br>
> [What AMBA-ALZ delivers](../Health-Models-Adoption#what-amba-alz-delivers) </br>
> [What a health model adds](../Health-Models-Adoption#what-a-health-model-adds) </br>
> [AMBA alerts become signals](../Health-Models-Adoption#amba-alerts-become-signals) </br>
> [Health models at Landing Zone scale](../Health-Models-Adoption#health-models-at-landing-zone-scale) </br>
> [An adoption path](../Health-Models-Adoption#an-adoption-path) </br>
> [References](../Health-Models-Adoption#references) </br>

## Overview

Azure Landing Zones give an organization a repeatable way to scale Azure. AMBA-ALZ adds alerting to that at organization scale, deployed by Azure Policy across the management group hierarchy. Azure Monitor health models add the layer above: the same measurements, arranged into a graph of entities, so that independent alert rules become one answer to "is this working right now".

A health model reads the same Azure Monitor data an AMBA alert rule reads. The difference is what the measurement produces.

- *In AMBA:* An alert rule produces an alert: something crossed a threshold, it goes into the list of many alerts from many rules.

- *In a health model:* the signal (aka Alert rule in AMBA) produces a state that is rolled up to the topics and flows it affects, and if one the affected flows has an alert, it fires it.

This page compares the two and describes how to introduce a health model into a landing zone.

## What AMBA-ALZ delivers

AMBA-ALZ deploys alert rules with Azure Policy, scoped to the Azure Landing Zones management group hierarchy:

- **A catalogue of alerts per service.** 675 alert definitions across 49 Azure services, split into metric, log search and activity log alerts. See [Alerts Details](../../getting-started/Alerts-Details) and the [metric](../../getting-started/Metric-Alerts-Table), [log search](../../getting-started/Log-Search-Alerts-Table) and [activity log](../../getting-started/Activity-Log-Alerts-Table) tables.
- **Grouped into initiatives, assigned per archetype.** Connectivity, identity, management, landing zone and service health each get their own initiative at the matching management group. See [Policy Initiatives](../../getting-started/Policy-Initiatives).
- **Deployed and kept in place by remediation.** Most definitions use `DeployIfNotExists`, so new resources are covered automatically and drift is corrected. See [Remediate Policies](../../HowTo/deploy/Remediate-Policies).
- **Tuned by tag, not by editing policy.** A tag opts a resource out of monitoring, and another overrides a threshold for one resource. See [Disable Policies](../../HowTo/Disabling-Policies) and [Override alert thresholds](../../HowTo/Threshold-Override).
- **Notification handled centrally.** An action group and an alert processing rule per subscription, or your own. See [Bring Your Own Notifications](../../HowTo/Bring-your-own-Notifications).

That solves coverage. Assign an initiative at a management group and every subscription that lands under it, today or next year, gets a consistent set of alert rules.

What it does not carry is dependency. Every alert rule is deployed independently of every other one, so nothing in the system knows that an ExpressRoute circuit, a firewall and a gateway are three parts of one connectivity service. Three alerts fire, and a human decides whether that adds up to an outage.

## What a health model adds

A health model is an Azure resource, `Microsoft.CloudHealth/healthmodels`, holding a graph of entities and the rules that turn monitoring data into a health state for each one.

A health model arranges measurements like your Alert rules into a graph of entities. These entities represent your user flows and systems flows:

- **Fewer alerts, not fewer measurements.** Alerting moves from every rule to the entities you decide matter, so notification volume tracks the number of real problems rather than the size of the estate.
- **Criticality is expressed once, in the graph.** A dependency says which components a service actually needs, so importance comes from position in the graph instead of a severity set per rule.
- **Recovery is visible, and measurable.** You see when something clears, and how long it was broken.

![A health model for a landing zone](../../media/a-health-model-for-a-landing-zone.svg)

Entities are the new part. A landing zone, a platform domain or a system flow has no representation in AMBA, and that is what the rest of the model hangs off: [health states](https://learn.microsoft.com/azure/azure-monitor/health-models/concepts) on each entity, [relationships](https://learn.microsoft.com/azure/azure-monitor/health-models/rollup) between them, a [health objective](https://learn.microsoft.com/azure/azure-monitor/health-models/concepts) on the root, and [alerts](https://learn.microsoft.com/azure/azure-monitor/health-models/alerts) on entity state.

Build and inspect all of it in the [designer](https://learn.microsoft.com/azure/azure-monitor/health-models/designer), the [health state views](https://learn.microsoft.com/azure/azure-monitor/health-models/analyze-health), or from [the Azure CLI](https://learn.microsoft.com/azure/azure-monitor/health-models/cli).

## AMBA alerts become signals

The health model designer has an **Import from alert rules** option that creates "a signal based on existing alert rules that are defined for the Azure resource represented by the entity. The same signal and criteria from the alert rule is used for the new signal."

![AMBA alerts become health model signals](../../media/amba-alerts-become-health-model-signals.svg)

Read the diagram bottom to top. Signals sit inside the resource they measure. Resource state rolls into a platform capability, capabilities roll into a landing zone flow, and the flow rolls into the domain root that carries the objective.

Two modelling decisions matter here. Hybrid connectivity uses a **not-healthy limit**, because it has a failover path: the primary network path is down but the failover is healthy, so the flow is degraded rather than an outage. The diagnostics pipeline is attached with **Suppressed**, so its state is visible on the graph but not impacting the connectivity flow.

| AMBA-ALZ construct | Health model equivalent | Notes |
|---|---|---|
| Metric alert | Azure resource signal | Direct equivalent. Namespace, metric name and aggregation carry over. |
| Log search alert | Log Analytics workspace signal | Equivalent in principle. The AMBA query needs rework, see gaps below. |
| Resource Health activity log alert | Azure Resource Health signal | Equivalent and simpler. A per-entity toggle instead of a deployed rule. |
| Service Health activity log alert | No equivalent | Service Health is a subscription level activity log event, not one of the four signal types. |
| Monitored Azure resource | Azure resource entity | One resource can appear in several models, each with independent state. |
| A service built from several resources | Generic entity with children | No AMBA-ALZ counterpart. Dependency between resources is what the entity graph expresses. |
| ALZ management group archetype | Generic entity populated by a discovery rule | The hierarchy becomes topology inside a model, not scope around it. |
| `MonitorDisable` tag | A `where` clause in the discovery query | Resource Graph discovery filters on tags, so one tag can govern both. |
| Threshold override tag | No equivalent | See gaps below. |
| Action group | The same action group | Health model alerts use the action groups you already have. |


## Health models at Landing Zone scale

At scale, it's best to use a layered approach. You start with a high level model per tenant and then split into the different domains of your landing zones and further down the platforms and flows that build up their capabilites domains.

![Health models at tenant scale](../../media/health-models-at-tenant-scale.svg)

Each domain references a health model designed for that domain and with that receives it's health state.

![Connectivity-Contoso-Prod](../../media/connectivity-contoso-prod.svg)

That referenced model is where the entities and signals live. It is owned by the team that runs connectivity, and it changes when their flows change, without touching the overarching model.

The same Azure resource can appear in several models, with different signals in each. You can imagine that an Azure Firewall might be used centrally for a variety of different flows. Health models allow to put measurements to the Azure Firewall, specific for each flow.

![One resource, two models](../../media/one-resource-two-models.svg)

The hub firewall is one resource. The connectivity model watches SNAT ports and tunnel state. The partner exchange model watches threat intel hits and denied flows on the same device. Neither model carries signals it does not care about.


## Health model adoption path

Azure Monitoring health models can be adopted in three stages:
1. Discovery: where you build up the inventory of all resources involved your landing zone
2. Analyze and Modelling: where you decide what you matters to you, what are the platform and committments you want to track and which signals on which resources carry your commitments.
3. Refine: over time and with gaining confidence in the model and signals you can improve the model and get an invaluable tool to track your landing zone health and commitments.

### 1. Discovery

Create one health model with a discovery rule per ALZ domain. This is inventory work: you are not deciding what healthy means yet, you are finding out what you actually run.

![Adoption step 1, discovery](../../media/adoption-step-1-discovery.svg)

A [Resource Graph discovery rule](https://learn.microsoft.com/azure/azure-monitor/health-models/discoveries) per domain brings in all involved Azure resources with recommended signals. This is your starting point to decide what matters to you.

- with the *recommended signals* option, the current AMBA baseline alerts are being added as signals on the resource.
- already a single signal over the threshold will degrade the whole domain. This means that you either have to update the threshold or split the domain.


### 2. Analyze and Modelling

With the overview from the previous step, you can now model your commitments, e.g. that you're offering Egress Control, Secret Management and Hybrid connectivity. These are capabilities that you offer. Once you modelled them, you add the Azure resource that contribute to them as dependencies underneath with signals that represent that the resources are working as expected. The health state of the capability is then rolled up.

![Adoption step 2, analyze](../../media/adoption-step-2-analyze.svg)

This is a design step more than a configuration step. The output is a first model of platform health: a handful of aspects, each backed by the measurements you already trust.

### 3. Refine

For each ALZ domains you might create a specific health model that represents the capabilities and flows of that domain, which the resources that contribute to it. You can use a mix of manually build entities and modelling and combine it with discovery rules for places where you don't control the life cycle of the contributing resources.

![Adoption step 3, refine](../../media/adoption-step-3-refine.svg)

You build in failover paths and alerting on the levels where it's most meaningful to you. You can now separate between an outage of the secondary path and a full outage of all connectivity.

## References

Azure Monitor health models:

- [Health models in Azure Monitor (preview)](https://learn.microsoft.com/azure/azure-monitor/health-models/overview)
- [Health model concepts](https://learn.microsoft.com/azure/azure-monitor/health-models/concepts)
- [Signals](https://learn.microsoft.com/azure/azure-monitor/health-models/signals)
- [Alerts](https://learn.microsoft.com/azure/azure-monitor/health-models/alerts)
- [Create discovery rules](https://learn.microsoft.com/azure/azure-monitor/health-models/discoveries)
- [Create a health model](https://learn.microsoft.com/azure/azure-monitor/health-models/create)
- [Configure health rollup](https://learn.microsoft.com/azure/azure-monitor/health-models/rollup)
- [Configure using the designer](https://learn.microsoft.com/azure/azure-monitor/health-models/designer)
- [Analyze health state](https://learn.microsoft.com/azure/azure-monitor/health-models/analyze-health)
- [Create a health model with the Azure CLI](https://learn.microsoft.com/azure/azure-monitor/health-models/cli)
- [Health models FAQ](https://learn.microsoft.com/azure/azure-monitor/health-models/health-models-faq)

Resource provider reference:

- [Microsoft.CloudHealth/healthmodels](https://learn.microsoft.com/azure/templates/microsoft.cloudhealth/healthmodels)
- [az monitor health-models](https://learn.microsoft.com/cli/azure/monitor/health-models)

Wider guidance:

- [Well-Architected health modeling design guide](https://learn.microsoft.com/azure/well-architected/design-guides/health-modeling)
