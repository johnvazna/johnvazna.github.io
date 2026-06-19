---
title: 'Supabase with Hexagonal Architecture'
seoTitle: 'Supabase with Hexagonal Architecture: Postgres, RLS, and Clean Data Boundaries'
description: 'Supabase is strongest when treated as a Postgres application platform with clear domain boundaries, repositories, RLS, and schema ownership.'
pubDate: 'Feb 11 2026'
heroImage: '../../assets/blog/supabase-hexagonal-data-structures/hero.png'
tags:
  - Supabase
  - Hexagonal architecture
  - Postgres
  - RLS
---

Supabase makes it easy to build quickly, but speed can hide architectural debt. If every screen knows the exact table shape, every schema change becomes a frontend migration. If authorization lives only in UI conditions, security becomes a convention. If business rules are scattered across client code, the database stops being a system of record and becomes a shared spreadsheet with APIs.

That is not a Supabase problem. It is an architecture problem.

Supabase is strongest when treated as a Postgres application platform. Postgres owns persistence and integrity. Row Level Security owns data access boundaries. Edge Functions can coordinate privileged workflows. The application still needs a domain model and clean adapters.

That is where hexagonal architecture helps.

## Why Hexagonal Architecture Fits Supabase

Hexagonal architecture, also called ports and adapters, separates the domain from infrastructure. The domain defines what the application means. Adapters define how the application talks to databases, APIs, queues, storage, and external systems.

In a Supabase project, this separation matters because Supabase gives the frontend a very convenient path to the database. That convenience is useful, but it should not become the only architecture.

For small CRUD surfaces, direct Supabase queries from the client can be acceptable. For products with rules, permissions, workflows, billing, teams, or audit requirements, direct table coupling becomes expensive.

The question is not "can the frontend query this table?" The question is "should this table shape be the application contract?"

I still use Supabase client APIs where they make sense. The difference is that I do not let every component invent its own data access pattern:

```ts
// UI code should express intent, not database shape.
const projects = await projectService.listWorkspaceProjects({
	workspaceId,
	actorId: currentUser.id,
});
```

The service can then decide whether this reads from a table, view, RPC function, or Edge Function. That decision belongs to the adapter layer, not every screen.

## The Risk of Table-Driven UI

A table-driven UI often starts clean:

```ts
const { data } = await supabase
	.from('projects')
	.select('*')
	.eq('owner_id', user.id);
```

This is fine for a prototype. The problem appears when the product grows:

- The UI depends on column names.
- Business rules are duplicated across screens.
- Access rules are assumed by the client.
- Joins become embedded in component logic.
- Schema changes require coordinated frontend rewrites.
- Tests need real database state for basic domain behavior.

At that point, the database has leaked into the product surface.

## A Better Boundary

A cleaner structure introduces a port:

```ts
type ProjectRepository = {
	findVisibleProjects(userId: string): Promise<ProjectSummary[]>;
	createProject(input: CreateProjectInput): Promise<Project>;
	archiveProject(projectId: string, actorId: string): Promise<void>;
};
```

The UI does not need to know whether `findVisibleProjects` reads from a table, a view, an RPC function, or an Edge Function. The adapter can change without rewriting the domain or presentation layer.

This does not mean hiding SQL because SQL is bad. SQL is a strength. The point is to avoid making raw persistence details the only interface.

An adapter can still be thin:

```ts
class SupabaseProjectRepository implements ProjectRepository {
	constructor(private readonly client: SupabaseClient) {}

	async findVisibleProjects(userId: string) {
		const { data, error } = await this.client
			.from('project_summaries')
			.select('id,name,status,updated_at')
			.eq('member_id', userId)
			.order('updated_at', { ascending: false });

		if (error) throw new Error(`project_lookup_failed:${error.code}`);

		return data.map((row) => ({
			id: row.id,
			name: row.name,
			status: row.status,
			updatedAt: new Date(row.updated_at),
		}));
	}
}
```

The repository is not adding ceremony for ceremony's sake. It is translating infrastructure shape into application shape.

## Where Rules Should Live

One senior architecture decision is placing rules in the right layer.

Postgres should own data integrity:

- Primary keys.
- Foreign keys.
- Unique constraints.
- Check constraints.
- Required fields.
- Transaction boundaries.

RLS should own row-level access:

- Tenant isolation.
- User ownership.
- Role-based visibility.
- Access to shared resources.

