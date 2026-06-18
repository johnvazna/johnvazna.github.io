---
title: 'Data Quality as a Product'
seoTitle: 'Data Quality as a Product: Contracts, Validation, and Pipeline Reliability'
description: 'Reliable pipelines need explicit data contracts, validation layers, observability, and ownership instead of ad hoc checks after something breaks.'
pubDate: 'Jun 17 2026'
heroImage: '../../assets/blog/data-quality-as-a-product/hero.png'
tags:
  - Data engineering
  - Data quality
  - Data contracts
  - Pipelines
---

Data quality is often treated as a cleanup activity. A dashboard looks wrong, someone opens an incident, the team patches a transformation, and a few checks are added after the damage is already visible.

That is not data engineering. That is data maintenance.

For important datasets, quality should be designed as a product contract. The dataset has consumers, expectations, failure modes, owners, and release criteria. If those things are not explicit, the pipeline may be running, but the data product is not engineered.

## The Difference Between A Job And A Product

A job has a schedule. A product has a contract.

A job answers:

- Did the task run?
- Did the task finish?
- Did the orchestrator mark it green?

A data product answers:

- Is the data fresh enough for the consumer?
- Is the schema still compatible?
- Are the business rules still true?
- Can consumers trust the output?
- Who owns the dataset when it fails?
- What happens when quality drops?

This distinction matters because modern datasets rarely serve one dashboard. The same table might feed BI, machine learning features, operational alerts, reconciliation workflows, and AI agents. When the data is wrong, every downstream system inherits that uncertainty.

## Contracts Before Checks

Checks are useful, but only after the contract is clear.

A good data contract defines:

- Required columns and types.
- Nullability expectations.
- Uniqueness rules.
- Accepted value ranges.
- Freshness requirements.
- Semantic meaning of fields.
- Backward compatibility expectations.
- Ownership and escalation path.

For example, a payments table might define that `payment_id` is unique, `amount` is positive, `currency` is present, `processed_at` cannot be earlier than `created_at`, and daily volume cannot drop by 80 percent without an expected event.

Those are not random tests. They are product expectations.

I like writing that contract in a format that can be read by both humans and automation:

```ts
type DataContract = {
	dataset: string;
	owner: string;
	freshnessMinutes: number;
	primaryKey: string[];
	requiredColumns: Record<string, 'string' | 'number' | 'timestamp' | 'boolean'>;
	rules: Array<{
		name: string;
		severity: 'warning' | 'blocking';
		sql: string;
	}>;
};

const paymentsContract: DataContract = {
	dataset: 'gold.payments_daily',
	owner: 'payments-platform',
	freshnessMinutes: 90,
	primaryKey: ['payment_id'],
	requiredColumns: {
		payment_id: 'string',
		amount: 'number',
		currency: 'string',
		created_at: 'timestamp',
		processed_at: 'timestamp',
	},
	rules: [
		{
			name: 'amount_must_be_positive',
			severity: 'blocking',
			sql: 'amount > 0',
		},
		{
			name: 'processed_after_created',
			severity: 'blocking',
			sql: 'processed_at >= created_at',
		},
	],
};
```

The exact representation can be YAML, JSON, Python, or database metadata. The important part is that the contract is explicit enough to generate checks, documentation, and ownership.

## Validation Should Exist At Multiple Boundaries

One mistake I see often is putting all validation at the end of the pipeline. That catches failures late, after invalid data has already moved through several layers.

I prefer layered validation:

- Ingestion validation for raw shape and required fields.
- Bronze-to-silver validation for parsing and normalization.
- Silver-to-gold validation for business rules.
- Consumer-facing validation for freshness and compatibility.

Each layer answers a different question. Raw validation asks, "Did we receive something structurally usable?" Business validation asks, "Does the transformed data still mean what consumers think it means?"

```sql
select
	count(*) as invalid_rows
from silver_payments
where payment_id is null
   or amount <= 0
   or processed_at < created_at;
```

