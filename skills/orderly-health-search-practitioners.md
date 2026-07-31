---
name: Search the Orderly Provider Directory
description: Look up healthcare practitioners in the Orderly Provider Directory by NPI, name, location, specialty, care category, accepted insurance, or DEA number, with confidence-scored fields and offset pagination.
api: openapi/orderly-health-provider-directory-openapi.json
operations:
  - "POST /opd/practitioner/search"
---

# Search the Orderly Provider Directory

Use this skill to query the Orderly Provider Directory API for practitioner
records. The directory blends multiple data sources and returns the
highest-confidence value per field, with a confidence score and a source label.

## Prerequisites

- A demo (or production) Orderly account. Request one from
  https://orderlyhealth.com/orderly-provider-directory/.
- An API token minted from the "API tokens" area of https://app.orderlyhealth.com.
  Copy the token once — it cannot be retrieved again.

## Authentication

Send the token in the `Authorization` header. The value copied from the portal
already includes the `Basic ` prefix (token passed over Basic Auth):

```
Authorization: {Your Encoded Authorization Header}
Content-Type: application/json
Accept: application/json
```

A missing or invalid token returns `401` with `{ "code": 401, "error": "..." }`.

## Steps

1. **Build the search body.** Every filter type is combined with AND; multiple
   values inside one filter are combined with OR. Common fields:
   - `npi` — exact National Provider Identifier lookup.
   - `first_name` / `last_name` — name search.
   - `zip` + `distance` — geographic radius search (miles).
   - `states` — array of full state names (e.g. `["Colorado"]`).
   - `specialties` — NUCC specialty names.
   - `practitioner_types` — e.g. `["Physician"]`.
   - `care_categories`, `payer_names`, `drug_enforcement_agency_numbers`.
   - `required_fields` — force inclusion of `email`, `fax`, `credentials`,
     `insurance_accepted`, or `facility_affiliations`.
   - `allow_sanctioned` — set `true` to include sanctioned providers.

2. **Call the operation.** `POST /opd/practitioner/search` on
   `https://api.orderlyhealth.com`.

   ```bash
   curl --request POST \
     --url https://api.orderlyhealth.com/opd/practitioner/search \
     --header 'Authorization: {Your Encoded Authorization Header}' \
     --header 'Content-Type: application/json' \
     --data '{"last_name":"Williams","states":["Colorado"],"size":25}'
   ```

3. **Page through results.** The response has `total_hits` (full match count) and
   `practitioners` (this page). Use `size` (default 25) and `from` (zero-based
   offset) to iterate: request `from: 0`, then `from: 25`, etc., until
   `from >= total_hits`.

4. **Read confidence + source.** For address/phone/fax fields, pair each value
   with its `*Confidence` (probability the value is still correct today) and
   `*Source` (Orderly Data, Curated Public Data, Partner Data, or Phone
   Attested). Prefer higher-confidence values for outreach.

## Error handling

Errors return `{ "code": <int>, "error": <string> }`:
- `400` Bad Request — malformed search body.
- `401` Unauthorized — check the Authorization header/token.
- `500` Orderly Internal Error — retry with backoff; the search is a read and is
  safe to retry (no idempotency key is required).
