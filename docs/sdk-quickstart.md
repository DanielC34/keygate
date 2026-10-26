# License API quickstart

This guide walks through issuing a license, activating a device, verifying
it, and deactivating it. The examples use the public SDK API and do not need
an API key for those public SDK calls. The admin license-creation example
requires an admin session or an appropriately scoped API key.

For a local Compose instance, set the server URL before following the steps:

```sh
KEYGATE_URL="http://localhost:9000"
```

## 1. Issue a license

Complete first-run setup and sign in to the Keygate dashboard. Create a
product that supports device activations and a plan for it, then issue a
license for a customer. SaaS products do not support device activation;
floating plans use the separate floating-license workflow.

An authorized server-side integration can create the license with
`POST /api/v1/admin/licenses`. This is an admin operation and requires an
admin session or an API key with the `admin` or `licenses:write` scope:

To use the API-key example, create an API key with one of those scopes in
Keygate and set `KEYGATE_ADMIN_API_KEY` to that key in your shell. Do not
embed the key in an SDK client. Alternatively, make this request from an
authenticated admin session.

```sh
curl -i -X POST "$KEYGATE_URL/api/v1/admin/licenses" \
  -H "Authorization: Bearer $KEYGATE_ADMIN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "product_id": "<product-uuid>",
    "plan_id": "<plan-uuid>",
    "email": "customer@example.com"
  }'
```

The response is `201 Created` and includes the newly issued `license_key`.
That plaintext key is returned only when creating a license (or through the
audited key-reveal endpoint); handle it like a credential. The key is also
emailed to the customer on a best-effort basis. Use a real product and plan
ID belonging to that product.

## 2. Get the verification public key

Set the issued license key and a stable device identifier:

```sh
LICENSE_KEY="<license-key-from-the-dashboard-or-create-response>"
DEVICE_ID="stable-unique-device-id"
API="$KEYGATE_URL/api/v1"
```

Fetch the server's public Ed25519 key. It is public and may be cached; the
response is a 32-byte key encoded as 64 hexadecimal characters.

```sh
curl -sS "$API/license/pubkey"
```

```json
{
  "success": true,
  "data": {
    "algorithm": "ed25519",
    "format": "hex",
    "public_key": "<64-hex-character-public-key>"
  }
}
```

## 3. Activate the device

`license_key` is the credential; do not send an API key from an SDK client.
Use a stable identifier so retries and later verification refer to the same
device. `identifier_type` is optional and defaults to `device`; `label` is
optional. `Idempotency-Key` is optional but recommended when retrying an
activation request. Reuse it only for the same request.

```sh
curl -i -X POST "$API/license/activate" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: <unique-request-id>" \
  -d "{
    \"license_key\": \"$LICENSE_KEY\",
    \"identifier\": \"$DEVICE_ID\",
    \"identifier_type\": \"device\",
    \"label\": \"Docs test device\"
  }"
```

A successful response is `200 OK`. The status is `activated` for a new
activation or `already_activated` if that identifier is already activated.
Both include a signed token:

```json
{
  "success": true,
  "data": {
    "status": "activated",
    "license_id": "<license-uuid>",
    "token": "<base64url-payload>.<base64url-signature>",
    "meta": {
      "server": "Keygate",
      "url": "https://keygate.app"
    }
  }
}
```

## 4. Verify on launch

Verify the license and device identifier when the client checks in. A `200`
response includes the current status, plan, features, and a newly signed
token. `valid_until` and `updates_until` are omitted when they do not apply.

```sh
curl -i -X POST "$API/license/verify" \
  -H "Content-Type: application/json" \
  -d "{
    \"license_key\": \"$LICENSE_KEY\",
    \"identifier\": \"$DEVICE_ID\"
  }"
```

The response uses the standard envelope; its `data` includes fields such as:

```json
{
  "success": true,
  "data": {
    "status": "active",
    "plan_id": "<plan-id>",
    "plan_name": "<plan-name>",
    "features": {},
    "token": "<base64url-payload>.<base64url-signature>",
    "grace_days": 0,
    "meta": {
      "server": "Keygate",
      "url": "https://keygate.app"
    }
  }
}
```

