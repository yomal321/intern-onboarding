# ADR 0001: Use ADRs to Record Architecture Decisions

## 1. Status (current decision state)

Accepted

Date: 2026-05-28

## 2. Context (why this decision was needed)

The GreenChit team is making many technical decisions.

Examples:

- hosting platform
- database choice
- file upload method
- audit log design
- authentication strategy

Without written records:

- people may forget the reason
- team members may remember different things
- future developers may not understand old choices
- design review becomes harder

## 3. Decision (what the team chose)

The team will write an ADR for every major technical decision.

ADR files will be stored in:

`adrs/`

ADR files will be numbered in order:

- `0001`
- `0002`
- `0003`

The team will use the Nygard ADR format.

Each ADR should include:

- Status
- Context
- Decision
- Consequences
- Alternatives considered

## 4. Consequences (benefits and risks)

Benefits:

- easier to explain decisions during review
- easier to remember why choices were made
- easier for new team members to understand the system
- clearer decision history

Risks:

- writing ADRs takes extra time
- documentation must be maintained
- the deadline is short

## 5. Alternatives Considered (other options rejected)

Option 1: Write nothing down

Rejected because:

- people forget
- decisions become unclear later
- the team may not remember why choices were made

Option 2: Use a Confluence page

Rejected because:

- wiki pages are edited over time
- original decision history can be lost
- ADR files keep decisions clearer

## 6. Simple Summary (one-line recap)

GreenChit will use numbered ADR files to record important architecture decisions and the reasons behind them.