This check is simple, but it encodes a real contract. It is valuable because it maps to a business invariant, not because it is complex.

For production, I prefer persisting validation results instead of only failing a notebook or job:

```sql
insert into data_quality_results (
	dataset_name,
	rule_name,
	severity,
	invalid_rows,
	checked_at
)
select
	'gold.payments_daily' as dataset_name,
	'amount_must_be_positive' as rule_name,
	'blocking' as severity,
	count(*) as invalid_rows,
	current_timestamp as checked_at
from gold.payments_daily
where amount <= 0;
```

Once results are stored, quality becomes observable over time. You can trend failures, alert on regression, and prove whether a fix actually improved the dataset.

## Freshness Is A Quality Dimension

Data can be correct and still be useless if it is late.

Freshness should be part of the contract, especially for operational analytics and AI workflows. A support agent using a customer summary does not only need accurate data. It needs recent data. A fraud workflow does not only need clean transactions. It needs them within a time window where action still matters.

Freshness checks should be explicit:

- Last successful load timestamp.
- Maximum accepted lag.
- Expected arrival cadence.
- Volume compared with historical baseline.
- Consumer-specific freshness requirements.

The same dataset may have different freshness expectations depending on the consumer. A monthly finance report and a real-time operational alert should not share the same quality bar.

A simple freshness query is often enough to start:

```sql
select
	max(processed_at) as latest_processed_at,
	extract(epoch from (current_timestamp - max(processed_at))) / 60 as lag_minutes
from gold.payments_daily;
```

The important decision is what happens when `lag_minutes` exceeds the contract. If it only shows up in a dashboard nobody watches, it is not a control.

## Observability Is Not Optional

Data observability should explain what changed, not only that something failed.

At minimum, I want to know:

- Row counts by partition.
- Null rates for important fields.
- Distribution shifts.
- Schema changes.
- Freshness lag.
- Failed validation rules.
- Upstream source delays.
- Downstream consumers affected.

Without this, every incident starts with manual archaeology. Engineers query tables, compare yesterday with today, inspect orchestrator logs, and ask whether the source system changed. That is slow and expensive.

Good observability turns the question from "what happened?" into "what do we do next?"

## Failure Handling Is Part Of The Design

A quality rule without a failure strategy is incomplete.

When validation fails, the system should know whether to:

- Stop publishing.
- Quarantine bad records.
- Publish with warnings.
- Preserve the last known good dataset.
- Alert the owner.
- Trigger a backfill.
- Block downstream jobs.

Not all failures deserve the same response. A minor optional-field null increase might be a warning. A broken primary key should probably block publication. A missing partition might require preserving the previous valid snapshot.

The important part is deciding this before the incident.

## Ownership Is A Technical Requirement

Data quality problems often survive because ownership is vague.

If a dataset has no owner, every issue becomes a negotiation. The analytics team blames the pipeline, the pipeline team blames the source, and the source team says nothing changed.

A mature data product has:

- A technical owner.
- A business owner or domain expert.
- Documented consumers.
- Expected SLAs or SLOs.
- A known escalation path.

Ownership is not bureaucracy. It is how the system recovers.

## What Senior Data Engineering Looks Like

The senior move is not adding hundreds of checks. The senior move is choosing the right contracts and making them operational.

That means:

- Validating business-critical fields first.
- Designing checks that produce actionable failures.
- Separating warnings from blockers.
- Keeping validation close to the layer it protects.
- Measuring quality trends, not only binary pass/fail states.
- Making quality visible to consumers.

It also means accepting tradeoffs. Overly strict validation can block useful data. Weak validation can publish misleading data. The right design depends on the cost of being wrong.

## Takeaways

Data quality is not a side effect of clean code. It is a product capability.

If a dataset is important, it deserves a contract. If it has a contract, it deserves validation. If validation can fail, it deserves observability and ownership. That is how pipelines become reliable systems instead of scheduled scripts.

The goal is not perfect data. The goal is trustworthy data with known boundaries, visible failures, and clear recovery paths.
