# UCP 101 — A Mechanical Guide for Agents

A condensed, do-this-then-do-that guide for an agent that needs to talk to a UCP-enabled Shopify storefront using the **MCP transport**. Worked examples use `https://retro-rewind-shop.myshopify.com/` throughout.

The full spec lives at https://github.com/Universal-Commerce-Protocol/ucp and https://ucp.dev. This document is intentionally lossy: it covers only what's needed to discover a business, search its catalog, link identity via OAuth, and complete a checkout.

---

## 0. Pre-flight: what you need before any task

UCP requires that every request you make declares **who you are** by URL. You must therefore host a **platform profile** JSON document at a stable HTTPS URL before doing anything else.

### Your platform profile (one-time setup)

Host this somewhere reachable (S3, GitHub Pages, your own server). Call its URL `$AGENT_PROFILE_URL`.

```json
{
  "ucp": {
    "version": "2026-04-08",
    "services": {
      "dev.ucp.shopping": [
        {
          "version": "2026-04-08",
          "spec": "https://ucp.dev/2026-04-08/specification/overview",
          "transport": "mcp",
          "schema": "https://ucp.dev/2026-04-08/services/shopping/mcp.openrpc.json",
          "endpoint": "https://your-agent.example.com/ucp/mcp"
        }
      ]
    },
    "capabilities": {
      "dev.ucp.shopping.catalog.search":  [{"version": "2026-04-08", "spec": "https://ucp.dev/2026-04-08/specification/catalog", "schema": "https://ucp.dev/2026-04-08/schemas/shopping/catalog_search.json"}],
      "dev.ucp.shopping.catalog.lookup":  [{"version": "2026-04-08", "spec": "https://ucp.dev/2026-04-08/specification/catalog", "schema": "https://ucp.dev/2026-04-08/schemas/shopping/catalog_lookup.json"}],
      "dev.ucp.shopping.checkout":        [{"version": "2026-04-08", "spec": "https://ucp.dev/2026-04-08/specification/checkout", "schema": "https://ucp.dev/2026-04-08/schemas/shopping/checkout.json"}],
      "dev.ucp.shopping.fulfillment":     [{"version": "2026-04-08", "spec": "https://ucp.dev/2026-04-08/specification/fulfillment", "schema": "https://ucp.dev/2026-04-08/schemas/shopping/fulfillment.json", "extends": "dev.ucp.shopping.checkout"}],
      "dev.ucp.common.identity_linking":  [{"version": "2026-04-08", "spec": "https://ucp.dev/specification/identity-linking", "schema": "https://ucp.dev/schemas/common/identity_linking.json"}]
    },
    "payment_handlers": {}
  }
}
```

The URL serving this JSON **MUST**:

1. Be HTTPS.
2. Return `Cache-Control: public, max-age=86400` (or any `max-age` ≥ 60, **never** `private`, `no-store`, or `no-cache`). Businesses will reject the request otherwise.
3. Not redirect (no 3xx).
4. Be a valid JSON object matching the schema above.

### Environment variables used in the examples

```bash
export BUSINESS_HOST="https://retro-rewind-shop.myshopify.com"
export AGENT_PROFILE_URL="https://your-agent.example.com/.well-known/ucp"   # the JSON above
```

### The single canonical request envelope

