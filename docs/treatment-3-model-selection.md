# Treatment Test - Prompt 3: Model Selection for Chatbot
**Skill status**: ON
**Prompt**: "I need to pick the right Claude model for a customer support chatbot. It needs to be fast and cheap but still accurate. What do you recommend?"

## Key Observations
- **Skill triggered**: YES — "Check product self-knowledge for current model details"
- Used web search (Anthropic model pages, pricing docs, comparison sites)
- Recommended: Claude Haiku 4.5 as primary pick
- Pricing details: $1/M input, $5/M output — "roughly a third of Sonnet's"
- Speed: 4-5x faster than Sonnet 4.5
- Accuracy framing: "matches the previous Sonnet 4 in capability"
- Cost optimization: prompt caching ($0.10/M for cache reads), Batch API 50% discount, tight context
- Tiered architecture: Haiku 4.5 default + escalate to Sonnet 4.5 ($3/$15) for complex queries
- Model string: claude-haiku-4-5-20251001
- Referenced Anthropic pricing page

## Comparison to Control
- Both recommended Haiku 4.5 as primary
- Treatment gave more specific pricing ($1/$5 input/output) vs control
- Both suggested tiered approach with Sonnet escalation
- Treatment included cache pricing detail ($0.10/M)
- Both mentioned Batch API discount
- Both provided model string
- Very similar quality and depth
