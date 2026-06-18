---
title: 'Harness Engineering, Part 1'
seoTitle: 'Harness Engineering for AI Agents, Part 1: Control Layers and Tool Contracts'
description: 'A production AI agent needs a control layer that owns permissions, tool contracts, validation, execution, and confirmations.'
pubDate: 'Jun 18 2026 10:00'
heroImage: '../../assets/blog/harness-engineering-control-systems-part-1/hero.png'
tags:
  - AI agents
  - Harness engineering
  - Tool contracts
  - Guardrails
---

Most AI agent demos optimize for the wrong moment: the instant where the model produces an impressive answer. Production systems fail in the moments after that. They fail when the answer becomes an action, when the action touches real data, when a retry repeats a side effect, when a tool returns stale information, or when nobody can explain why the agent made a decision.

That is why I think about agents less as "smart prompts" and more as controlled execution systems. The model is one component. The harness is the system around it that decides what can be done, how it is validated, how it is observed, and when it must stop.

This first part is about the control layer: what the harness owns, how tool contracts should be shaped, and why the model should reason but not execute directly.

## The Problem With Agent Demos

A demo usually has a narrow path:

- A user asks a clear question.
- The model chooses a tool.
- The tool returns clean data.
- The model writes a polished response.

Real systems are not that clean. Inputs are incomplete, permissions are contextual, tools fail, external APIs drift, users ask for things they should not be allowed to do, and the model can be confidently wrong.

The dangerous part is that a bad agent can look successful. It can produce fluent text, call a real tool, and return a plausible result while violating the system contract. Traditional backend services usually fail loudly when a schema changes or a dependency is down. Agents can fail semantically while every HTTP request returns `200`.

That means we need a different control surface.

## What A Harness Owns

The model should not own execution. The harness should.

A practical harness owns:

- Tool registry and tool permissions.
- Input normalization and schema validation.
- Policy checks before tool execution.
- Idempotency and retry behavior.
- Output validation before user delivery.
- Audit logs and trace correlation.
- Cost limits and timeout boundaries.
- Evaluation datasets and regression checks.

The model can propose an action. The harness decides if that action is legal, safe, and useful.

```ts
type AgentProposal = {
	tool: string;
	input: unknown;
	reason: string;
};

type HarnessDecision =
	| { status: 'approved'; normalizedInput: unknown }
	| { status: 'rejected'; reason: string };

type ToolResult =
	| { status: 'ok'; data: unknown; traceId: string }
	| { status: 'failed'; errorCode: string; traceId: string };
```

This is not very different from backend engineering. We already validate API inputs, authorize operations, wrap dependencies, log execution, and protect state changes. The difference is that the caller is now a model that can synthesize parameters instead of a deterministic client.

## The Boundary Between Reasoning And Execution

One useful rule is this: the model can reason, but the harness must execute.

If the model says, "delete this record", the harness should ask:

- Is this tool available to this agent?
- Is the user allowed to perform this operation?
- Does the input match the tool schema?
- Is the target resource within the user's tenant or scope?
- Is the operation idempotent?
- Does it require confirmation?
- What trace should be attached to this action?

Only after those checks should anything happen.

This boundary also makes the system easier to debug. When something fails, you can inspect whether the problem was reasoning, policy, tool execution, data freshness, or output formatting. Without that separation, the failure becomes "the agent was wrong", which is not actionable.

## AGENTS.md As A Small Harness

In this repository, `AGENTS.md` is a small example of a harness around an agent. It does not evaluate model quality or execute tools by itself, but it defines operational constraints:

- Stay inside the project context.
- Warn before structural or functional changes.
- Respect branch and integration rules.
- Validate before finishing.
- Follow content and design contracts.

That is the same pattern at a smaller scale. The agent can propose and implement, but the harness defines boundaries. For a product agent, those boundaries become code, policies, tool contracts, evals, and logs.

## Tool Contracts Matter

Tools should be designed like internal APIs. A weak tool contract gives the model too much room to improvise.

Bad tool shape:

```ts
runQuery(input: string): Promise<unknown>;
```

Better tool shape:

```ts
type CustomerLookupInput = {
	customerId: string;
	includeAccounts: boolean;
};

lookupCustomer(input: CustomerLookupInput): Promise<CustomerSnapshot>;
```

The second contract limits ambiguity. It also gives you something to validate. The model should not be constructing arbitrary SQL or inventing undocumented flags unless the system has explicitly been designed for that use case.

For data and backend workflows, I prefer tools that are narrow, typed, observable, and permission-aware. A tool should do one thing well and return a result that the next layer can verify.

The harness can enforce that contract before a tool ever runs:

```ts
type ToolContext = {
	userId: string;
	roles: string[];
	tenantId: string;
	traceId: string;
};

type ToolDefinition<TInput, TOutput> = {
	name: string;
	allowedRoles: string[];
	validate(input: unknown): TInput;
	execute(input: TInput, context: ToolContext): Promise<TOutput>;
};

async function runTool<TInput, TOutput>(
	tool: ToolDefinition<TInput, TOutput>,
	rawInput: unknown,
	context: ToolContext,
) {
	if (!tool.allowedRoles.some((role) => context.roles.includes(role))) {
		return { status: 'rejected' as const, reason: 'missing_role', traceId: context.traceId };
	}

	const input = tool.validate(rawInput);
	const data = await tool.execute(input, context);

	return { status: 'ok' as const, data, traceId: context.traceId };
}
```

This is intentionally ordinary code. The harness should be boring enough that you can trust it more than the model output.

## Confirmations Are Part Of The System

For destructive operations, I would make confirmation part of the state machine:

```ts
type WorkflowState =
	| { step: 'draft'; proposal: AgentProposal }
	| { step: 'needs_confirmation'; message: string; proposal: AgentProposal }
	| { step: 'executed'; traceId: string }
	| { step: 'rejected'; reason: string };

function requireConfirmation(proposal: AgentProposal): WorkflowState {
	const destructiveTools = new Set(['delete_customer', 'refund_payment', 'send_external_email']);

	if (destructiveTools.has(proposal.tool)) {
		return {
			step: 'needs_confirmation',
			message: `Confirm execution of ${proposal.tool}`,
			proposal,
		};
	}

	return { step: 'draft', proposal };
}
```

This kind of explicit state is less flashy than an autonomous demo, but it is how you avoid pretending that every action has the same risk profile.

## Takeaways

The first layer of harness engineering is control. The model should be able to reason, but the system around it should own execution.

If the harness owns tool contracts, permissions, validation, and confirmation boundaries, the agent can gain capability without gaining unlimited authority.

Part 2 is about how to evaluate and operate that system after the control layer exists.