Every MCP tool call has this shape. Pay attention to `meta.ucp-agent.profile` — it is mandatory on **every** call.

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "<tool_name>",
    "arguments": {
      "meta": {
        "ucp-agent": { "profile": "$AGENT_PROFILE_URL" }
      },
      "<domain_payload_key>": { ... }
    }
  }
}
```

`<domain_payload_key>` is `catalog` for catalog tools and `checkout` for checkout tools. For `complete_checkout` / `cancel_checkout`, `meta` must also contain an `idempotency-key` (UUIDv4).

### Reading any UCP response

* Transport failure → JSON-RPC `error` field. Most common: `-32001 UCP discovery failed` (your profile URL is wrong or uncacheable). Fix and retry.
* Business outcome → JSON-RPC `result.structuredContent`. Inside that:
  * `ucp.version` / `ucp.capabilities` — the negotiated set for this response.
  * `messages[]` — zero or more diagnostics. Each has `type` (`error`/`warning`/`info`), `code`, `content`, and (for errors) `severity`.
  * The domain payload (e.g. `products`, `id`/`status`/`line_items` for checkouts).

---

## Task A — Discover the business via `/.well-known/ucp`

### Preconditions

* You know the business's base domain (`$BUSINESS_HOST`).

### Procedure

1. `GET $BUSINESS_HOST/.well-known/ucp`. Follow one redirect if returned (Shopify shops redirect `retro-rewind-shop.myshopify.com` to a canonical hash subdomain).
2. Parse JSON. Cache by `Cache-Control` directives.
3. Locate the MCP endpoint: `ucp.services["dev.ucp.shopping"][?transport=="mcp"].endpoint`. Call this `$MCP_ENDPOINT`.
4. Confirm the capabilities you need are present in `ucp.capabilities` (e.g. `dev.ucp.shopping.catalog.search`, `dev.ucp.shopping.checkout`, `dev.ucp.common.identity_linking`).
5. Note `ucp.payment_handlers` — you'll need them at checkout completion time.
6. If `ucp.capabilities["dev.ucp.common.identity_linking"][0].config.scopes` is **absent or empty**, identity linking is purely optional (e.g. retro-rewind-shop). If populated, the listed scopes gate the operations they name behind user auth (see Task D).

### cURL

```bash
curl -sLi "$BUSINESS_HOST/.well-known/ucp"
```

### Postconditions

* `$MCP_ENDPOINT` is set. For `retro-rewind-shop.myshopify.com` it resolves to:
  ```
  https://78eae1-95.myshopify.com/api/ucp/mcp
  ```
* You have a map of capabilities and payment handlers.

### What you'll see for retro-rewind-shop (abridged)

```json
{
  "ucp": {
    "version": "2026-04-08",
    "services": {
      "dev.ucp.shopping": [
        { "transport": "mcp",      "endpoint": "https://78eae1-95.myshopify.com/api/ucp/mcp" },
        { "transport": "embedded", "schema": "https://ucp.dev/2026-04-08/services/shopping/embedded.openrpc.json" }
      ]
    },
    "capabilities": {
      "dev.ucp.shopping.checkout":         [{ "version": "2026-04-08" }],
      "dev.ucp.shopping.fulfillment":      [{ "extends": ["dev.ucp.shopping.checkout","dev.ucp.shopping.cart"] }],
      "dev.ucp.shopping.discount":         [{ "extends": ["dev.ucp.shopping.checkout","dev.ucp.shopping.cart"] }],
      "dev.ucp.shopping.cart":             [{ "version": "2026-04-08" }],
      "dev.ucp.shopping.order":            [{ "version": "2026-04-08" }],
      "dev.ucp.shopping.catalog.search":   [{ "version": "2026-04-08" }],
      "dev.ucp.shopping.catalog.lookup":   [{ "version": "2026-04-08" }],
      "dev.ucp.common.identity_linking":   [{ "version": "2026-04-08" }]
    },
    "payment_handlers": {
      "com.google.pay":      [ { "id": "gpay", ... } ],
      "dev.shopify.card":    [ { "id": "shopify.card", ... } ],
      "dev.shopify.shop_pay":[ { "id": "shop_pay",   ... } ]
    }
  }
}
```

---

## Task B — How to search a catalog

### Preconditions

* Task A completed. `$MCP_ENDPOINT` is known.
* `dev.ucp.shopping.catalog.search` is present in `ucp.capabilities`.
* `$AGENT_PROFILE_URL` is live and properly cached.

### Procedure

1. Issue an MCP `tools/call` to `search_catalog` with the search query, optional filters, and pagination.
2. A valid request **MUST** include at least one of: `query`, `filters`, or an extension-defined input.
3. Read `result.structuredContent.products[]` from the response.
4. Pagination: pass the returned `pagination.cursor` back in the next request to page through.

### Required request fields

| Field | Type | Notes |
| :--- | :--- | :--- |
| `params.name` | string | `"search_catalog"` |
| `params.arguments.meta.ucp-agent.profile` | URL | `$AGENT_PROFILE_URL` |
| `params.arguments.catalog.query` | string | Free-text. Omit for browse. |
| `params.arguments.catalog.filters` | object | Optional. `categories`, `price.min`, `price.max` (in minor currency units). |
| `params.arguments.catalog.context.address_country` | ISO 3166-1 alpha-2 | Recommended to get region-correct prices. |
| `params.arguments.catalog.pagination.limit` | int | Requested page size. May be clamped silently. Default 10. |
| `params.arguments.catalog.pagination.cursor` | string | Opaque cursor from prior response. |

### cURL

```bash
curl -sL -X POST "$MCP_ENDPOINT" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
      "name": "search_catalog",
      "arguments": {
        "meta": { "ucp-agent": { "profile": "'"$AGENT_PROFILE_URL"'" } },
        "catalog": {
          "query": "retro t-shirt",
          "context": { "address_country": "US" },
          "filters": { "price": { "max": 5000 } },
          "pagination": { "limit": 10 }
        }
      }
    }
  }'
