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
ship in the JSON so you can recompute with your own token mix. Table, data and generator are in
[llm-api-pricing](https://github.com/xyzs996/llm-api-pricing); if a number looks wrong, an issue
there is the fastest way to get it changed.

### The same table, as a bill

**[Put your own token counts in](https://xyzs996.github.io/llm-cost-calculator/)** — one HTML
file, nothing to install, no account. It reads the same JSON as the table above, so it is never
a day behind it, and it does the three things a rate card structurally cannot:

- **Peak/off-peak resolved live.** DeepSeek's weekend rule changed on 2026-08-23: off-peak all
  weekend, at half price ([which hours, and why every summary I could find gets it wrong](https://xyzs996.github.io/llm-api-pricing/deepseek-peak-hours.html)). The page says which phase you are in right now, what the same call
  costs in the other one, and how long until it flips. The Beijing weekend runs Fri 16:00 → Sun
  16:00 UTC, and both official peak windows sit clear of that seam — so a `getUTCDay()`
  implementation returns *identical prices for all 168 hours of a week* and no test written
  against the published windows can tell.
- **The long-context cliff.** One token past a threshold reprices **every** token in the
  request, including the ones below it. At `gemini-3.1-pro-preview`'s 200 000 line, one extra
  token takes the same call from $0.2060 to $0.4090.
- **Your mix, not the median.** The default is a coding agent's ~95.6% cache-read share. Slide
  it to what your own logs say and the ordering of the cheapest ten changes.

Source and issues: [llm-cost-calculator](https://github.com/xyzs996/llm-cost-calculator).

The clock cases that trip implementations are published separately as a fixture anyone can
depend on — [deepseek-peak-hours](https://github.com/xyzs996/deepseek-peak-hours),
plain JSON, no dependency, usable from any language. Seven projects have taken a fix found by
running them, and the bundled harness scores nine published DeepSeek billing plugins against the
same vectors: on 2026-08-23, two passed.

### Where the numbers came from

The table is the artifact; these are the arguments behind it. Each one leads with a figure you
can check, and each is posted as a thread with a reply box — if a number is wrong, that is the
place to say so.

| | |
|---|---|
| [Chinese models are not 2x cheaper once your agent starts caching](https://github.com/xyzs996/llm-api-pricing/discussions/66) | On list price the median Chinese model is 2.47x cheaper. Repriced at a coding agent's real token mix it is 1.51x. Nothing expired — the comparison was reading the input column, and an agent spends 95.6% of its tokens on cache reads. |
| [Does your DeepSeek cost code read the weekday off the shifted clock?](https://github.com/xyzs996/llm-api-pricing/discussions/44) | The bug behind the four merged fixes below. A `getUTCDay()` implementation returns identical prices for all 168 hours of the week, so no test written against the published windows can catch it. Four projects had it; each was found by running the same eight timestamps. |
| [The two best AI code reviewers score the same. One costs $1.43 a run, the other $9.05](https://github.com/xyzs996/llm-api-pricing/discussions/36) | 43.1% vs 41.2% Pass@1 on ReactBench — two points apart, 6.3x apart on price. |
| [Your AI coding bill scales with your repo, not your output](https://github.com/xyzs996/llm-api-pricing/discussions/56) | The dominant cost is re-reading project context, not the code the model writes. That is why the bill surprises people in month three and why file layout turns out to be cost engineering. |
| [1.6 billion free tokens is a compression ratio, not a strategy](https://github.com/xyzs996/llm-api-pricing/discussions/12) | The advertised quota is 10,000 tokens compressed to 1,080, times a free tier. It tells you nothing about which model answers your next request, which is the only thing your bill depends on. |

All 53 are indexed at [the writing list](https://xyzs996.github.io/llm-api-pricing/), rendered
from the same repo as the table.

### Sending the corrections upstream

Where the same defect exists in a public catalog, it goes back as a PR — or, where I cannot
sign the project's CLA, as an issue with the worked example attached — rather than staying a
footnote on my page.

Landed, all the same defect — the weekday is read off the raw UTC instant, so the Beijing
weekend is billed at 2x for eight hours at each end:

| | |
|---|---|
| [OmniRoute#11210](https://github.com/diegosouzapw/OmniRoute/pull/11210) | merged — the price table's own comment said the discount ran every day |
| [LangAlpha#365](https://github.com/ginlix-ai/LangAlpha/pull/365) | merged, with the schedule lifted out of the code into the provider manifest and two test files pinning it |
| [llmgateway#3776](https://github.com/theopenco/llmgateway/pull/3776) | merged — a test that fails if the off-peak day is ever read off the UTC clock again |
| [CodeWhale#5545](https://github.com/Hmbown/CodeWhale/pull/5545) | merged |
| [TokenTracker#505](https://github.com/xiufengsun/TokenTracker/pull/505) | merged |
| [OpenCowork#160](https://github.com/AIDotNet/OpenCowork/pull/160) | merged |
| [waveloom#6](https://github.com/Menfre01/waveloom/issues/6) | shipped in [v0.7.7](https://github.com/Menfre01/waveloom/releases/tag/v0.7.7), with a regression test pinning both weekend windows |

And one that isn't the weekday defect: [dsh-meter#10](https://github.com/dshworks/dsh-meter/pull/10) —
DeepSeek's vision variant has no flat-price history, so a feed builder that assumes one falls back to
the wrong model and reports 3x.

A different one closed with the field retired rather than the patch taken:
[models.dev#5277](https://github.com/anomalyco/models.dev/pull/5277) argued that `context_over_200k`
reads as a flat 200K threshold while 193 of the 357 entries carrying it use a different one. It was
closed as superseded by [#5335](https://github.com/anomalyco/models.dev/pull/5335), which marks the
field deprecated in the SDK types and points consumers at `tiers` for the real threshold. Same
outcome from the consumer's side — the number that lied is no longer the one you're told to read.

Open right now:

| | |
|---|---|
| [dify-official-plugins#3736](https://github.com/langgenius/dify-official-plugins/pull/3736) | Gemini 3.6 Flash priced at the standard rate that starts 2027-01-01 instead of the introductory rate running now — 2x, and it turns correct by itself in January. The sibling file one directory over has the right number and writes down why. The existing test pinned the wrong one |
| [dify-official-plugins#3737](https://github.com/langgenius/dify-official-plugins/issues/3737) | Both Gemini `-latest` aliases are byte-copies of the models Google's January changelog said they pointed to — stale price *and* stale parameter card. The alias charges $0.50/$3.00; not one of the three models it could resolve to costs that |
| [lobehub#18647](https://github.com/lobehub/lobehub/pull/18647) | Two `gemini-*-latest` aliases are priced as the model they used to point at, not the one their own description names — one of them at a rate that doesn't start until 2027. Plus Gemini 2.5 cache reads at the retired 25% rate. The test that would have caught it existed, but only pinned the one alias that happens to be right |
| [continue#13184](https://github.com/continuedev/continue/issues/13184) | `"gpt-4.1".startsWith("gpt-4")` is true, and `"gpt-4"` is the most expensive row in the table — so the whole 4.1 family bills at 15x–300x |
| [genai-prices#583](https://github.com/pydantic/genai-prices/pull/583) | 6 models priced as if long-context were a marginal tier; ~2x under-priced past the threshold |
| [genai-prices#584](https://github.com/pydantic/genai-prices/pull/584) | `grok-4.6` missing entirely — pricing it raises `LookupError` |
| [genai-prices#581](https://github.com/pydantic/genai-prices/pull/581) | DeepSeek V4 peak/off-peak repricing |
| [helicone#5791](https://github.com/Helicone/helicone/pull/5791) | Gemini 2.5 cache reads billed at 25% of input where Google charges 10% — a 2.5x overcharge, held over from the retired 2.0 rate; the repo's own 3.x entries already use 10% |
| [models.dev#5280](https://github.com/anomalyco/models.dev/pull/5280) | DeepInfra cache-read price derived two different ways; 1 of 17 tier segments disagrees with itself |
| [opencode#44223](https://github.com/anomalyco/opencode/pull/44223) | a hardcoded legacy 200k price overrides the model's real context tier |
| [opencode#44229](https://github.com/anomalyco/opencode/pull/44229) | cache-write tokens added on top of the prompt count when OpenAI already counts them inside it |
| [litellm#38015](https://github.com/BerriAI/litellm/pull/38015) | 13 `wandb/*` rows priced 100,000x too high; three rows in the same block already use the right unit |
| [litellm#38016](https://github.com/BerriAI/litellm/pull/38016) | two `azure/gpt-realtime` rows bill a cache read at the full input price; their OpenAI twin doesn't |
| [llm-prices#67](https://github.com/simonw/llm-prices/pull/67), [#68](https://github.com/simonw/llm-prices/pull/68), [#71](https://github.com/simonw/llm-prices/pull/71) | Missing cached-input prices for 16 Claude/Gemini models; DeepSeek V4 repricing |
| [litellm#38062](https://github.com/BerriAI/litellm/issues/38062) | four `gemini *-latest` aliases bill a cache read at the deprecated 2.0 rate — 25% of input where Google charges 10%; the same alias sits in the file twice with the same input price and two different cache prices |
| [litellm#38064](https://github.com/BerriAI/litellm/issues/38064) | `cost-based-routing` never reads the cache-read price, so on an exact input+output tie the 10x-cheaper provider wins only if you happened to list it first |
| [llmgateway#3782](https://github.com/theopenco/llmgateway/issues/3782) | provider selection scores on input and output only and never reads `cachedInputPrice`, so cache-heavy traffic routes to the more expensive provider — up to 2.6x on their own price data |

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
