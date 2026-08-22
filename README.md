### Prices that aren't a single number

A growing share of models don't have *a* price. They have one below 200k prompt tokens and
another above it, or one during peak hours and another off-peak. Every published price table
I could find prints one number per row anyway, and the gap is usually a factor of two.

**[What a coding-agent model costs, next to how it ranks](https://xyzs996.github.io/llm-api-pricing/prices)**
— 60 ranked models, regenerated from the source catalog, with the parts other tables drop:

- 12 of the 60 cost more than the price printed in their own row, and the trigger is prompt
  length. It's a cliff, not a tier: one token over the line roughly doubles the bill.
- The bigger the advertised context window, the smaller the share of it the advertised price
  covers. `grok-4.20` advertises 2 000k of context at $1.25/M and steps to $2.50 at 200k, so
  **90% of that window bills at a number that isn't in the row.**
- List price is not the bill. A coding agent re-reads its context every step, so ~95.6% of the
  tokens it sends are cache reads. Repriced at that mix, list input price overstates what an
  agent pays by a median 6.5x — and the 3.4x–7.9x spread in that multiple is what actually
  separates two models whose list prices look identical.

Every claim on that page says where its number came from and on what date, and the weights
ship in the JSON so you can recompute with your own token mix.

### Sending the corrections upstream

Where the same defect exists in a public catalog, it goes back as a PR rather than staying a
footnote on my page. Open right now:

| | |
|---|---|
| [genai-prices#583](https://github.com/pydantic/genai-prices/pull/583) | 6 models priced as if long-context were a marginal tier; ~2x under-priced past the threshold |
| [genai-prices#584](https://github.com/pydantic/genai-prices/pull/584) | `grok-4.6` missing entirely — pricing it raises `LookupError` |
| [genai-prices#581](https://github.com/pydantic/genai-prices/pull/581) | DeepSeek V4 peak/off-peak repricing |
| [models.dev#5277](https://github.com/anomalyco/models.dev/pull/5277) | `context_over_200k` documented as a flat 200K; 193 of 357 entries carrying it aren't |
| [models.dev#5280](https://github.com/anomalyco/models.dev/pull/5280) | DeepInfra cache-read price derived two different ways; 1 of 17 tier segments disagrees with itself |
| [litellm#37930](https://github.com/BerriAI/litellm/pull/37930) | `wandb/*` prices stored as $/1M in a per-token field — 100,000x off |
| [litellm#37932](https://github.com/BerriAI/litellm/pull/37932) | 3 realtime rows whose cache-read price contradicts their own schema |
| [llm-prices#67](https://github.com/simonw/llm-prices/pull/67), [#68](https://github.com/simonw/llm-prices/pull/68), [#71](https://github.com/simonw/llm-prices/pull/71) | Missing cached-input prices for 16 Claude/Gemini models; DeepSeek V4 repricing |

### The other lists

Same rule: single purpose, regenerated automatically, every number says where it came from.

| | |
|---|---|
| [free-llm-api](https://github.com/xyzs996/free-llm-api) | Verified free LLM API tiers: rate limits, no-card options, OpenAI-compatible endpoints. |
| [free-proxy-health-list](https://github.com/xyzs996/free-proxy-health-list) | Free HTTP/SOCKS proxy health list — verified, JSON/TXT/CSV, updated automatically. |
| [iptv-doctor](https://github.com/xyzs996/iptv-doctor) | M3U/M3U8 playlist checker and XMLTV EPG fixer. |

Corrections are welcome on any of them — open an issue on the repo in question. If you maintain
a pricing catalog and want the numbers above as a diff against your format, say so on the repo
and I'll send one.