```

### Postconditions

* `result.structuredContent.products[]` is an array of products. Each product has:
  * `id` — opaque product identifier.
  * `title`, `description.plain`, `url`, `media[]`.
  * `price_range.min.amount` / `price_range.max.amount` — in minor units (cents).
  * `variants[]` — each variant has its own `id`, `sku`, `price`, `availability.available`, and `options[]`.
* `result.structuredContent.pagination.has_next_page` / `pagination.cursor` for paging.
* If you need full product detail with selectable options, follow up with `get_product`:

  ```bash
  curl -sL -X POST "$MCP_ENDPOINT" -H "Content-Type: application/json" -H "Accept: application/json, text/event-stream" -d '{
    "jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"get_product","arguments":{
      "meta":{"ucp-agent":{"profile":"'"$AGENT_PROFILE_URL"'"}},
      "catalog":{"id":"<PRODUCT_ID>","selected":[{"name":"Size","label":"M"}],"context":{"address_country":"US"}}
    }}
  }'
  ```
* For bulk identifier resolution, use `lookup_catalog` with `catalog.ids: ["prod_a","prod_b"]`.

### Failure modes

* `result.structuredContent.ucp.status == "error"` with `messages[].code == "not_found"` — product/identifier missing. Not a transport error.
* JSON-RPC `error.code == -32602` — input validation failure (e.g., empty request body).
* JSON-RPC `error.code == -32001` — discovery error. Re-check your platform profile URL and its cache headers.

---

## Task C — How to complete a checkout

The checkout has a small state machine. The platform drives it forward by calling tools; the business is the authoritative state holder.

```
incomplete  ──update──▶  ready_for_complete  ──complete──▶  complete_in_progress  ──▶  completed
    │                                                                                       ▲
    │                                                                                       │
    ▼                                                                                       │
