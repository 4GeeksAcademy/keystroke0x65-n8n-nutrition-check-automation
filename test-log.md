# SnackCheck - Nutrition Check: Test Log

- **Workflow:** SnackCheck - Nutrition Check
- **Endpoint tested:** production webhook `POST /webhook/nutrition-check`
- **Test date:** 2026-10-25
- **Tester:** Ash
- **Result:** 14 of 14 test cases passed.

Two builds were tested. Cases marked **(pre-fix)** ran before the prompt hardening in v1.0.0; cases marked **(v1.0.0)** ran on the published version `202866d0`. Where a case was re-run after the fix, both results are shown.

---

## Functional tests (valid barcode, verdict)

### TC-001 Unhealthy product, cautionary tone
- **Input:** `{"barcode":"3017620422003"}` (Nutella)
- **Expected:** 200 `text/plain`, `UNHEALTHY`, sugar and fat red, salt green, concern 2/3, Nutri-Score E, cautionary paragraph.
- **Actual (pre-fix and v1.0.0):** 200 `text/plain; charset=utf-8`. Sugar 56.3 g 🔴, salt 0.107 g 🟢, fat 30.9 g 🔴, energy 539 kcal, Nutri-Score E, concern 2/3, `🔴 ... UNHEALTHY`, cautionary paragraph that suggests an occasional treat.
- **Result:** Pass

### TC-002 Soda caught by Nutri-Score (worse-of-two rule)
- **Input:** `{"barcode":"5449000000996"}` (Coca-Cola Original)
- **Expected:** `UNHEALTHY` even though no nutrient is `high`. Concern 0/3, all nutrient markers green, Nutri-Score D or E.
- **Actual (pre-fix):** UNHEALTHY, concern 0/3, all three nutrient lines 🟢, Nutri-Score E. The paragraph invented a can size ("330 ml", "around 35 g of sugar") and a daily-limit claim that were not in the data.
- **Actual (v1.0.0):** UNHEALTHY, concern 0/3, all 🟢, Nutri-Score E. The paragraph uses only the per-100 g figures. No can size, serving size or daily-limit claim.
- **Result:** Pass (the pre-fix paragraph defect was fixed by the prompt change, see CHANGELOG 1.0.0)

### TC-003 Nutri-Score A product, encouraging tone
- **Input:** `{"barcode":"3229820129488"}` (Bjorg muesli with raisins, dates and figs)
- **Expected:** 200 `text/plain`, `HEALTHY` with the ✅ headline, Nutri-Score A, concern 0/3, encouraging paragraph with no negatives.
- **Actual (pre-fix):** `✅ ... HEALTHY`, sugar 14 g 🟢, salt 0.03 g 🟢, fat 5.8 g 🟢, energy 352 kcal, Nutri-Score A, concern 0/3, encouraging paragraph with no negatives. Only the encouraging branch ran and the Merge passed it through.
- **Result:** Pass

