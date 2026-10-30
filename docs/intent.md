# Intent - Cart and Pay-on-Collection Ordering

Status: Partially approved - F1 approved; F2-F6 awaiting approval
Approved by: User in this conversation (F1 only; name not supplied)
Date: 2026-10-30

## Sources

- **S1:** [Shopper feedback and owner notes](inputs/shopper-feedback.md),
  referenced below by the existing FB-01 through FB-10 and ON-01 through ON-05 IDs.
- **S2:** [Project constraints](../AGENTS.md), especially Guardrails and Conventions.
- **S3:** [Domain glossary](glossary.md), especially Stock, Cart, Line item,
  Checkout, and Order.
- **S4:** [ADR 0001](adr/0001-in-memory-store.md), the accepted in-memory-only
  storage constraint.

## Problem

Shoppers can browse the catalogue but cannot buy through the site. Ordering by
email fragments requests, makes quantities and totals unclear, and leaves
shoppers uncertain whether an order succeeded. The owner spends around an hour
a day handling these threads and sometimes promises stock that is unavailable.

## Pain Points

| ID | Pain point | Evidence |
| --- | --- | --- |
| P1 | Shoppers cannot place a single on-site order for their selected books; the owner handles error-prone email threads. | S1: FB-01, FB-02, ON-01 |
| P2 | Quantities are difficult to change or verify, and accidental repeated selections can cause unwanted copies. | S1: FB-04, FB-10 |
| P3 | Shoppers learn too late that a book is unavailable, low stock is easy to miss, and the owner oversells. | S1: FB-03, FB-06, ON-01, ON-03 |
| P4 | Shoppers cannot review the complete amount before committing. | S1: FB-05 |
| P5 | Shoppers receive no clear confirmation or record of what they ordered. | S1: FB-07 |
| P6 | Shoppers who remember only an author struggle to find books. | S1: FB-08 |
| P7 | Gift buyers want wrapping and a note, but the owner prioritizes basic purchasing first. | S1: FB-09, ON-05 |

## Desired Outcomes

- A shopper can select several books and place one order without an email
  exchange or account registration.
- A shopper can deliberately review and change quantities before ordering.
- Shoppers see availability before committing, and accepted orders never cause
  stock to become negative.
- A shopper sees the complete amount due before checkout and receives a clear
  confirmation afterward.
- The owner has an order record for each successful checkout during the current
  server run, reducing reliance on fragmented email requests.

No numerical improvement target is established by the sources. The owner's
reported hour per day is a baseline observation, not an agreed success target.

## Constraints

- Payment is on collection; no online card payment or payment provider
  (S1: ON-02; S2).
- No customer accounts, authentication, or sessions (S1: ON-04; S2).
- Stock must never fall below zero (S1: ON-03).
- Keep the existing Express and TypeScript backend, vanilla-JS frontend, and
  in-memory store. No database, new dependencies, CSS framework, or build
  tooling; styling remains in the existing single stylesheet (S2, S4).
- Cart and order state need not survive a server restart. Restarting resets
  in-memory data; durable retention is not promised (S4).
- Use a returned cartId for the account-free cart, as required by the existing
  project constraints; this document does not prescribe additional API or data
  model design (S2).
- Later delivery must be phased, starting with a minimal vertical tracer bullet
  across route, store, and tests. Phase completion requires fresh test,
  typecheck, and lint evidence (S2).

## Features

F1's requirements are approved. Priorities for the remaining features are
proposals for approval, not recorded stakeholder agreement.
Core purchasing comes before gift services, as explicitly requested in ON-05.
Feature IDs are references, not a mandatory implementation order.

| ID | Feature | Solves | Proposed priority |
| --- | --- | --- | --- |
| F1 | Select and manage a cart | P1, P2 | Core purchase scope |
| F2 | Show availability and prevent overselling | P3 | Core purchase scope |
| F3 | Review the complete order total | P4 | Core purchase scope |
| F4 | Place and confirm a pay-on-collection order | P1, P5 | Core purchase scope |
| F5 | Find books by author | P6 | Later candidate; priority unconfirmed |
| F6 | Request gift wrapping and a note | P7 | Deferred until basic purchasing works |

### F1 - Select and Manage a Cart

- **Approval:** Approved by the user in this conversation on 2026-10-30.
- **User value:** Shoppers collect several books into one cart and control how
  many copies they intend to order.
- **Sources:** S1: FB-01, FB-02, FB-04, FB-10, ON-01.
- **Done when:**
  - A shopper can select different books and review them together in one cart.
  - Each book appears as one line item with its quantity visible, even after
    repeated selection of the same book.
  - A shopper can increase or decrease quantities and remove a book before
    checkout; zero or negative quantities cannot remain as line items.
  - Repeated selection makes the resulting quantity apparent before ordering;
    whether it increments quantity or leaves it unchanged remains a design
    decision.
  - Cart changes do not themselves place an order or charge the shopper.
- **Not in this feature:** Checkout, payment, accounts, or a promise of cart
  recovery after a browser close or server restart.

### F2 - Show Availability and Prevent Overselling

- **User value:** Shoppers can recognize unavailable or scarce books, and the
  owner does not accept orders for copies that are not available.