requires_escalation  ──(buyer finishes via continue_url)─────────────────────────────────────
```

Plus `canceled` (terminal, may be reached from anywhere).

### Preconditions

* Task A completed. `$MCP_ENDPOINT` and `ucp.payment_handlers` are known.
* `dev.ucp.shopping.checkout` is in the business's capabilities.
* You have one or more variant `id`s and `quantity`s to purchase (from Task B or pre-known).
* You have buyer email and (for shipping) a destination address.
* You have a way to generate **UUIDv4 idempotency keys**.
* If the business's `dev.ucp.common.identity_linking.config.scopes` includes `dev.ucp.shopping.checkout:manage`, you need a Bearer access token from Task D. **retro-rewind-shop does NOT gate checkout**, so a token is not required for that store.

### Step C1 — Create the checkout session

Identifier rule: on `create_checkout`, omit `id` on the top-level `checkout` *and* on each `line_items[]` entry. The business mints CartLine GIDs and returns them on the response — reuse those GIDs on subsequent `update_checkout` / `complete_checkout` calls when you need to reference a line.

```bash
curl -sL -X POST "$MCP_ENDPOINT" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
      "name": "create_checkout",
      "arguments": {
        "meta": { "ucp-agent": { "profile": "'"$AGENT_PROFILE_URL"'" } },
        "checkout": {
          "buyer": {
            "email": "jane.doe@example.com",
            "first_name": "Jane",
            "last_name": "Doe"
          },
          "line_items": [
            { "item": { "id": "<VARIANT_ID_FROM_SEARCH>" }, "quantity": 1 }
          ],
          "currency": "USD",
          "fulfillment": {
            "methods": [
              {
                "type": "shipping",
                "destinations": [
                  {
                    "street_address": "123 Main St",
                    "address_locality": "Springfield",
                    "address_region": "IL",
                    "postal_code": "62701",
                    "address_country": "US"
                  }
                ]
              }
            ]
          }
        }
      }
    }
  }'
```

Capture from the response:

* `result.structuredContent.id` → `$CHECKOUT_ID`
* `result.structuredContent.status`
* `result.structuredContent.continue_url` (use for buyer handoff if escalation occurs)
* `result.structuredContent.expires_at` (don't act after this timestamp)
* `result.structuredContent.ucp.payment_handlers` — authoritative list of handlers/instruments available to **this** checkout. Use this, not the business profile's list.

### Step C2 — Resolve messages and update if needed

Inspect `messages[]` and partition by `severity`:

| Severity | Action |
| :--- | :--- |
| `unrecoverable` | Stop. Inform user. Maybe restart with different inputs. |
| `recoverable` | Fix the offending input (path is given in `message.path`) and call `update_checkout`. Repeat until clean. |
| `requires_buyer_input` | Hand off to `continue_url` — you cannot complete from the API. |
| `requires_buyer_review` | Hand off to `continue_url` — buyer must approve. |

`update_checkout` is a **full replacement** of the resource. Send the entire desired state of `checkout`, omit only the top-level `id`. Read `line_items[i].id` from the create response — those are server-minted CartLine GIDs — and use them verbatim wherever you reference the line (e.g., `fulfillment.methods[].line_item_ids`):

```bash
curl -sL -X POST "$MCP_ENDPOINT" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{
    "jsonrpc": "2.0",
    "id": 2,
    "method": "tools/call",
    "params": {
      "name": "update_checkout",
      "arguments": {
        "meta": { "ucp-agent": { "profile": "'"$AGENT_PROFILE_URL"'" } },
        "id": "'"$CHECKOUT_ID"'",
        "checkout": {
          "buyer": { "email": "jane.doe@example.com", "first_name": "Jane", "last_name": "Doe" },
          "line_items": [
            { "id": "<CART_LINE_ID_FROM_CREATE>", "item": { "id": "<VARIANT_ID>" }, "quantity": 2 }
          ],
          "currency": "USD",
          "fulfillment": {
            "methods": [
              { "id": "shipping_1", "line_item_ids": ["<CART_LINE_ID_FROM_CREATE>"],
                "groups": [{ "id": "package_1", "selected_option_id": "express" }] }
            ]
          }
        }
      }
    }
  }'
