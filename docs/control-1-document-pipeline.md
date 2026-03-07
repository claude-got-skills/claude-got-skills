# Control Test - Prompt 1: Document Processing Pipeline
**Skill status**: OFF
**Prompt**: "I'm building a document processing pipeline in Python. Each document is about 50 pages. What's the best way to set this up with your API?"

## Key Observations
- Claude used web search (consulted product knowledge / web sources)
- Referenced: github anthropic-sdk-python, Message Batches API page, Claude Docs batch processing, PyPI
- Mentioned PDF native support, base64 encoding, URL reference, Files API
- Gave code example with `anthropic.Anthropic()` client
- Mentioned prompt caching with `cache_control: ephemeral`
- Covered Message Batches API with 50% discount, 10k queries/batch
- Token estimation: 1500-3000 tokens per page
- Model recommendation: Sonnet 4.5 for quality, Haiku 4.5 for cost
- Links to PDF support and Batch processing docs

## Notable: Used web search to get current info, not just training knowledge
## Notable: Mentioned Files API as an upload option
## Notable: Addressed Liam by name (from memory)
