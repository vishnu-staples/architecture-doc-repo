# Summary
We are migrating shopping cart ownership from Shopify to an in‑house cart embedded within the existing checkout application. Today, our .ca site maintains its own cart data but relies on Shopify for cart state and rules, which limits customization and prevents a seamless express‑checkout experience. Consolidating cart and checkout in a single application removes cross‑app friction, unlocks rapid iteration on cart UX/logic, and reduces vendor lock‑in.

**Primary Objectives**
- Unify cart and checkout to provide a single, streamlined purchase flow.
- Fully replace the Shopify-managed cart on .ca with our in-house cart, enabling express checkout on cart, richer promotions, targeted merchandising, and experimentation.
- Improve reliability, performance, and observability by owning the full funnel.

**Scope (at a glance)**
- Rebuild cart domain logic, APIs, and UI under the checkout application.
- Implement and validate core capabilities natively (pricing, discounts, taxes, shipping estimates, analytics/attribution) and decommission the Shopify cart on .ca.
- Roll out behind feature flags with gradual audience expansion and clear rollback.

# RACI

**RACI** = a simple responsibility model for each major task/deliverable:
- **R — Responsible:** does the work.
- **A — Accountable:** the single owner who signs off
- **C — Consulted:** gives input before decisions (two-way).
- **I — Informed:** kept up to date (one-way).

| Deliverable / Activity                  | PO | Architect | Tech Lead | Dev Team | QA | Security |
|-----------------------------------------|----|-----------|-----------|----------|----|----------|
| System/Domain architecture & data model | I  | A         | I         | I        | I  | C        |
| Cart API ownership & state management   | I  | C         | A         | R        | I  | C        |
| Observability                           | I  | C         | A         | R        | I  | C        |
| Rollback plan                           | A  | C         | R         | R        | I  | C        |
| Testing                                 | A  | C         | A         | C        | R  | C        |

# Current state As-is

<a href="https://lucid.app/publicSegments/view/bb006507-06c0-45c7-8ea6-eab8596648bf/image.png" target="_blank">
  <img src="https://lucid.app/publicSegments/view/bb006507-06c0-45c7-8ea6-eab8596648bf/image.png" alt="Current Cart" width="600" />
</a>

# Future state
<a href="https://lucid.app/publicSegments/view/a38a4f42-7f72-4f1f-aad8-0969da3d9b68/image.png" target="_blank">
  <img src="https://lucid.app/publicSegments/view/a38a4f42-7f72-4f1f-aad8-0969da3d9b68/image.png" alt="Current Cart" width="600" />
</a>

# Feature Requirements

### Must Have
1. Add/Update/Remove items from the Cart
2. Move for Later
3. Wish List
4. GWP (Gift With Purchase)
5. Pricing/Discount/Product Summary detailed along with images
6. Express checkout
7. APIs to external services that includes Segment, Klaviyo and other platforms

### Should Have
1. Save Cart for logged in users across devices
2. When logging in from multiple cart, should be merged with previous cart items
3. Recently viewed

### Could Have
1. Coupons
2. Recommendations

### Won't Have
1. Same domain as `http://staples.ca` - Will be part of checkout domain