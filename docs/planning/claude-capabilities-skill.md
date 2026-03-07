
 # Task: Design a publishable Claude capabilities awareness skill (and potentially plugin)

 ## Problem we're solving:
  Claude's training data has a cutoff, which means it doesn't always recognise its own current capabilities. For example, Claude may not know about features released after its
  training date — computer use improvements, new tool types, agent skills, fast mode, 1M context windows, etc. This creates a gap where Claude under-utilises its own capabilities, or
  designs around limitations that no longer exist.

 ## What we want to build:

  A skill (and potentially a plugin) that keeps Claude aware of its current capabilities. Two components:

  1. **Capabilities Awareness Skill**
  - Registers with a description that triggers at session start (similar to how superpowers:using-superpowers works)
  - Injects a condensed summary of Claude's current capabilities that may not be in its training data
  - Covers: API features, tool types, Claude Code features, agent capabilities, context window sizes, new model features
  - Does NOT cover third-party plugins/skills (those are project-specific)
  - The summary should be concise enough to not waste context budget, but comprehensive enough to prevent Claude from designing around non-existent limitations

  2. **Update Command**
  - /update-claude-capabilities — fetches latest information from Anthropic's documentation, diffs against current local knowledge, extracts relevant changes, updates local capability
   files, and regenerates the skill summary
  - Sources: Anthropic API docs, Claude.ai feature pages, Claude Code CHANGELOG
  - Should be semi-automated: fetch and diff is programmatic, interpreting significance may need AI

  3. **Publish Command**
  - /publish-claude-capabilities — once local updates are verified, packages the skill for distribution
  - Currently skills.sh only supports skills; a plugin with commands would need a marketplace
  - The plugin-dev plugin (plugin-dev:create-plugin, plugin-dev:skill-development, plugin-dev:command-development) provides guidance on building both

 ## Design questions to research and explore:
  1. Should this be a standalone skill (publishable via skills.sh) or a plugin (with skill + commands)?
  2. What's the right format for the capabilities summary? (Markdown table? Structured sections? How concise can it be while remaining useful?)
  3. How should versioning work? (Tied to Claude model versions? Independent versioning?)
  4. What's the maintenance model? (Who updates it? How often? Community contributions?)
  5. Could the update command use Distill-style change detection, or is manual trigger sufficient?
  6. What's the minimum viable version? (Just the awareness skill, with update/publish added later?)

 ## Relevant context:
  - We have a local folder of Claude capabilities docs at .planning/research/claude-capabilities/ with ~20 files covering API features, Agent SDK, computer use, hooks, plugins, MCP, structured outputs, etc.
  - The plugin-dev plugin provides up-to-date guidance on building plugins, skills, commands, agents, and hooks
  - skills.sh is the current distribution channel for skills
  - We're building a bid management platform where this capability awareness is critical for our own development workflow, but the skill would also be valuable to the broader Claude
  user community

 ## Output: 
 
 A design document covering the skill structure, content format, update mechanism, and distribution strategy. Include mockups of what the capabilities summary would look like and how the update command would work.