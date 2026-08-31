## ADDED Requirements

### Requirement: Visitors see clear poll entry paths
The system SHALL render a landing page at the frontend root that presents the
product name, the promise “Ask one good question. Hear every voice.”, a primary
Create a poll action, and a distinct Join a poll form.

#### Scenario: Visitor opens the landing route
- **WHEN** a visitor opens the frontend root route
- **THEN** they see the product identity, a create-poll action, and a labelled
  control for entering a poll ID or link

#### Scenario: Visitor chooses to create a poll
- **WHEN** a visitor activates the Create a poll action
- **THEN** the browser navigates to `/polls/new`

### Requirement: Visitors can join using an ID or poll URL
The system SHALL accept a non-empty poll identifier or a complete URL whose
path identifies a poll, and SHALL navigate to the matching `/polls/:pollId`
destination after submission.

#### Scenario: Visitor joins with an identifier
- **WHEN** a visitor submits a trimmed non-empty poll ID
- **THEN** the browser navigates to `/polls/<poll-id>`

#### Scenario: Visitor joins with a complete poll URL
- **WHEN** a visitor submits a URL ending in `/polls/<poll-id>`
- **THEN** the system extracts the poll ID and navigates to `/polls/<poll-id>`

#### Scenario: Visitor submits invalid join input
- **WHEN** a visitor submits an empty value or a URL that does not identify a
  poll
- **THEN** the page remains in place and displays an inline error associated
  with the join field

### Requirement: Landing interactions are responsive and accessible
The system SHALL provide semantic landmarks, visible labels, keyboard-visible
focus, and controls with a minimum 44 px interactive height. The layout SHALL
start as a single-column mobile experience and adapt to show the hero and join
card side by side when viewport space permits.

#### Scenario: Keyboard user navigates the landing page
- **WHEN** a keyboard user tabs through the page
- **THEN** the create action and join controls have a visible focus indicator
  and the join input retains a visible programmatic label

#### Scenario: Visitor uses a narrow viewport
- **WHEN** the viewport is 320 px wide
- **THEN** the content remains readable without horizontal scrolling and the
  create action appears before the join form

### Requirement: Landing styles use Tailwind CSS
The system SHALL use Tailwind CSS v4 with semantic theme variables for landing
page layout, colour, typography, responsive behavior, interaction states, and
reduced-motion behavior. Authored CSS SHALL be limited to global base rules
and decorative pseudo-elements or artwork that cannot be reasonably expressed
with Tailwind utilities.

#### Scenario: Frontend build processes the landing styles
- **WHEN** the frontend production build runs
- **THEN** Tailwind CSS processes the landing page utilities and the generated
  stylesheet includes the semantic product theme
