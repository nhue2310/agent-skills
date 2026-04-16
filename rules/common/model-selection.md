
## Model Selection Strategy

**Haiku 4.5** (90% of Sonnet capability, 3x cost savings):
- Lightweight agents with frequent invocation
- Pair programming and code generation
- Worker agents in multi-agent systems

**Sonnet 4.6** (Best coding model):
- Main development work
- Orchestrating multi-agent workflows
- Complex coding tasks

**Opus 4.5** (Deepest reasoning):
- Complex architectural decisions
- Maximum reasoning requirements
- Research and analysis tasks

## Context Window Management

Avoid last 20% of context window for:
- Large-scale refactoring
- Feature implementation spanning multiple files
- Debugging complex interactions

Lower context sensitivity tasks:
- Single-file edits
- Independent utility creation
- Documentation updates
- Simple bug fixes

## Extended Thinking + Plan Mode

Extended thinking is enabled by default, reserving up to 31,999 tokens for internal reasoning.

Control extended thinking via:
- **Toggle**: Option+T (macOS) / Alt+T (Windows/Linux)
- **Config**: Set `alwaysThinkingEnabled` in `~/.claude/settings.json`
- **Budget cap**: `export MAX_THINKING_TOKENS=10000`
- **Verbose mode**: Ctrl+O to see thinking output

For complex tasks requiring deep reasoning:
1. Ensure extended thinking is enabled (on by default)
2. Enable **Plan Mode** for structured approach
3. Use multiple critique rounds for thorough analysis
4. Use split role sub-agents for diverse perspectives


## 10. Context Window Management (AI-Assisted Development)

When working with AI coding assistants (Claude, Copilot, etc.), context window efficiency directly affects output quality.

### What to Avoid at High Context Usage (Last 20%)

Avoid starting these tasks when the context window is near capacity — the AI loses access to earlier instructions and code:

- Large-scale refactoring spanning multiple files
- Feature implementation that touches many layers
- Debugging complex multi-service interactions
- Architecture decisions requiring broad context

### Lower-Context Tasks (Safe at Any Point)

These tasks are self-contained and context-efficient:

- Single-file edits and bug fixes
- Independent utility or helper functions
- Documentation updates
- Schema or migration files
- Writing tests for a specific function

### Extended Thinking for Complex Problems

For architectural decisions, algorithm design, or deep debugging, enable extended thinking for more thorough analysis:

```bash
# Claude Code — toggle extended thinking
Option+T (macOS) / Alt+T (Windows/Linux)

# Set budget cap to avoid excessive token usage on routine tasks
export MAX_THINKING_TOKENS=10000

# View thinking output
Ctrl+O (verbose mode)
```

Use **Plan Mode** for multi-step implementations — it forces a structured approach before writing any code, reducing expensive mid-implementation corrections.
