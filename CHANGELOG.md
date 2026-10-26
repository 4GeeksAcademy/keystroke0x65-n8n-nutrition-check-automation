# Changelog

All notable changes to the **SnackCheck - Nutrition Check** n8n workflow are documented here.
This project follows [Semantic Versioning](https://semver.org/) and the [Keep a Changelog](https://keepachangelog.com/) format.
  
## [1.0.0] - 2026-10-25

First release. Published to production after all 14 test cases passed (see TEST_LOG.md).

### Changed
- Both AI prompts ("AI - Health Verdict (Cautionary)" and "AI - Health Verdict (Encouraging)") now end with: "Only use the numbers and facts listed above. Do not mention serving sizes, can or bottle sizes, daily-intake percentages, or any figure that is not given in the product data." This fixed the Coca-Cola paragraph inventing a 330 ml can and a daily-limit claim (TC-002).

## [0.2.0] - 2026-10-25

### Changed
- `Validate - Has Nutrient Data?` now checks the raw Open Food Facts response instead of the extracted fields, so a missing value can never be coerced to 0 and classified as healthy.
- `Set - Format Verdict Message` reads upstream data with `.first()` instead of `.item`, avoiding paired-item errors after the LLM chain and Merge.
- Renamed the `README` sticky note to `Note - README` to follow the `[Action] - [Purpose]` naming convention.
- Workflow README note: documented the empty-body versus missing-barcode behaviour, the "insufficient data even if a Nutri-Score exists" rule, the outage behaviour and the missing-single-nutrient rule.

## [0.1.0] - 2026-10-25

### Added
- Webhook `POST /nutrition-check`, with self-documenting usage JSON for an empty body.
- Barcode validation (present and digits only) with a 400 JSON error and example.
- Open Food Facts lookup with "never error" handling so the 404 body reaches the workflow, plus a 404 JSON response for unknown products.
- Field extraction into flat variables, including `energy-kcal_100g` via bracket notation.
- Insufficient-data check and JSON response.
- Traffic-light classification (sugar, salt, fat per 100 g), concern score in a single-item Code node, and verdict level as the worse of the traffic-light and Nutri-Score readings, with a traffic-light-only fallback.
- Conditional AI routing: cautionary prompt for `unhealthy`, encouraging prompt for `healthy` and `moderate`, rejoined with a Merge (Append).
- Formatted plain-text verdict with emoji headline, red/green markers, Nutri-Score, concern score and AI paragraph.
- Inline notes on every node, a README sticky note and stage sticky notes.

### Notes
- The AI step uses Anthropic Chat Model nodes (Claude Sonnet 4.6) rather than the Groq node named in the course brief.