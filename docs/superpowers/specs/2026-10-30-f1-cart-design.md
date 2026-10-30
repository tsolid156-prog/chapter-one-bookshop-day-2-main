# F1 - Select and Manage a Cart

Traces to: approved F1 requirements in [docs/intent.md](../../intent.md).

- Status: Approved - gate 1; post-approval commit pending Git setup.
- Draft date: 2026-10-30.
- Requirements: [F1 and Approval Record](../../intent.md).
- Requirements approver: User in this conversation; no personal name supplied.
- Requirements approval: 2026-10-30; recorded evidence: "Approve F1."
- Product decision: User in this conversation, 2026-10-30: "repeated selection
  of a book already in the cart MUST increment quantity by one."
- Decision scope: Resolves the repeated-selection choice left open by F1's
  approved requirements; it does not approve this written spec.
- Design approver: User in this conversation; no personal name supplied.
- Design approval date: 2026-10-30.
- Design approval evidence: User message "Approved." in response to the
  request to approve this written design.
- Gate 1: Approved for this written spec, including the product decision above.
- Delivery mode: Interactive feature; orchestrator coordinates subsequent gates.

## Purpose and Scope

Allow a shopper to collect different books into one cart, review one line item
per book, deliberately change quantities, and remove books. Cart changes never
place an order, charge the shopper, reserve stock, or change catalogue stock.

Only F1 is approved. F2-F6 remain unapproved. This design does not introduce
availability badges, stock-based quantity limits, overselling protection, line
totals, a complete order total, checkout, orders, payment, author search, or gift
services. Existing catalogue prices and browsing remain unchanged.

There are no accounts, authentication, sessions, database, new dependencies,
CSS frameworks, or build system. Cart recovery after browser close or server
restart is not promised. This feature alone is not a purchasing flow.

## Existing Context and Local Design Hypothesis

The current store owns catalogue data and returns copies. API routes use that
store and are mounted in the listener-free `createApp()`. The storefront uses
same-origin fetches, exported pure render helpers, and guarded DOM bootstrap.
API tests use Supertest; render tests check returned HTML without a DOM.

Proposed hypothesis: extending the existing store with carts addressed by a
returned `cartId` is sufficient for F1, without a second source of cart truth.
A discriminating implementation check is to create a cart, select a book, then
read that same cart through the API and observe exactly one line at quantity
one; another cart must remain empty. This check is proposed, not executed.

## Approaches Considered

1. **Server-owned cart with page-memory identity (recommended).** The store owns
   line items; the browser retains the returned `cartId` while the page is open.
   This matches project constraints and avoids recovery or synchronization work.
   Reloading the page starts a new cart context.
2. **Server-owned cart with browser-persisted identity.** This could retain a
   reference across reloads, but adds stale-identity and recovery behavior that
   F1 does not require. Not proposed for this feature.
3. **Browser-only cart.** Simple locally, but does not satisfy the required
   returned `cartId` and server route/store tracer bullet. Rejected.

The recommendation is a draft technical design, not recorded gate 1 approval.

## Proposed Shopper Experience

- Preserve the current catalogue, book detail, Home, and About views. Add an
  explicit Add to cart command to catalogue cards and book detail, plus a Cart
  navigation entry and a dedicated cart section using the existing view pattern.
- The first selection of a book creates one line at quantity one. Each subsequent
  deliberate Add to cart for that book MUST increment its quantity by one in
  the same line, unless doing so would exceed the safe-integer range. Visibly
  and accessibly confirm the resulting quantity after each successful selection.
- The cart section is an unframed list, not a new marketing page or modal.
  Each line shows the book title, author, existing cover asset, visible quantity,
  quantity controls, and a remove command. Do not add price summaries or totals.
- Use a labelled positive-integer quantity input with increment/decrement
  controls. Decrement is disabled at one; removal is a separate explicit action.
  Reject zero rather than silently treating it as removal. The final removed
  line leaves an empty cart with a return-to-catalogue command.
- At `Number.MAX_SAFE_INTEGER`, disable the line's increment control and that
  book's Add to cart controls. Explain the quantity limit accessibly; allow
  decreasing, setting another valid quantity, or removing the line. The server
  enforces the same limit even if a request bypasses disabled controls.
