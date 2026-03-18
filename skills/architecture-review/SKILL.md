---
name: Architecture Review
description: >
  Analyzes structural health of a codebase: dependency direction, module boundaries,
  layering violations, API consistency, and abstraction quality. Finds the
  architectural drift that accumulates sprint over sprint.
category: architecture
---

# Architecture Review

Analyze the structural health of the codebase. Good architecture makes change easy and
safe. Bad architecture makes every change a risk. This analysis finds the places where
the intended architecture has drifted — where boundaries are violated, dependencies point
the wrong way, and patterns are inconsistent.

## Analysis Strategy

### 1. Discover the intended architecture
- Read project configuration: directory structure, package boundaries, module organization
- Look for architectural documentation: ADRs, ARCHITECTURE.md, module READMEs
- Infer the layering from directory names and import patterns: is there a domain/core layer? An infra/adapter layer? A presentation/API layer?
- Identify the dependency rule: what should depend on what?

### 2. Map actual dependencies
- For each module/package/directory, trace its imports
- Build a mental model of the dependency graph: which modules depend on which?
- Look for violations: domain importing from infra, core importing from UI, circular dependencies
- Check: do module boundaries match the conceptual boundaries?

### 3. Assess pattern consistency
- How are similar things done across the codebase? (error handling, data validation, API responses, logging)
- Are there multiple patterns for the same concern? (3 different ways to validate input)
- Which pattern is the "intended" one vs. legacy/accidental divergence?

### 4. Evaluate API design
- For HTTP APIs: are naming conventions consistent? Are error responses standardized?
- For internal APIs (module interfaces): are they narrow and well-defined, or do modules reach into each other's internals?
- Check for breaking change risks: are contracts explicit (schemas, types) or implicit?

## What to Look For

### Dependency Direction Violations
- **Domain depending on infrastructure**: Business logic importing database drivers, HTTP frameworks, or cloud SDK types. The domain layer should define interfaces that infra implements
- **Core depending on presentation**: Shared/core code importing React components, API route handlers, or CLI-specific code
- **Library depending on application**: Utility/shared packages importing from the application that uses them
- **How to find**: Start from the innermost layer (domain/core). Trace every import. Any import from an outer layer is a violation

### Circular Dependencies
- Module A imports from Module B which imports from Module A (directly or transitively)
- Packages that reference each other in their dependency lists
- Files that use `require` instead of `import` to work around circular import issues
- **Impact**: Makes modules impossible to understand, test, or deploy independently

### Module Boundary Violations
- Code reaching into another module's internal files instead of using its public API/index
- Importing from `../other-module/internal/helper` instead of `../other-module`
- Database models used directly across module boundaries instead of through a service layer
- Shared mutable state (global variables, singletons) coupling unrelated modules

### Layering Violations
- Controllers/handlers containing business logic instead of delegating to a service/use-case layer
- Business logic directly constructing HTTP responses, SQL queries, or framework-specific objects
- Data access mixed with presentation formatting in the same function
- Validation split inconsistently between layers (some in controller, some in service, some in model)

### Abstraction Problems
- **Missing abstraction**: The same 10-line pattern copy-pasted across 5+ locations with minor variations
- **Premature abstraction**: A generic framework built for one use case, adding complexity without reuse
- **Leaky abstraction**: An interface that exposes implementation details (e.g., a "repository" that returns ORM-specific query builders)
- **Wrong abstraction**: An inheritance hierarchy that forces unrelated things to share behavior, or a utility that handles too many concerns

### API Design Issues
- **Inconsistent naming**: Mix of camelCase and snake_case in the same API surface, inconsistent pluralization, inconsistent verb usage (get/fetch/retrieve)
- **Inconsistent response shapes**: Some endpoints return `{ data: ... }`, others return the object directly, others return `{ result: ... }`
- **Inconsistent error handling**: Some endpoints return errors in body with 200, others use proper HTTP status codes
- **Missing versioning**: Breaking API changes with no versioning strategy
- **Overly chatty**: Client needs 5 API calls to render a single view (suggests missing aggregation endpoint)

### Configuration and Environment
- Hardcoded values that should be environment-specific (URLs, feature flags, limits, timeouts)
- Configuration scattered across code instead of centralized
- Environment-specific code branches (`if (process.env.NODE_ENV === 'production')`) deep in business logic
- Missing validation of required configuration at startup (app starts but crashes later when config is accessed)

### Dead and Orphaned Code
- Exported functions/classes with zero importers (excluding public API surfaces)
- Feature flags that are always on/off with no mechanism to clean up
- Commented-out code blocks committed to the repository
- Entire files/modules that are unreachable from any entry point
- Database tables or API endpoints that nothing references

## Severity Classification

| Severity | Criteria |
|----------|----------|
| **critical** | Circular dependency between core modules that prevents independent deployment or testing. Missing abstraction causing bugs to recur across 5+ locations. |
| **high** | Domain layer depending on infrastructure (makes the domain untestable without mocking infra). Major API inconsistencies visible to consumers. Significant dead module still receiving maintenance. |
| **medium** | Module boundary violations that increase coupling. Inconsistent patterns across the codebase. Configuration scattered through business logic. |
| **low** | Minor naming inconsistencies. Small amounts of dead code. Slightly leaky abstractions. |

## Reporting

Use these rule ID prefixes:

- `architecture/dependency-violation` — Wrong-direction imports, domain depending on infra
- `architecture/circular-dependency` — Direct or transitive circular imports between modules
- `architecture/boundary-violation` — Reaching into module internals, bypassing public API
- `architecture/layer-violation` — Business logic in controllers, SQL in handlers, etc.
- `architecture/missing-abstraction` — Duplicated patterns that should be unified
- `architecture/premature-abstraction` — Unnecessary complexity for a single use case
- `architecture/api-inconsistency` — Inconsistent naming, response shapes, or error handling
- `architecture/dead-code` — Unreachable modules, unused exports, stale feature flags
- `architecture/config-sprawl` — Hardcoded values, scattered configuration, missing validation

For each rule **`shortDescription`**, write a concise 1-sentence title (e.g. "Circular dependency between auth and user modules").

For each rule **`fullDescription`**, explain:
1. What the structural issue is and which modules/layers are involved
2. Why it matters (what becomes harder: testing, changing, deploying, understanding)
3. What the intended structure likely is, based on the rest of the codebase

For each result **`message`**, provide instance-specific context — mention the file, function, or import involved.
