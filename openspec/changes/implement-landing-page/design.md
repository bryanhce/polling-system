## Context

The frontend root route currently renders the Vite starter screen. The product
design already specifies the landing route as the first experience for
Aethelgard Voice: visitors must be able to create a poll or enter an existing
poll ID/link. The project uses React, TypeScript, and Vite without a routing
library. The landing page uses Tailwind CSS v4 for shared layout and component
styling, with authored CSS reserved for its complex decorative artwork.

## Goals / Non-Goals

**Goals:**

- Deliver the responsive, accessible landing experience specified in
  `frontend/DESIGN.md`.
- Make creation the visually primary route while giving joining equal practical
  visibility.
- Normalize supported pasted poll links into a poll ID before navigation.
- Establish Tailwind CSS v4 as the frontend styling system while keeping the
  implementation buildable through Vite.

**Non-Goals:**

- Building the create-poll or poll-detail screens, or calling backend APIs.
- Adding a client-side routing package, authentication, dashboards, or
  real-time result behavior.
- Replacing the broader frontend architecture before its subsequent pages are
  implemented.

## Decisions

### Render the landing page directly in `App`

`App` will become the landing-page composition with semantic header, main, and
footer landmarks. This replaces only starter content and keeps the initial
implementation simple. A routing package was considered, but the app has no
existing route structure and this change only needs client-side destinations;
standard anchor URLs provide those destinations without a dependency.

### Implement navigation with URL destinations and local join validation

The create CTA will be an anchor to `/polls/new`. The join form will accept a
plain ID or a URL whose path ends in `/polls/<id>`, trim input, and navigate to
`/polls/<id>` after validation. Invalid inputs will remain in place with an
inline, programmatic error. Backend validation is deliberately deferred to the
destination route, as required by the product design.

### Use Tailwind CSS v4 with the Vite plugin and semantic theme variables

The frontend installs `tailwindcss` and `@tailwindcss/vite`, registers the
Tailwind Vite plugin alongside the React plugin, and imports Tailwind from the
global stylesheet. Semantic product tokens are defined as Tailwind theme
variables, allowing components to use meaningful utilities such as
`bg-canvas`, `text-ink`, and `bg-primary` instead of repeating raw colours.

Layout, spacing, typography, responsive variants, state styles, and reduced-
motion variants live in component utility classes. The global stylesheet
will be limited to the Tailwind import, theme variables, and truly global base
rules. A small stylesheet is permitted only for decorative pseudo-elements or
complex artwork that would be materially less maintainable as utilities.
Decorative shapes remain `aria-hidden`, avoiding image assets and preserving
performance. A component library was considered but would add an unneeded
dependency for the page's small set of native controls.

### Prioritize native accessibility primitives

The join control will use a visible `<label>`, `aria-invalid`, and
`aria-describedby` when needed. Links and form controls will meet a 44 px
minimum height and use a prominent focus-visible ring. This is more robust than
custom clickable containers and works without additional JavaScript behavior.

## Risks / Trade-offs

- [Destination routes are not yet implemented] → Links and submitted join URLs
  can lead to Vite's fallback during development; this landing change confines
  itself to producing the correct future routes.
- [An arbitrary ID format could create unusable URLs] → Only accept trimmed,
  non-empty identifier-safe text and known poll URL paths; poll existence is
  validated by the future destination screen.
- [Decorative elements can crowd narrow screens] → Use absolute, hidden-from-
  assistive-technology shapes that are reduced or hidden at small widths.
- [Utility-heavy markup becomes hard to scan] → Group repeated patterns in
  React components and use a limited, documented CSS escape hatch only for
  pseudo-elements and complex decorative art.
- [Remote display fonts can delay rendering] → Use a local system rounded
  display-font fallback stack rather than adding a network font dependency.

## Migration Plan

1. Tailwind CSS and its Vite plugin are installed and registered, and the
   global stylesheet imports Tailwind.
2. Product tokens are defined as Tailwind theme variables; landing-page layout,
   typography, and responsive styles use utilities.
3. The CSS escape hatch is retained only for decorative pseudo-elements, and
   the frontend production build and lint checks run before deployment.
4. Deploy as a static frontend update; rollback consists of restoring the
   previous frontend bundle because no persisted data or API contract changes
   are involved.

## Open Questions

- None for the landing-page scope. The create and poll-detail routes will be
  implemented as follow-up changes.