- Give icon controls accessible names and tooltips, with no new icon dependency.
  Announce confirmed cart changes, including the resulting quantity, in an
  accessible status region. Keep focus usable after removal and view changes.
- Preserve the established visual language. All new styling belongs in the
  existing stylesheet; controls and text must fit desktop and narrow mobile
  layouts. No inline styles are added for cart UI.
- Keep the cart identity and last confirmed response in page memory, without
  cookies or browser persistence. Navigating among in-page views retains it.
  Do not create a cart merely to display the empty initial cart view.
- Serialize cart writes and prevent overlapping creation requests. Disable
  relevant mutation controls while a write is pending, then render the server's
  confirmed result rather than guessing success locally.
- There is no checkout button, order confirmation, or payment action.

## Proposed Data and Ownership

The existing store remains the single source of truth. A cart contains an opaque,
server-generated `cartId` and line items keyed uniquely by `bookId`. Each line
stores a positive safe-integer quantity. Generate identities using existing Node
capabilities, not a dependency, and keep carts separate from catalogue data.

The response representation is `{ cartId, items }`, where each item contains
`bookId`, `title`, `author`, `coverImage`, and `quantity`. Resolve book display
fields from the catalogue; do not accept those fields from clients. Do not add
monetary totals, stock reservations, or order fields. Return detached snapshots
so callers cannot mutate the store through responses.

Store operations create/read carts, select a book, set an existing line's
absolute quantity, and remove a line. Each operation completes synchronously
within the current process, including duplicate handling: selection inserts a
missing line at one or increments the existing quantity by exactly one. Check
the safe-integer limit before mutation; never wrap, clamp, or silently ignore
an overflowing selection. `store.reset()` clears
carts as well as restoring copied seed books. Different carts remain isolated.
Restarting discards carts; a previously issued identity then returns not found.

## Proposed API Contract

Routes live under `/api/carts`, mounted before the existing API fallback. All
body validation happens at the route edge before mutation. Errors use JSON
`{ "error": "message" }`; rejected operations leave the cart unchanged.

| Request | Successful result | Rejection |
| --- | --- | --- |
| `POST /api/carts` with `{}` | `201`, empty cart and returned `cartId` | `400` for invalid body |
| `GET /api/carts/:cartId` | `200`, current cart snapshot | `404` for unknown cart |
| `POST /api/carts/:cartId/items` with `{ bookId }` | `201` and snapshot for a new line at quantity one; `200` and snapshot with an existing line incremented by exactly one | `400` for invalid body or an increment exceeding the safe-integer range; `404` for unknown cart or book |
| `PATCH /api/carts/:cartId/items/:bookId` with `{ quantity }` | `200`, snapshot with the existing line set to that quantity | `400` for invalid quantity; `404` for unknown cart or line |
| `DELETE /api/carts/:cartId/items/:bookId` | `204`, no body; client then reads the cart | `404` for unknown cart or line |

Require a non-empty string `bookId` for selection. Quantity must be a JSON
number that is a positive safe integer; reject missing values, numeric strings,
fractions, zero, negatives, and unsafe integers. Do not infer a stock limit.
Accept `Number.MAX_SAFE_INTEGER` as an absolute quantity. Selecting that book
again returns `400` with `{ "error": "Quantity cannot exceed the safe-integer limit" }`
and leaves the entire cart unchanged. Incrementing from one below the maximum
succeeds and returns the maximum. Handle malformed JSON on these routes with
the same JSON error shape rather than an HTML error page.

Selection is deliberately non-idempotent: each successful request for an
existing line increments quantity by one. `PATCH` instead sets an absolute
quantity and does not add to it. Do not automatically retry selection writes.

## Error and Recovery Behavior

- Check every cart fetch response, including bodyless deletion responses.
  Show an accessible error on network or API failure; do not claim a change
  succeeded or discard the last confirmed cart because a request failed.
- Do not automatically replay an ambiguous selection request: if its response
  was lost after a successful write, replay could add an unintended copy.
  Re-read the cart before enabling further mutations and visibly and accessibly
  present the confirmed quantity. Any later deliberate selection increments
  that confirmed quantity by one; it is not a replay of the uncertain request.
- If a refresh also fails, show the cart as unconfirmed and offer a retry read;
  keep mutation controls disabled and do not label stale quantities as current.
