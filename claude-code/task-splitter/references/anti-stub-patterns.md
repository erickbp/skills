# Anti-Stub Patterns

AI agents commonly produce files that exist but contain placeholder implementations. Use this reference when writing artifact constraints in Must-Haves to catch stubs during verification.

## Universal Stub Indicators (any file type)

- Comment placeholders: `TODO`, `FIXME`, `PLACEHOLDER`, `implement later`, `coming soon`, `...`
- Empty/trivial returns: `return null`, `return {}`, `return []`, `pass`, `...`
- Hardcoded values where dynamic data is expected: string IDs, fixed counts, static display values
- Log-only functions: handler bodies that only `console.log` or `print` without real logic

## Frontend Component Stubs

- Returns trivial JSX: `<div>Component</div>`, `<div>Placeholder</div>`, `<></>`, or `null`
- Empty event handlers: `onClick={() => {}}`, `onSubmit` that only calls `preventDefault`
- No state or props usage: renders static content with no dynamic behavior
- No data fetching: data-display components with no fetch/query calls

## API Route Stubs

- Returns static responses: `{ message: "Not implemented" }`, empty arrays with no database query
- No database/service interaction: handler body has no query, create, update, or delete calls
- No input validation: accepts request body without parsing or schema validation
- Console-log-only handlers: logs the request and returns `{ ok: true }`

## Wiring Stubs (components exist but aren't connected)

- Component never imported or rendered in any route/page
- API route exists but no frontend code calls it
- Form `onSubmit` never calls API or mutation
- State declared but never rendered in output
- Database query result fetched but not returned in response

## How to Use During Planning

When a task creates files that are high-risk for stubbing (API routes, data-fetching components, business logic modules), include at least one substance constraint in the Artifacts field of Must-Haves. Good constraints: `min_lines: 30`, `contains: prisma.*.findMany`, `exports: [handleSubmit, validateInput]`. The constraint should match the real work, not just file existence.