```

Loop until `status == "ready_for_complete"`.

### Step C3 — Collect payment instrument

Look at the `ucp.payment_handlers` you got back. Each handler is one of:

* `com.google.pay` — collect via Google Pay JS SDK using `merchant_info` + `allowed_payment_methods` from `config`.
* `dev.shopify.card` — direct card entry; you'd use Shopify's tokenization endpoint.
* `dev.shopify.shop_pay` — Shop Pay redirect/handoff.

Each handler defines how to produce a `payment.instruments[i].credential` blob. This protocol-101 doc does not reproduce each handler's spec — follow the handler's `spec` URL.

The output of collection is a `payment.instruments[]` array of the shape:

```json
{
  "instruments": [
    {
      "id": "pi_xyz_1",
      "handler_id": "<id FROM payment_handlers, e.g. \"gpay\" or \"shopify.card\">",
      "type": "card",
      "selected": true,
      "display": { "brand": "visa", "last_digits": "4242", "description": "Visa •••• 4242" },
      "billing_address": {
        "street_address": "123 Main St",
        "address_locality": "Springfield",
        "address_region": "IL",
        "postal_code": "62701",
        "address_country": "US"
      },
      "credential": { "type": "PAYMENT_GATEWAY", "token": "<handler-issued-token>" }
    }
  ]
}
```

### Step C4 — Complete the checkout

This is the moment the order is placed. **Idempotency key is required**.

```bash
IDEMPOTENCY_KEY=$(uuidgen)

curl -sL -X POST "$MCP_ENDPOINT" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{
    "jsonrpc": "2.0",
    "id": 3,
    "method": "tools/call",
    "params": {
      "name": "complete_checkout",
      "arguments": {
        "meta": {
          "ucp-agent": { "profile": "'"$AGENT_PROFILE_URL"'" },
          "idempotency-key": "'"$IDEMPOTENCY_KEY"'"
        },
        "id": "'"$CHECKOUT_ID"'",
        "checkout": {
          "payment": {
            "instruments": [
              {
                "id": "pi_xyz_1",
                "handler_id": "shopify.card",
                "type": "card",
                "selected": true,
                "display": { "brand": "visa", "last_digits": "4242" },
                "billing_address": {
                  "street_address": "123 Main St",
                  "address_locality": "Springfield",
                  "address_region": "IL",
                  "postal_code": "62701",
                  "address_country": "US"
                },
                "credential": { "type": "PAYMENT_GATEWAY", "token": "<handler-token>" }
              }
            ]
          },
          "signals": {
            "dev.ucp.buyer_ip": "203.0.113.42",
            "dev.ucp.user_agent": "Mozilla/5.0"
          }
        }
      }
    }
  }'
```

If `complete_checkout` times out or returns a network error, **retry with the same idempotency key**. The business will deduplicate and return the same final outcome.

### Step C5 — Cancel (if needed, before completion)

```bash
curl -sL -X POST "$MCP_ENDPOINT" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{
    "jsonrpc":"2.0","id":9,"method":"tools/call",
    "params": {
      "name":"cancel_checkout",
      "arguments":{
        "meta":{"ucp-agent":{"profile":"'"$AGENT_PROFILE_URL"'"},"idempotency-key":"'"$(uuidgen)"'"},
        "id":"'"$CHECKOUT_ID"'"
      }
    }
  }'