- A safe-integer-limit rejection leaves the confirmed quantity unchanged.
  Show the limit error, do not claim a copy was added, and do not retry the write.
- An unknown cart identity clears the invalid local reference and presents an
  empty cart with an explicit unavailable-cart notice. Do not silently restore
  or replay lost selections; a subsequent deliberate selection creates a cart.
- Escape all API-provided values interpolated into HTML with `escapeHtml`.
  Keep render helpers pure and exported, and DOM startup guarded for tests.

## Done When

1. A shopper selects two different catalogue books and sees them together in the
   same cart, each at quantity one, while existing browsing still works.
2. Selecting the same book repeatedly never creates a second line for that book.
  Starting at one, two further successful selections produce quantities two
  and three in that same line, confirmed both by API reads and visibly and
  accessibly in the storefront.
3. A shopper increases and decreases a line's quantity deliberately; the cart
   read back from the API reflects the confirmed positive integer quantity.
4. Invalid quantities are rejected without changing the cart; no zero or negative
   line item can remain. Decrement at one does not remove a line implicitly.
  Setting `Number.MAX_SAFE_INTEGER` is valid; selection from one below it
  reaches that maximum. A further selection returns JSON `400` without changing
  any line, and the UI explains the limit without reporting a successful add.
5. Removing a book removes only that line; removing the last line displays an
   empty cart. A fresh API read agrees with the rendered state.
6. Two returned cart identities have independent contents. Reset discards cart
   state, and unknown identities receive JSON `404` responses.
7. Cart mutations leave catalogue stock unchanged and do not create an order,
   charge, reservation, total, or checkout action.
8. Pending, failed, and uncertain mutations are distinguishable from confirmed
   changes. A lost response does not trigger an automatic duplicate selection.
  A successful recovery read shows the actual quantity before mutations resume;
  a failed recovery read keeps quantities unconfirmed and mutations disabled.
9. Keyboard users can select, review, change quantity, remove, and return to the
   catalogue; controls remain usable without overlap on desktop and mobile.
10. Rendered book fields are escaped, and the required checks below have fresh
    evidence before implementation is called complete.

## Verification and Later Delivery Constraints

Use Vitest and Supertest against `createApp()` for externally visible cart
creation, selection, reads, increment-by-one repeated selection, safe-integer
boundaries and unchanged state on overflow rejection, quantity validation,
removal, isolation, unknown resources, JSON errors, and unchanged catalogue
stock. Reset the shared store before mutation tests. Extend existing pure-render
tests for empty/populated cart output, quantity controls, and escaping.
Browser verification should cover navigation, keyboard/focus behavior, responsive
layout, safe-integer-limit controls, and delayed or failed requests, including a
lost selection response followed by successful or failed recovery reads; do not
add a testing dependency.

Later phase 1 must be a minimal vertical tracer bullet through route, store,
and tests: create a cart, select one book, and read it back. This is a delivery
constraint, not an implementation plan; this designer handoff does not authorize
planning or implementation.

Each implementation phase requires fresh `npm test`, `npm run typecheck`, and
`npm run lint` evidence. Final review also requires `npm run format:check` and
`npm audit --audit-level=high`. These commands are verification requirements,
not results claimed by this draft.

Resolved later artifact paths are `docs/superpowers/plans/2026-10-30-f1-cart.md`
for the plan and `docs/superpowers/reviews/f1-cart-review.md` for final review.
They are handoff context only; neither artifact is created or updated here.

## Unresolved Product Decisions

None remain for F1 based on the supplied requirements and decision. The user
resolved repeated selection on 2026-10-30: it MUST increment quantity by one.
The earlier leave-quantity-unchanged recommendation is superseded, not an
alternative available to implementers. This decision resolves a choice expressly
delegated to design in the approved intent; the intent file is not changed here.
F2-F6 decisions remain outside this spec and unapproved.

## Handoff and Gate Status

The user approved this written spec on 2026-10-30. No F1 product questions
remain. The required post-approval commit is pending Git setup; the plan and
implementation have not been approved.

Workspace Git context is supplied as not a Git repository. Base branch and remote
remain unresolved; CI mentioning `main` does not establish a base. Do not
initialize Git or infer a branch. This blocks the post-approval commit workflow
until resolved.