Domain services should own workflow rules:

- Can this user archive this project?
- Is this status transition allowed?
- Should this action create an audit event?
- Does this operation require a notification?

Edge Functions can own privileged orchestration:

- Admin operations.
- Multi-step workflows.
- Secure calls to third-party APIs.
- Operations requiring service-role access.

The UI should own interaction and presentation, not enforcement.

## RLS Is a Boundary, Not a Feature Toggle

Row Level Security is one of the most important parts of Supabase architecture. It should not be treated as something added at the end.

If a table is exposed to client-side access, RLS is the database boundary that prevents users from reading or writing rows outside their scope. Without it, the application is trusting the client to behave.

A simple ownership policy might look like this:

```sql
alter table public.projects enable row level security;

create policy "Users can read their own projects"
on public.projects
for select
to authenticated
using (
	auth.uid() is not null
	and owner_id = auth.uid()
);
```

That policy is not just a security rule. It is part of the application contract.

For more complex systems, policies may need team membership, roles, organizations, or entitlement tables. The important part is that access is enforced where the data lives.

For a team-based product, the policy usually needs membership:

```sql
create policy "Workspace members can read projects"
on public.projects
for select
to authenticated
using (
	exists (
		select 1
		from public.workspace_members wm
		where wm.workspace_id = projects.workspace_id
		  and wm.user_id = auth.uid()
	)
);
```

I would still add explicit filters in application queries for performance and clarity. RLS is the security boundary, not an excuse to query every row and let the database clean it up.

## Views, RPC, and Read Models

Not every frontend read needs to map directly to a base table.

For stable read models, views or RPC functions can reduce coupling:

- A dashboard card can read from a view.
- A search screen can call an RPC function.
- A summary page can use a denormalized read model.
- A privileged workflow can go through an Edge Function.

This lets the database evolve while preserving a stable consumer contract. It also makes performance tuning easier because the data access pattern becomes explicit.

The tradeoff is that every abstraction needs ownership. A view without security awareness can become a bypass. A function with broad privileges can become a risk. The architecture is stronger only when the boundaries are intentional.

For Postgres 15 and above, a read model that should respect caller permissions can use `security_invoker`:

```sql
create view public.project_summaries
with (security_invoker = true)
as
select
	p.id,
	p.workspace_id,
	wm.user_id as member_id,
	p.name,
	p.status,
	p.updated_at
from public.projects p
join public.workspace_members wm
	on wm.workspace_id = p.workspace_id;
```

That view still needs a clear access model. If the view becomes public API for the app, treat it like a contract.

## Data Structure as Product Design

Data modeling is product design. The schema decides what the product can express safely.

For Supabase, I would model around domain concepts, not UI components. A good schema usually has:

- Tables named around business entities.
- Explicit join tables for relationships.
- Constraints for invariants.
- Timestamps for auditability.
- Status fields with controlled transitions.
- RLS policies aligned with the access model.
- Migrations that describe intent.

If the schema only mirrors screens, it will break when the product changes. If the schema models the domain, the UI can evolve around it.

## Testing the Boundary

A Supabase architecture should be tested at multiple levels:

- Unit tests for domain services.
- Repository tests for database adapters.
- SQL tests or migration checks for constraints.
- RLS tests for access boundaries.
- Integration tests for Edge Functions.

The most important tests are often negative tests:

- A user cannot read another tenant's row.
- A user cannot update a row without the required role.
- A status transition fails when invalid.
- A privileged operation cannot be called from the public client.

These tests prove the architecture, not just the happy path.

## A Practical Project Shape

For a TypeScript app, I like this kind of separation:

```text
src/
  domain/
    projects/
      project.ts
      project-service.ts
      project-repository.ts
  infrastructure/
    supabase/
      supabase-project-repository.ts
      supabase-client.ts
  app/
    actions/
    routes/
    components/
```

The exact folders matter less than the dependency direction. The domain should not import the Supabase client. The Supabase adapter can implement domain ports.

That keeps the database powerful without making every layer depend on it directly.

## Takeaways

Supabase is not just a quick backend. It is a serious Postgres platform when used with discipline.

The senior approach is not avoiding Supabase features. It is placing them correctly. Use Postgres constraints for integrity, RLS for access, Edge Functions for privileged orchestration, and domain ports for application boundaries.

When those pieces are aligned, Supabase can support fast product development without turning the frontend into a thin wrapper around table shapes.