### TC-004 Moderate level from Nutri-Score C, portion tip
- **Input:** `{"barcode":"3017800238592"}` (D'Aucy curry vert de légumes riz)
- **Expected:** `MODERATE` (🟡), Nutri-Score C, concern 0/3, encouraging tone with a portion tip.
- **Actual (v1.0.0):** `🟡 ... MODERATE`. Sugar 1.6 g, salt 0.93 g, fat 5.7 g (all 🟢), energy 144 kcal, Nutri-Score C, concern 0/3. Encouraging paragraph with a tip about portions and no scary language.
- **Result:** Pass

### TC-005 Unknown Nutri-Score falls back to traffic-light count
- **Input:** `{"barcode":"3183280000896"}` (La Baleine sea salt, Nutri-Score "not-applicable")
- **Expected:** `Nutri-Score: n/a`. Level taken from the traffic-light count alone: 1 high nutrient gives `MODERATE`.
- **Actual (v1.0.0):** `🟡 ... MODERATE`. Sugar 0 g 🟢, salt 98 g 🔴, fat 0 g 🟢, Nutri-Score n/a, concern 1/3. Encouraging paragraph with a portion tip.
- **Result:** Pass
- **Note:** Pure salt rates only `moderate` because it counts as one high nutrient. This follows the founder's rules as written.

---

## Integration tests (live Open Food Facts API)

### TC-006 Live lookup with barcode injected and fields extracted
- **Input:** `{"barcode":"3017620422003"}`
- **Expected:** The API call succeeds with the barcode in the URL. `product_name`, `brands`, `nutriscore_grade`, `sugars_100g`, `salt_100g`, `fat_100g` and the hyphenated `energy-kcal_100g` (bracket notation) all appear in the verdict.
- **Actual:** Product name, brand, all three nutrients, Nutri-Score and energy (539 kcal) all appeared correctly. Same check passed for the other four products in TC-002 to TC-005.
- **Result:** Pass

### TC-007 Open Food Facts 404 reaches the workflow without crashing
- **Input:** `{"barcode":"0000000000000"}`
- **Expected:** The HTTP node does not fail on the upstream 404. The body (`status: 0`) is read and routed to the not-found response.
- **Actual:** The workflow completed and returned the structured 404 from TC-012. No execution error.
- **Result:** Pass

---

## Error tests

### TC-008 Empty body returns usage docs
- **Input:** `{}`
- **Expected:** 200 JSON with service, method, required field, example request and response description.
- **Actual:** HTTP 200, `application/json; charset=utf-8`, body contains `service`, `description`, `method`, `required_field`, `example_request` (`{"barcode":"3017620422003"}`) and `response`.
- **Result:** Pass

### TC-009 Non-numeric barcode
- **Input:** `{"barcode":"abc"}`
- **Expected:** 400 JSON with a helpful message and example.
- **Actual:** HTTP 400, `application/json; charset=utf-8`, `{"error":"invalid_barcode","message":"The 'barcode' field is required and must contain digits only.","example":{"barcode":"3017620422003"}}`
- **Result:** Pass

### TC-010 Body present, barcode field missing
- **Input:** `{"foo":1}`
- **Expected:** 400 `invalid_barcode` (not the usage docs).
- **Actual:** HTTP 400, same `invalid_barcode` body as TC-009.
- **Result:** Pass

### TC-011 Empty-string barcode
- **Input:** `{"barcode":""}`
- **Expected:** 400 `invalid_barcode`.
- **Actual:** HTTP 400, same `invalid_barcode` body as TC-009.
- **Result:** Pass
- **Note:** Only a truly empty body (`{}`) returns the usage docs. This is documented in the README.

### TC-012 Product not in the database
- **Input:** `{"barcode":"0000000000000"}`
- **Expected:** 404 JSON with a friendly message and a valid example barcode.
- **Actual:** HTTP 404, `application/json; charset=utf-8`, `{"error":"product_not_found","message":"No product with this barcode exists in the Open Food Facts database.","example":{"barcode":"3017620422003"}}`
- **Result:** Pass

### TC-013 Product found but has no nutrition data
- **Input:** `{"barcode":"3596710355440"}` (Bière blonde 4.2%, "Nutrition facts not specified")
- **Expected:** 200 JSON `insufficient_data`. No classification, no AI call, never `healthy`.
- **Actual:** HTTP 200, `application/json; charset=utf-8`, `{"status":"insufficient_data","message":"This product was found but doesn't have enough nutrition data to give a verdict.","product":"Bière blonde -4,2%","example":{"barcode":"3017620422003"}}`
- **Result:** Pass
- **Note:** The data check reads the raw API response, so missing values are never coerced to 0.

---

## Performance

### TC-014 Response time
Measured from n8n execution records, start to finish, on v1.0.0.

| Scenario | Execution | Duration (ms) |
|---|---|---|
| Nutella, full path with AI | 140 | 3,385 |
| Empty body `{}`, usage docs | 141 | 37 |
| Unknown barcode, 404 | 142 | 121 |
| Coca-Cola (reference) | 137 | 5,485 |
| Curry (reference) | 138 | 4,443 |
| Sea salt (reference) | 139 | 2,870 |

- **Expected:** Non-AI paths return in well under a second. AI paths return in a few seconds.
- **Actual:** Non-AI paths took 37 to 121 ms. AI paths took 2.9 to 5.5 s, so the AI call accounts for most of the time. Per-node timing was not available from the execution records, so the AI node's own duration was not measured.
- **Result:** Pass

---

## Summary
  
| ID | Type | Case | Result |
|---|---|---|---|
| TC-001 | Functional | Nutella, unhealthy, cautionary | Pass |
| TC-002 | Functional | Coca-Cola, Nutri-Score override | Pass |
| TC-003 | Functional | Nutri-Score A, healthy, encouraging | Pass |
| TC-004 | Functional | Nutri-Score C, moderate, portion tip | Pass |
| TC-005 | Functional | Unknown Nutri-Score fallback | Pass |
| TC-006 | Integration | Live API call and field extraction | Pass |
| TC-007 | Integration | Upstream 404 handled without crash | Pass |
| TC-008 | Error | Empty body, usage docs | Pass |
| TC-009 | Error | Non-numeric barcode | Pass |
| TC-010 | Error | Barcode field missing | Pass |
| TC-011 | Error | Empty-string barcode | Pass |
| TC-012 | Error | Product not found | Pass |
| TC-013 | Error | No nutrition data | Pass |
| TC-014 | Performance | Response times | Pass |

## Known observations (not failures)

1. **Facts beyond the data:** v1.0.0 cautionary paragraphs still occasionally add general statements, such as calling Nutri-Score E "the lowest rating on the scale", and one encouraging paragraph said seeds "add a nice nutritious touch". These are not numbers, but they are not in the product data.
2. **Drinks:** Coca-Cola's values are per 100 ml but are labelled "per 100 g".
3. **Open Food Facts availability:** during test preparation some lookups from the test environment failed or returned nothing, apparently from rate limiting. If the API is down, callers see `product_not_found`.