```

### Postconditions for a successful complete

* `result.structuredContent.status == "completed"`.
* `result.structuredContent.order.id` — the placed order's identifier. Persist this.
* `result.structuredContent.order.permalink_url` — a URL the buyer can visit for a hosted order receipt.
* The business sends a confirmation email to the buyer.

### Failure modes worth knowing

| Symptom | Meaning | What to do |
| :--- | :--- | :--- |
| `messages[].code == "out_of_stock"` (`recoverable`) | Item went out of stock between search and checkout. | Update with different variant/quantity. |
| `messages[].code == "address_undeliverable"` | Cannot ship there. | Update with a different `destinations[]`. |
| `messages[].code == "payment_failed"` | Payment did not authorize. | Collect a new instrument and call `complete_checkout` again with a **new** idempotency key (this is a new attempt). |
| `messages[].code == "eligibility_invalid"` | A claimed loyalty / member benefit could not be verified. | Either remove the claim from `context.eligibility` (and re-update), or switch to the qualifying instrument. |
| HTTP 401 with `WWW-Authenticate: Bearer` and `code == "identity_required"` | This business requires user auth for checkout. Go do Task D first, then retry with `Authorization: Bearer <token>`. |
| HTTP 403 with `code == "insufficient_scope"` | Token is missing a scope. **Do not** restart linking; use the OAuth incremental-authorization flow to request only the missing scope listed in the `scope` parameter of the `WWW-Authenticate` header. |

---

## Task D — How to perform identity linking (OAuth)

This is only required if the business's `/.well-known/ucp` lists scopes under `dev.ucp.common.identity_linking.config.scopes`, **or** if you receive an `identity_required` / `insufficient_scope` error mid-flow. For retro-rewind-shop it is optional but unlocks personalization.

UCP v1 identity linking is OAuth 2.0 **Authorization Code with PKCE (S256)**, per RFC 6749 / 7636 / 9207.

### Preconditions

* `dev.ucp.common.identity_linking` capability advertised in business profile.
* You have a registered OAuth client (`$CLIENT_ID`, optional `$CLIENT_SECRET` for confidential clients) with the business.
* You have a registered `$REDIRECT_URI` that exactly matches what the business expects (byte-for-byte; loopback `127.0.0.1`/`[::1]` is the only exception, where the port is ignored).
* You have a user agent (browser) you can launch.

### Step D1 — Discover the OAuth authorization server

Two-tier strict hierarchy. **Do not silently fall through on any error other than 404 in step 1.**

1. `GET $BUSINESS_HOST/.well-known/oauth-authorization-server`
   * `2xx` → use it.
   * `404` → try step 2.
   * Any other error → abort.
2. `GET $BUSINESS_HOST/.well-known/openid-configuration`
   * `2xx` → use it.
   * Anything else → abort.

```bash
curl -sL "$BUSINESS_HOST/.well-known/oauth-authorization-server"
```

For retro-rewind-shop you get back (abridged):

```json
{
  "issuer": "https://shopify.com/authentication/84411744278",
  "authorization_endpoint": "https://shopify.com/authentication/84411744278/oauth/authorize",
  "token_endpoint":         "https://shopify.com/authentication/84411744278/oauth/token",
  "jwks_uri":               "https://shopify.com/authentication/84411744278/.well-known/jwks.json",
  "scopes_supported":       ["openid","email","customer-account-api:full","customer-account-mcp-api:full","ucp:scopes:checkout_session"],
  "response_types_supported":            ["code"],
  "code_challenge_methods_supported":    ["S256"],
  "token_endpoint_auth_methods_supported":["client_secret_basic"],
  "grant_types_supported":               ["authorization_code","refresh_token","urn:ietf:params:oauth:grant-type:jwt-bearer"]
}
```

Set:

```bash
export AUTH_ENDPOINT="https://shopify.com/authentication/84411744278/oauth/authorize"
export TOKEN_ENDPOINT="https://shopify.com/authentication/84411744278/oauth/token"
export ISSUER="https://shopify.com/authentication/84411744278"
```

### Step D2 — Derive the scope set you want

1. Read `dev.ucp.common.identity_linking.config.scopes` from the business profile.
2. Keep only scopes whose capability prefix you implement.
3. Pick what your user is consenting to (e.g. `dev.ucp.shopping.order:read dev.ucp.shopping.order:manage`).
4. **Verify each selected scope appears in `scopes_supported`**. If not, abort; the merchant will reject anyway.
5. Request **only** the derived set — never a superset.

### Step D3 — Generate PKCE + state

```bash
# code_verifier: 43..128 chars URL-safe
CODE_VERIFIER=$(openssl rand -base64 96 | tr -d '+/=\n' | head -c 96)
CODE_CHALLENGE=$(printf '%s' "$CODE_VERIFIER" | openssl dgst -sha256 -binary | openssl base64 | tr -d '=\n' | tr '/+' '_-')
STATE=$(uuidgen)
```

### Step D4 — Send the user to the authorization endpoint

Open this URL in a browser (or print it for the user). Reserved characters in `scope` and `redirect_uri` must be percent-encoded.

```
$AUTH_ENDPOINT
  ?response_type=code
  &client_id=$CLIENT_ID
  &redirect_uri=$REDIRECT_URI
  &scope=<derived space-separated scopes>
  &code_challenge=$CODE_CHALLENGE
  &code_challenge_method=S256
  &state=$STATE
