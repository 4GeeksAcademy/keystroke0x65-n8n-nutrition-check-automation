# SnackCheck - Nutrition Check (n8n workflow)

## Purpose

Turns a packaged-food barcode into a short, friendly health verdict. An app (SnackCheck) sends a barcode to a webhook; the workflow looks the product up on Open Food Facts (free, no key), classifies it as `healthy`, `moderate` or `unhealthy`, has an LLM write a short paragraph whose tone matches the level, and returns a formatted plain-text message.

## How It Works

1. **Webhook** receives `POST /nutrition-check` with `{ "barcode": "..." }`.
2. **Empty body?** `{}` returns self-documenting JSON usage docs (200).
3. **Barcode valid?** The barcode must be present and digits only, otherwise a 400 JSON error with an example.
4. **API call** to Open Food Facts with the barcode in the URL. The HTTP node is set to *never error*, so a 404 body (`status: 0`) reaches the next check.
5. **Product found?** `status === 1`, otherwise a 404 JSON response.
6. **Extract fields** into flat variables: `product_name`, `brands`, `nutriscore_grade`, `sugars_100g`, `salt_100g`, `fat_100g`, and `energy_kcal_100g` (read with bracket notation `['energy-kcal_100g']` because of the hyphen).
7. **Has nutrient data?** Checked against the raw API response. If sugar, salt and fat are all missing, return `insufficient_data` JSON (200) and stop.
8. **Classify** each nutrient per 100 g:
   - Sugar: low < 5 g, medium 5 to 22.5 g, high > 22.5 g
   - Salt: high > 1.5 g
   - Fat: high > 17.5 g
9. **Concern score** (Code node, single item): the count of `high` nutrients, 0 to 3.
10. **Verdict level** = the worse of two readings:
    - Traffic-light: 0 high = healthy, 1 = moderate, 2 or 3 = unhealthy
    - Nutri-Score: a/b = healthy, c = moderate, d/e = unhealthy
    - If `nutriscore_grade` is unknown or missing, the traffic-light reading is used alone.
11. **Route by tone:** `unhealthy` goes to the cautionary AI prompt; `healthy` and `moderate` go to the encouraging AI prompt (with a portion tip for moderate).
12. **Merge (Append)** rejoins the two branches (only one runs per request).
13. **Format** the message (emoji headline, nutrient lines with red/green markers, Nutri-Score, concern score, AI paragraph) and **respond** as `text/plain`.

## Setup

- Import the workflow JSON into n8n and publish it so the production URL is live.
- **LLM credential:** the two Chat Model sub-nodes use an **Anthropic** credential (Claude Sonnet 4.6, temperature 0.6). The course brief names Groq; to use Groq instead, replace the two Anthropic Chat Model nodes with Groq Chat Model nodes and select your Groq credential. Nothing else changes.
- Open Food Facts needs no key and no signup.

## Usage

```
POST https://<your-n8n-host>/webhook/nutrition-check
Content-Type: application/json

{ "barcode": "3017620422003" }
```

Successful response (`200`, `text/plain; charset=utf-8`):

```
🔴 Nutella (Nutella, Ferrero) - UNHEALTHY

Per 100 g:
Sugar:  56.3 g  🔴
Salt:   0.107 g  🟢
Fat:    30.9 g  🔴
Energy: 539 kcal
Nutri-Score: E
Concern score: 2/3

<short AI paragraph>
```

Markers are red only when a nutrient is `high`. A `medium` value (for example 10.6 g sugar) shows green.

## Error Handling

| Situation | Status | Content-Type | Body |
|---|---|---|---|
| Empty body `{}` | 200 | JSON | Usage docs (service, required field, example) |
| Body present but barcode missing, empty or non-numeric (`{"foo":1}`, `{"barcode":""}`, `{"barcode":"abc"}`) | 400 | JSON | `invalid_barcode` with message and example |
| Barcode not in Open Food Facts | 404 | JSON | `product_not_found` with example |
| Product found, but sugar, salt and fat are all missing | 200 | JSON | `insufficient_data` (no verdict, even if a Nutri-Score exists) |
| Valid product with data | 200 | text/plain | Formatted verdict |

## Limitations  

- Data quality depends on Open Food Facts contributors.
- Thresholds are per 100 g, not per portion. Drinks are really measured per 100 ml but are labelled "per 100 g".
- If Open Food Facts is down, rate-limited or returns a malformed response, the caller sees `product_not_found`, because the HTTP node is set to never error.
- A single missing nutrient (when not all three are missing) is treated as 0 when classifying.
- A product with no data for all three nutrients gets no verdict, even if it has a Nutri-Score.
- A product with a single very high value (for example pure salt at 98 g) counts as one high nutrient and is rated `moderate`, as the rules specify.
- The AI paragraph is instructed to use only the supplied numbers, but it may still add general statements (for example describing what Nutri-Score E means). It is advisory, not medical advice.
- One barcode per request; AI calls add roughly 3 to 5 seconds.