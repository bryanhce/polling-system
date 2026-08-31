## Why

The frontend is still the default Vite starter screen, so visitors cannot begin
using the polling product. A focused landing page is needed to make the two
core entry points—creating or joining a poll—clear, friendly, and accessible.

## What Changes

- Replace the Vite starter UI at the frontend root route with the Aethelgard
  Voice landing experience described in `frontend/DESIGN.md`.
- Add a prominent path to create a poll and a separate poll ID/link entry form
  that validates and normalizes a pasted poll URL before navigating.
- Establish the product's responsive visual foundation: warm canvas, rounded
  surfaces, accessible focus states, and non-essential decorative artwork using
  Tailwind CSS v4 utilities and theme variables.

## Capabilities

### New Capabilities

- `landing-page`: Provides the responsive product landing page and its create
  and join-poll entry paths.

### Modified Capabilities

- None.

## Impact

- Affects `frontend/src/App.tsx`, `frontend/src/index.css`, and the Vite
  configuration; the landing-page CSS is migrated to Tailwind
  utilities and small purpose-built CSS only where pseudo-elements require it.
- Adds `tailwindcss` and `@tailwindcss/vite` to the frontend development tooling
  and configures the Tailwind Vite plugin.
- Adds client-side navigation behavior for `/polls/new` and `/polls/:pollId`;
  it does not require backend API changes.