```

### Step D5 — Validate the redirect

The business redirects to `$REDIRECT_URI?code=...&state=...&iss=...`.

**Mandatory checks** before continuing:

1. `state` returned == `$STATE` you sent. Otherwise discard.
2. `iss` returned == the `issuer` from Step D1 metadata, **byte-for-byte**. Do not trim trailing slashes. Otherwise discard.
3. If either check fails, abort. Do not proceed to the token endpoint.

### Step D6 — Exchange code for tokens

Confidential client (Shopify retro-rewind-shop's advertised method is `client_secret_basic`):

```bash
curl -sL -X POST "$TOKEN_ENDPOINT" \
  -u "$CLIENT_ID:$CLIENT_SECRET" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "grant_type=authorization_code" \
  --data-urlencode "code=$AUTH_CODE" \
  --data-urlencode "redirect_uri=$REDIRECT_URI" \
  --data-urlencode "code_verifier=$CODE_VERIFIER"
```

Public client (no secret, PKCE-only):

```bash
curl -sL -X POST "$TOKEN_ENDPOINT" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "grant_type=authorization_code" \
  --data-urlencode "client_id=$CLIENT_ID" \
  --data-urlencode "code=$AUTH_CODE" \
  --data-urlencode "redirect_uri=$REDIRECT_URI" \
  --data-urlencode "code_verifier=$CODE_VERIFIER"
```

Expected response:

```json
{
  "access_token": "<opaque or JWT>",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "<opaque>",
  "scope": "<granted scopes, space-separated>"
}
```

### Step D7 — Use the access token

On subsequent UCP MCP requests, present the token in the standard HTTP header **in addition to** the `meta.ucp-agent.profile` field:

```bash
curl -sL -X POST "$MCP_ENDPOINT" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{ "jsonrpc":"2.0","id":1,"method":"tools/call","params":{ ... }}'
```

### Postconditions

* `$ACCESS_TOKEN` (and `$REFRESH_TOKEN`) are stored securely.
* Subsequent gated capability requests authenticate the user.
* On user unlink action, you **MUST** call the business's `revocation_endpoint` (RFC 7009) with the refresh token to invalidate the grant.

### Error handling

* HTTP `401 identity_required` from a UCP endpoint: token absent/invalid/expired. Refresh or re-initiate Step D4.
* HTTP `403 insufficient_scope`: token is good but missing scopes. Read the `scope=` parameter in the `WWW-Authenticate` header (the **full** required set for that operation), diff against scopes you already hold, and request just the missing ones via incremental authorization (re-run D3–D6 with the reduced set). **Do not** re-link from scratch.
* PKCE mismatch at token endpoint → `invalid_grant`. Restart from D3.
* Client auth failure at token endpoint → `invalid_client`. Check the auth method matches `token_endpoint_auth_methods_supported`.

---

## Quick reference

### MCP tool ↔ UCP operation

| Tool | Capability | Idempotency key required? |
| :--- | :--- | :---: |
| `search_catalog` | `dev.ucp.shopping.catalog.search` | no |
| `lookup_catalog`, `get_product` | `dev.ucp.shopping.catalog.lookup` | no |
| `create_checkout` | `dev.ucp.shopping.checkout` | no |
| `get_checkout` | `dev.ucp.shopping.checkout` | no |
| `update_checkout` | `dev.ucp.shopping.checkout` | no |
| `complete_checkout` | `dev.ucp.shopping.checkout` | **yes** |
| `cancel_checkout` | `dev.ucp.shopping.checkout` | **yes** |

### JSON-RPC error codes you'll actually see

| Code | Meaning | Likely cause |
| :--- | :--- | :--- |
| `-32600` | Invalid request | Malformed JSON-RPC envelope. |
| `-32601` | Method not found | Wrong tool name. |
| `-32602` | Invalid params | Schema validation failed. Re-check required fields. |
| `-32000` | Server / capability error | Generic UCP error. Read `error.data`. |
| `-32001` | UCP discovery / negotiation failed | Profile URL unreachable, malformed, or non-cacheable. Fix the platform profile in §0. |

### Money in UCP

All `amount` fields are **integers in minor units** of `currency` (e.g. cents for USD). Negative = subtractive. A `totals[]` array contains exactly one `subtotal` entry and exactly one `total` entry; do not aggregate or reorder when displaying — render in the order the business sent.

### Caching contracts

| Resource | Required cache headers | Notes |
| :--- | :--- | :--- |
| Your platform profile (`$AGENT_PROFILE_URL`) | `Cache-Control: public, max-age=N` with `N ≥ 60` | Must not redirect. HTTPS only. |
| Business profile (`/.well-known/ucp`) | Same | You as platform respect what they return. |

If you see `-32001 invalid_cache_control`, the offender is almost always your platform profile.

---

## Appendix — a minimal end-to-end happy-path session

```bash
# 0. one-time
export BUSINESS_HOST="https://retro-rewind-shop.myshopify.com"
export AGENT_PROFILE_URL="https://your-agent.example.com/.well-known/ucp"

