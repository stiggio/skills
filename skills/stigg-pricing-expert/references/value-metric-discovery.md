# Value Metric Discovery — Reference

The **value metric** is what you charge for. When more of it happens, the customer gets more value. Picking the right one is the single highest-leverage decision in your pricing model.

## What makes a good value metric

A value metric should be:

1. **Aligned with customer value.** When they get more, they're getting more out of the product.
2. **Measurable in real time.** If you can't count it, you can't bill on it.
3. **Predictable at the customer's end.** Customers should be able to forecast their bill.
4. **Aligned with your costs.** When the metric grows, your COGS should grow proportionally — otherwise you're either subsidizing or gouging.

Two of four is workable; three is solid; four is rare and indicates a strong fit.

## Diagnostic questions

### "Where does usage scale with value?"

The customer's usage of *what* tracks how much they care about the product?

- A team Slack: number of messages? team size? channels? — likely **active users**.
- An email tool: emails sent? contacts? campaigns? — likely **emails sent** or **contacts** depending on use case.
- An LLM wrapper: tokens consumed? generations? users? — likely **tokens** or **generations**.

### "What would the customer pay more for if it grew?"

Sanity-check the metric. If the metric grew 10×:

- Would they happily pay 10× more? (Strong metric.)
- Would they grumble but pay? (OK metric.)
- Would they churn? (Wrong metric.)

### "Is the metric measurable in real time?"

If not:

- **Use a proxy** that *is* real-time and correlates strongly. (E.g., bill on "API calls" rather than "minutes of generated audio" if minutes are hard to count.)
- **Use periodic reporting** (`reportUsage` calculated style) instead of per-event ingestion.

### "Does the metric track your costs?"

If your COGS scale with the metric (LLM tokens → API costs to OpenAI; GB stored → S3 costs), great. If not (per-seat charge but the heavy users cost you 50× more than light users), reconsider — you'll bleed at the top end.

## Common mismatches

| Mismatch | Symptom | Fix |
|---|---|---|
| Charging per-seat but cost is per-usage | Heavy-usage seats subsidized by light-usage seats; gross margin variance | Switch to usage-based or hybrid (per-seat base + per-call overage) |
| Charging per-API-call but value is "outcomes" | Customers feel nickel-and-dimed; can't predict bills | Charge by outcome (e.g., per converted lead) and measure API calls internally |
| Charging flat fee but customer expectations vary 10× | Some customers leaving 90% of value on the table; others rage at limits | Tier the flat fee, or add a usage component |
| Charging per "generation" on AI when models cost vary | One tier of customers consuming way more compute than another | Use **credits** with a custom formula (`stigg-credits`) |

## Multiple metrics — pick a primary

If multiple metrics surface during discovery, pick one as **primary** (it's the headline value metric) and others as:

- **Soft caps** in the entitlement (block at a generous limit; only matters for the worst abusers).
- **Add-on monetization** (sell more of the secondary metric as an explicit add-on).
- **Tier differentiation** (Pro tier includes more of the secondary metric than Starter).

Don't try to bill on three metrics simultaneously self-serve — customers can't reason about the bill.

## Examples — a few real-world value-metric picks

| Product | Value metric | Charge model |
|---|---|---|
| GitHub | Active users (seats) + storage + minutes | Per-seat plus usage (hybrid) |
| Stripe | Successful charges (dollar volume processed) | Percentage of GMV (usage-based) |
| Datadog | Hosts monitored, custom metrics, log GB | Per-unit per resource type (usage-based) |
| Vercel | Bandwidth, function invocations, build minutes | Hybrid (flat plan + usage overages) |
| OpenAI API | Tokens (with model multipliers) | Credits with custom formula |
| Notion | Active users + AI usage | Per-seat + AI add-on (hybrid) |

## Don't reverse-engineer from competitor pricing

"Competitor X charges $Y per Z" tells you what works in their context, not yours. Anchor on **your customer's value metric**. Use competitor pricing as a sanity-check on numerical magnitudes, not as the source of the model.
