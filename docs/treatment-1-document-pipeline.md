# Treatment Test - Prompt 1: Document Processing Pipeline
**Skill status**: ON
**Prompt**: "I'm building a document processing pipeline in Python. Each document is about 50 pages. What's the best way to set this up with your API?"

## Key Observations
- **Skill triggered**: YES — thinking shows "Check product self-knowledge for accurate API details"
- Used web search (PDF support docs, batch processing docs, prompt caching docs)
- Mentioned three PDF input methods: URL reference, Base64-encoded, Files API
- Recommended Files API as cleanest for pipeline
- Gave code example with `client.files.create()` and `client.messages.create()`
- Prompt caching with `cache_control: {"type": "ephemeral"}`, 5 min default TTL, extendable to 1 hour
- Batch processing: 50% pricing, discounts stack with caching
- Token estimation: 1,500-3,000 tokens per page (same as control)
- Model recommendation: Sonnet 4.5 for quality/cost balance
- Addressed Liam by name
- Ended with offer to build complete pipeline script

## Comparison to Control
- Similar depth and accuracy
- Skill thinking step visible but response quality comparable
- Both used web search
- Control mentioned Sonnet 4.5 AND Haiku 4.5; treatment focused on Sonnet 4.5
- Both mentioned Files API, prompt caching, batch processing
