# Offer Extraction Pipeline (AT&T iPhone offers → CSV)

Takes the scraped product JSON and creates a CSV with 29 columns.
Every product gets 2 rows: one for "New Line" and one for "Upgrade".

## How it works
1. **Load data** – read Input.json, take basic fields directly (name, brand, variant, price, stock, shipment).
2. **Collect offer texts** – join title, description, modal text and card legal text of each offer.
   48 offers turn into 11 unique texts, so only 11 LLM calls are needed.
3. **LLM extraction (Gemini)** – each unique text goes to Gemini with a Pydantic schema
   (structured JSON output, temperature 0). The model returns category, eligible line types,
   amounts (copied exactly from the text), variant of each amount, and a short description.
   Results are saved in llm_cache.json, so re-running does not call the API again.
4. **Validation** – every amount returned by the LLM must appear in the source text,
   otherwise it is dropped (hallucination check). Unknown kinds are also dropped.
5. **Rules (Python)** – for every product and line type:
   - offers not valid for that line type are skipped (e.g. switcher only for New Line)
   - device discounts are taken only for the matching storage variant
   - one value goes to only one column
   - finance: New Line = welcome bonus + yearly credit + bill discount, Upgrade = only recurring parts
6. **Output** – output.csv with 24 rows (12 products × 2).

## Error handling
- Retry with backoff for server errors (503) and malformed JSON.
- 429 rate limit: wait 60s. If the daily quota of a model is over, switch to the next model in the list.
- During development the free daily quota did run out, so the model fallback was actually used.

## Assumptions
- BASE and MSRP both come from `pricing.retail` (PDF says "Base retail").
- MSRP is written without `$` as in the PDF example; offer amounts use `$`.
- Amounts like "$1,200" are written as "$1200".
- RESELLER LIST PRICE, PWP/GWP, subscription and additional offers are left empty
  (not clearly defined in the brief / not present in the data).
- Trade-In Value Type comes from the LLM ("Enhanced" for promo trade-in deals).

## How to run
1. Open the notebook in Google Colab.
2. Add your Gemini API key in Colab Secrets as `GEMINI_API_KEY`.
3. Upload Input.json (and llm_cache.json if you want to skip the API calls).
4. Runtime → Run all. The output is saved as output.csv.