# A. discover
curl -sL "$BUSINESS_HOST/.well-known/ucp" > /tmp/biz.json
MCP_ENDPOINT=$(jq -r '.ucp.services["dev.ucp.shopping"][] | select(.transport=="mcp") | .endpoint' /tmp/biz.json)

# B. search
curl -sL -X POST "$MCP_ENDPOINT" -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"search_catalog","arguments":{
    "meta":{"ucp-agent":{"profile":"'"$AGENT_PROFILE_URL"'"}},
    "catalog":{"query":"retro shirt","context":{"address_country":"US"},"pagination":{"limit":5}}
  }}}' > /tmp/search.json
VARIANT_ID=$(jq -r '.result.structuredContent.products[0].variants[0].id' /tmp/search.json)

# C1. create checkout
curl -sL -X POST "$MCP_ENDPOINT" -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"create_checkout","arguments":{
    "meta":{"ucp-agent":{"profile":"'"$AGENT_PROFILE_URL"'"}},
    "checkout":{
      "buyer":{"email":"jane@example.com","first_name":"Jane","last_name":"Doe"},
      "line_items":[{"item":{"id":"'"$VARIANT_ID"'"},"quantity":1}],
      "currency":"USD",
      "fulfillment":{"methods":[{"type":"shipping","destinations":[{
        "street_address":"123 Main St","address_locality":"Springfield","address_region":"IL",
        "postal_code":"62701","address_country":"US"
      }]}]}
    }
  }}}' > /tmp/checkout.json
CHECKOUT_ID=$(jq -r '.result.structuredContent.id' /tmp/checkout.json)
STATUS=$(jq -r '.result.structuredContent.status' /tmp/checkout.json)

# C2/C3. (collect payment via the handler defined in /tmp/checkout.json -> .ucp.payment_handlers)
# C4. complete
curl -sL -X POST "$MCP_ENDPOINT" -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"complete_checkout","arguments":{
    "meta":{"ucp-agent":{"profile":"'"$AGENT_PROFILE_URL"'"},"idempotency-key":"'"$(uuidgen)"'"},
    "id":"'"$CHECKOUT_ID"'",
    "checkout":{"payment":{"instruments":[ /* handler-issued instrument */ ]}}
  }}}'
```