The values above illustrate the response shape; status, plan, feature values,
and optional fields depend on the license.

Successful license-creation, activation, verification, and deactivation
examples in this guide are illustrative and confirmed from source/OpenAPI,
not live-tested. Live API checks covered public-key retrieval and unsuccessful
verification cases only.

## 5. Verify a token offline

The token is **not a JWT**. It is
`base64url(payload).base64url(signature)`, using unpadded URL-safe Base64.
Split on the dot, decode the signature, and verify it with Ed25519 using the
public key from `/license/pubkey`. The signed bytes are the first (encoded
payload) segment itself, not the decoded JSON. Reject malformed tokens and
invalid Ed25519 signatures. Only after verifying the signature should a client
decode the payload and enforce `exp` (Unix seconds), rejecting expired tokens.
Compare the signed `did` with the expected device identifier, and check `fpr`
against the expected device and product context as applicable. `exp` is the
token's check-in deadline, not the license end date; `vun` carries the license
end date when present.

Offline verification uses the public key and token only. It does not replace
periodic calls to `/license/verify`, which check the license and activation
against the server.

## 6. Deactivate when the device is retired

Deactivation removes this identifier's activation and frees its slot. It
does not revoke the license.

```sh
curl -i -X POST "$API/license/deactivate" \
  -H "Content-Type: application/json" \
  -d "{
    \"license_key\": \"$LICENSE_KEY\",
    \"identifier\": \"$DEVICE_ID\"
  }"
```

Success is `200 OK`:

```json
{
  "success": true,
  "data": {
    "status": "deactivated"
  }
}
```

## Errors to handle

API errors use the same JSON envelope, with `success: false` and an `error`
containing a machine-readable `code` and `message`.

```json
{
  "success": false,
  "error": {
    "code": "LICENSE_NOT_FOUND",
    "message": "license not found"
  }
}
```

- `400 BAD_REQUEST`: missing fields or an invalid/blank/too-long identifier.
  Activation also rejects an unsupported `identifier_type` or an overlong
  label.
- `403` on activation: the license cannot currently be used
  (`LICENSE_EXPIRED`, `LICENSE_CANCELED`, `LICENSE_SUSPENDED`,
  `LICENSE_REVOKED`, or `LICENSE_INVALID`). A paused member seat can also
  return `SEAT_PAUSED`.
- `404 LICENSE_NOT_FOUND`: verification intentionally uses the same response
  for an unknown, unusable, or not-activated license/device pair. Deactivation
  also uses it when the key or activation is not found; activation uses it
  when the license key is not found. `FEATURE_NOT_AVAILABLE` means the product
  does not support that device operation.
- `409 ACTIVATION_LIMIT`: the plan's activation limit is reached.
  `409 DEVICE_HELD` means a member key attempted to activate a device already
  assigned to the license owner's key or another member's key. A legacy
  device activated with the shared license key before member keys existed may
  transfer to the member instead. `409 FLOATING_LICENSE` means use the
  floating checkout workflow instead.
- Activation idempotency can return `409 IDEMPOTENCY_IN_FLIGHT` with
  `Retry-After: 1`, or `422 IDEMPOTENCY_KEY_CONFLICT` if a key is reused with
  a different request body.
- `429 LOCKED_OUT` or `429 RATE_LIMITED`: back off; a `Retry-After` header is
  provided where applicable.

## Notes on the issue #14 examples

The proposed SDK paths and core activation/verification fields match the
current API. `identifier_type` may be omitted (it defaults to `device`), and
the `Idempotency-Key` on activation is optional. This guide adds the response
envelopes, deactivation, error behavior, and the token's actual Ed25519
format. The proposed usage and floating-license calls are separate workflows
and are not needed for this basic device-license flow.

For the full endpoint contract and additional cases, see the
[OpenAPI specification](./openapi.yaml) and [API contract](./api-contract.md).
