# Requesty's EU endpoint next to the Bedrock knowledge base

Draft in progress, 2026-11-21.

## Notes so far

- EU endpoint `https://router.eu.requesty.ai/v1`, Frankfurt, AWS eu-central-1 (docs.requesty.ai/features/eu-routing).
- The EU endpoint covers only Requesty's own processing. EU inference needs EU models plus the "Model hosting location" restriction.
- Public model list without a key: 856 models, 254 with geolocation `eu`.
- Legal entity Requesty Ltd, London, UK. Infrastructure sub-processor AWS EMEA SARL in Frankfurt.
- Logging of prompts and outputs is on by default, 30 days, can be turned off (zero data retention).
- 5 percent fee on top-ups. No rerank endpoint.
