# [TIL] Managing context with subagents assisted coding

Date: December 28, 2025

This is an ongoing doc to document all learnings about coding with sub agents

# TIL: Structuring AI Context with .agents, .contexts, and Docs

> Building ButterFlow taught me how to structure context for AI coding agents. This pattern eliminated repetitive explanations and improved consistency from 60% to 95%+.

---

**Resources**:

- [https://code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents)

## The Problem

Working with AI assistants, I kept hitting the same issues:

- **Repeating myself** every session about design patterns and brand voice
- **Inconsistent output** as AI “forgot” earlier decisions
- **Context loss** in long conversations
- **No persistent knowledge** between sessions

---

## The Solution: Subagents + 4-Layer Context Structure

### Subagent File Structure

Subagents are stored in:

- **Project subagents**: `.claude/agents/` (highest priority, version controlled)
- **User subagents**: `~/.claude/agents/` (global, across all projects)

### Complete Project Structure

```
/project-root
├── CLAUDE.md              # Project-wide instructions
├── .claude/
│   └── agents/            # Subagent configurations
│       └── blog-agent.md
├── .contexts/             # Technical decisions
│   └── blog-context.md
└── docs/                  # User-facing documentation
    └── PRD_BLOG_SYSTEM.md
```

---

## Layer 1: CLAUDE.md - Project Constitution

**Purpose**: Design philosophy and universal coding standards

**What goes here**:

- Design aesthetic and brand guidelines
- Code patterns to follow/avoid
- File organization principles

**Length**: As needed, but scannable
**Updates**: Rarely (this is your "constitution")

---

## Subagent Configuration

### Basic Subagent File Format

Subagents use YAML frontmatter followed by the system prompt:

```markdown
---
name: your-subagent-name # Required: lowercase with hyphens
description: When to use this subagent # Required: triggers auto-delegation
tools: Read, Grep, Glob, Bash # Optional: limit tool access
model: sonnet # Optional: sonnet, opus, haiku, 'inherit'
permissionMode: default # Optional: permission handling
skills: skill1, skill2 # Optional: auto-loaded skills
---

Your subagent's system prompt goes here.
Include specific instructions, examples, and constraints.
```

### Key Configuration Fields

**name**: Lowercase with hyphens (e.g., `code-reviewer`, `test-runner`)

**description**: Action-oriented language for auto-delegation

- ✅ "Use PROACTIVELY to review code after changes"
- ✅ "MUST BE USED to run tests and fix failures"
- ❌ "A code reviewer" (too vague)

**tools**: Comma-separated list (limits access for security/focus)

- Common: `Read, Grep, Glob, Bash`
- Exploration only: `Read, Grep, Glob`
- Omit to inherit all tools from parent

**model**:

- `sonnet` - Balanced (default for most tasks)
- `opus` - Most capable, complex reasoning
- `haiku` - Fast, lightweight
- `'inherit'` - Use parent conversation's model

### Built-in Subagents

**general-purpose**

- Model: Sonnet
- Tools: All tools
- Use: Complex multi-step tasks requiring exploration + modification

**explore**

- Model: Haiku (fast)
- Tools: Read-only (Glob, Grep, Read, Bash read-only commands)
- Use: Fast codebase searching without modifications
- Thoroughness: quick, medium, very thorough

**plan**

- Model: Sonnet
- Tools: Exploration tools
- Use: Research codebase before creating implementation plans

### Managing Subagents

**Recommended: Use `/agents` command**

```
/agents
```

Interactive interface for viewing, creating, editing, and deleting subagents.

**Manual Creation**

```bash
mkdir -p .claude/agents
nano .claude/agents/code-reviewer.md
```

---

## Layer 2: .claude/agents/ - Task Workflows (Subagents)

**Purpose**: Executable workflows for specific roles (content writer, code reviewer, etc.)

**What goes here**:

- YAML frontmatter configuration
- Exact workflow steps in system prompt
- Quality checklists
- Common mistakes to avoid
- Role-specific voice/style rules

**Example**: `.claude/agents/code-reviewer.md`

```markdown
---
name: code-reviewer
description: Expert code review specialist. Use PROACTIVELY after writing or modifying code.
tools: Read, Grep, Glob, Bash
model: inherit
---

You are a senior code reviewer ensuring high standards.

When invoked:

1. Run git diff to see recent changes
2. Focus on modified files only
3. Begin review immediately

Review checklist:

- Code clarity and readability
- Proper naming conventions
- No code duplication
- Error handling
- Security (no exposed secrets, SQL injection, XSS)
- Input validation
- Test coverage
- Performance considerations

Provide feedback organized by priority:

- Critical issues (must fix)
- Warnings (should fix)
- Suggestions (consider improving)
```

**Length**: ~200 lines max (action-oriented playbook)
**Updates**: When workflow changes

---

## Layer 3: .contexts/ - Technical Decisions

**Purpose**: Living record of architecture and “why we built it this way”

**What goes here**:

- Current architecture
- Key decisions and trade-offs
- Alternatives considered
- Future roadmap

**Example**: `.contexts/blog-context.md`

```markdown
# Blog System - Technical Context

## Current Architecture (v2.0)

-Individual .md files in /content/blog/
-Build-time generation (zero runtime cost)
-Custom parser (~200 lines, zero deps)

## Key Decisions

### Why Individual Markdown Files?

-Better Git workflow (one file per post)
-CMS-ready for future
-Portable content

### Why Custom Parser vs Libraries?

-Lightweight: 3KB vs 25KB+ (marked, remark)
-Zero dependencies
-Exact features we need

## Future Roadmap

-[ ] Search (at 10+ posts) -[ ] Syntax highlighting (needed soon)
```

**Length**: ~300 lines max (prune quarterly)
**Updates**: When architecture changes

---

## Layer 4: docs/ - Product Requirements & User Docs

**Purpose**: Product specs and external documentation

**What goes here**:

- PRDs (Product Requirements Documents)
- API documentation
- User guides
- Feature specifications

**Length**: Comprehensive (this is the spec)
**Updates**: When product requirements change

---

## Real Results

**Impact**: Context explanation dropped from 5-10 messages/session to 0. Brand inconsistency reduced from 40% to <5%.

**Biggest Win**: Blog migration - AI read `.contexts/blog-context.md`, understood v1.0 → v2.0, executed flawlessly (vs 50+ messages without it)

---

## Key Principles

1. **Context Over Conversation** - Write it once in structured docs, not chat
2. **Workflows Over Instructions** - “1. Do X, 2. Do Y” beats “write good code”
3. **Decisions Over Details** - Document **why**, not **what** (code shows what)
4. **Living Over Static** - Update when code changes
5. **Scannable Over Comprehensive** - Headers, bullets, <300 lines

---

## Do's and Don'ts

### ✅ Do

- Use YAML frontmatter with action-oriented descriptions
- Limit tool access to only what's needed
- Version control project subagents
- Keep files scannable: <200 lines (agents), <300 lines (contexts)
- Update when code changes (living docs)

### ❌ Don't

- Create over-generalized "do everything" subagents
- Write vague descriptions or 500+ line novels
- Grant excessive tool access
- Let docs get stale or duplicate content

---

## Advanced Subagent Usage

### Automatic Delegation

Claude proactively invokes subagents when task matches description with action-oriented language ("use PROACTIVELY", "MUST BE USED")

### Resumable Subagents

### Chaining Subagents

```
> First use the code-analyzer subagent to find performance issues,
> then use the optimizer subagent to fix them
```

---

## Why This Works

**For AI:**

- Structured format is easy to parse
- Persistent context across sessions
- Clear scope (which file for what)

**For Humans:**

- New teammates read same context as AI
- Documents decisions as you make them
- Forces clear articulation

**For Codebase:**

- Self-documenting architecture
- Reduced onboarding time
- Living knowledge baseSubagents can be resumed with
