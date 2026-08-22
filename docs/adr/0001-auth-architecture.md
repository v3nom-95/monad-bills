# ADR-0001: End-user auth via Supabase in our own app; Lago stays stock

- Status: Accepted
- Date: 2026-08-22

## Context

We need end-user authentication for monad-bill, with Lago as the billing engine.

The following facts were verified against the Lago v1.51.0 source in `lago/`, not
from recollection:

- **Lago has two entirely separate auth paths.** The admin UI / GraphQL path uses
  a JWT (`api/app/services/utils/auth_token.rb`): HS256, signed with
  `SECRET_KEY_BASE`, 3-hour lifetime, and **no refresh token** — instead the
  server renews the token in-flight and returns it in the `x-lago-token` response
  header when it is within an hour of expiry
  (`api/app/controllers/concerns/authenticable_user.rb`). The acting organization
  is resolved from a client-supplied `x-lago-organization` header, validated
  against the user's active memberships.
- **REST `/api/v1/*` is API-key auth only and has no `current_user` at all.**
  `api/app/controllers/api/base_controller.rb` authenticates solely by looking up
  the bearer token as an API key and setting `@current_organization`. A grep for
  `current_user` across `api/app/controllers/api/` returns **zero** matches. This
  is the single fact that makes the decision easy: there is no end-user identity
  in Lago's REST API to integrate with, so there is nothing to wire ours into.
- **A Lago `User` is meaningless without a `Membership`.** `UsersService#login`
  rejects a user with no active membership outright, and all permissions hang off
  Membership → Role.
- **Lago ships exactly three authentication methods:** `email_password`,
  `google_oauth`, `okta`. Okta is real OIDC and would have been the template to
  copy — but it is a premium integration (`Organization::PREMIUM_INTEGRATIONS`,
  gated on `License.premium?`), and there is **no generic OIDC or SAML provider**.
  So "just point Lago at Supabase" was never an available option.

## Decision

Lago stays **stock**, running the prebuilt images `getlago/api:v1.51.0` and
`getlago/front:v1.51.0`.

We build monad-bill as our own Next.js app using **Supabase for end-user auth**,
and talk to Lago **server-side only**, over its REST API, with an
organization-scoped API key. Lago's own login remains an internal admin console
for us.

### Two load-bearing design rules

1. **`customer.external_id` = the Supabase user uid.** That single field *is* the
   mapping between the two systems. `POST /api/v1/customers` is an upsert keyed on
   `external_id`, so provisioning is naturally idempotent.
2. **Provision the Lago customer lazily, on the first authenticated request** —
   never at signup, and never from a Supabase database webhook. Supabase is
   configured with `mailer_autoconfirm: false`, so a user row exists *before* the
   email is confirmed; provisioning at signup would create Lago customers for
   addresses that never confirm.

## Consequences

Positive:

- Lago upgrades are `docker compose pull`. Nothing to re-merge.
- `/api/v1` is a versioned public contract Lago maintains for third parties, so
  it is the surface least likely to break under us.
- The Lago API key never reaches a browser.
- Supabase JWT verification is done by Supabase's own SDK, rather than by code we
  forked into a Rails monolith.

Negative / accepted costs:

- **A pure SPA is not an option.** We must run a server/BFF tier. The Lago API key
  is organization-wide with read+write scope; leaking it exposes every customer,
  invoice and wallet in the organization.
- We own the entire end-user billing UI ourselves (see the rejected hosted-portal
  option below).
- The design assumes **end users never see Lago's admin UI**. If that assumption
  ever breaks, this ADR must be revisited — the first rejected option below stops
  being optional.

## Alternatives rejected

### Fork Lago to accept Supabase JWTs

The sane shape would have been a fourth `authentication_methods` entry cloned
from the `google_oauth` / `okta` path: verify the Supabase JWT once at a new
GraphQL mutation, then let Lago mint its own normal token — leaving token
renewal, `x-lago-organization`, and the customer-portal path untouched. Roughly
14–16 files (≈7 in `api`, including regenerated `schema.graphql` and
`schema.json`; ≈8 in `front`, including a regenerated
`src/generated/graphql.tsx`).

**Why rejected:** the file count was never the cost. We would abandon the
prebuilt images for build-from-source, and carry a permanent fork to be re-merged
on every Lago release — in files Lago is actively churning (the
`authentication_methods` concern, `user_devices`, the security-log calls in
`UsersService`), with a guaranteed textual conflict in `schema.graphql` on every
single upgrade. And it buys nothing, because our end users never log into Lago's
admin UI.

**This is explicitly not a one-way door.** The Okta/Google template stays
available at the same cost, should the "end users never see the admin UI"
assumption break.

### Lago's hosted customer portal

`GET /api/v1/customers/:external_id/portal_url` exists and would have removed the
need to build billing UI at all.

**Rejected by the user in favour of native rendering inside monad-bill** — full
control of look and UX, at the cost of building and maintaining every billing
view ourselves. Recorded here as a deliberate, informed trade, not an oversight.

### One Lago organization per tenant

Rejected. We use a single organization; end users are Lago **customers**, not
Lago users. One API key, no organization provisioning. This is what the API-key
model assumes.

### Deliberately not built (YAGNI)

- A generalized pluggable-auth layer for Lago.
- Migration of existing Lago users into Supabase.
- Any Supabase ↔ Lago sync daemon.
- Any `profiles` mapping table — rule 1 above makes it unnecessary.