- **Sources:** S1: FB-03, FB-06, ON-01, ON-03; S3: Stock.
- **Done when:**
  - A shopper can identify out-of-stock books in the catalogue and book view.
  - Books with one to three copies remaining are visibly identified as low
    stock, using the glossary's threshold.
  - Checkout cannot succeed for quantities exceeding available stock, including
    when another shopper orders the same book first.
  - A stock-related rejection identifies the affected book and explains that
    the requested quantity is unavailable; it does not claim order success.
  - Successful orders reduce stock by the ordered quantities, and stock never
    becomes negative.
- **Not in this feature:** Restocking tools, stock notifications, or a promise
  that adding a book to a cart reserves it.

### F3 - Review the Complete Order Total

- **User value:** Shoppers know the full amount due before committing.
- **Sources:** S1: FB-05; S3: Book price is expressed in dollars.
- **Done when:**
  - Before checkout, a shopper sees each selected book, its quantity, unit price,
    line total, and the complete amount due on collection in dollars.
  - Changing quantities or removing a book updates the displayed totals before
    the shopper commits.
  - Every applicable charge is included in the amount shown before commitment;
    no undisclosed charge first appears after ordering.
- **Not in this feature:** Discounts, currency conversion, or inventing tax,
  delivery, or collection fees. Applicable charges require owner clarification.

### F4 - Place and Confirm a Pay-on-Collection Order

- **User value:** Shoppers submit their chosen books in one order and know
  whether it succeeded and what they owe.
- **Sources:** S1: FB-01, FB-02, FB-07, ON-01, ON-02, ON-04.
- **Done when:**
  - A shopper can explicitly confirm a non-empty cart as one order without
    registering, emailing the owner, or providing card details.
  - Checkout uses the reviewed quantities and total and satisfies F2's stock
    protection requirements; an empty cart cannot become an order.
  - A successful checkout creates an order record and shows an order reference,
    ordered books, quantities, complete total, and payment-on-collection status.
  - An unsuccessful checkout clearly reports failure and does not display a
    successful confirmation or create a partial order.
- **Dependencies:** F1 supplies the cart; F2 protects stock; F3 supplies the
  complete pre-commitment total. Each feature can be reviewed separately, but
  the purchase flow is not deliverable to shoppers without all four.
- **Not in this feature:** Online payment, email or SMS confirmations, customer
  accounts, cancellation, refunds, delivery, or an owner administration screen.
  These are proposed boundaries, pending approval.

### F5 - Find Books by Author

- **User value:** Shoppers can find a book without remembering its title.
- **Sources:** S1: FB-08.
- **Done when:** A shopper can search using an author's name, see matching
  catalogue books, and distinguish no matches from a failed search.
- **Not in this feature:** Recommendations, external catalogue search, or an
  agreed commitment to include this in the first purchase release.

### F6 - Request Gift Wrapping and a Note

- **User value:** Gift buyers can communicate wrapping and message requests.
- **Sources:** S1: FB-09, ON-05.
- **Done when:** After this feature is prioritized and its service policy is
  agreed, a shopper can request gift wrapping and a note, see any associated
  charge before commitment, and see the request reflected in the order
  confirmation.
- **Not in this feature:** First-release delivery or assumptions about wrapping
  prices, availability, note limits, or fulfilment policy.

## Out of Scope

- Accounts and online payments are excluded by owner direction.
- Database persistence and infrastructure changes are excluded by the project.
- Gift services are deferred by owner direction, not rejected permanently.
- Author search is retained as a candidate; its release priority is unconfirmed.
- Additional capabilities listed under feature boundaries are proposed
  exclusions for this draft, not claims that the owner explicitly rejected them.

## Open Questions

These decisions remain unresolved; none is silently treated as approved.

1. F1 is approved. Should F2-F4 also be included in the initial purchase scope,
  with F5 later and F6 deferred? Those decisions remain unapproved.
2. Is the amount due simply the sum of book prices, or are there taxes or other
   collection charges that must be included? The sources do not establish this.
3. What shopper contact or collection information, if any, is needed to fulfil
   a pay-on-collection order without an account?
4. Is immediate on-screen confirmation sufficient initially, given that order
   history is not durable across server restarts?
5. For later design: how should repeated book selection behave, how should
   changed stock be presented, and should any stock reservation policy exist?
6. Before scheduling F6, what are the wrapping price, service availability, and
   gift-note limits?

## Approval Record

- **Approved feature:** F1 - Select and Manage a Cart, as written in this document.
- **Approver:** User in this conversation; no personal name supplied.
- **Approval date:** 2026-10-30.
- **Approval evidence:** User message: "Approve F1."
- **Requirement sources:** S1: FB-01, FB-02, FB-04, FB-10, ON-01; project
  constraints and terminology in S2-S4 continue to apply.
- **Scope:** F1 only. F2-F6 and the broader release scope remain unapproved.
- **Remaining F1 design decision:** Repeated selection must make the resulting
  quantity apparent; incrementing versus leaving it unchanged is still open.
- **Handoff:** F1 may proceed to the orchestrator for design through
  shipping-a-feature. This approval does not approve a design or implementation
  plan, authorize a commit, or start delivery automatically.