# Coding Agent Best Practices

**Repository Purpose:** Curated best practices and patterns for AI coding agents, organized by technology stack and domain.

This repository contains skills that guide agents to write high-quality, performant, and maintainable code. Skills are conditionally loaded based on project type to avoid context overload.

---

## 🎯 Two-Layer Skill Routing

Skills are loaded using a **two-layer detection system** to avoid context overload:

### Layer 1: Project Detection (Available Skills)

Detect the project type to determine which skills are **available**:

| Indicator | Project Type | Available Skills |
|-----------|--------------|------------------|
| `package.json` with `"react"` or `"next"` | React/Next.js | `react-performance`, `nextjs-architecture`, `web-design` |
| `.liquid` files, `theme.json` | Shopify Liquid | `web-design` only |
| `package.json` with `"vue"` | Vue.js | `web-design` only |
| `package.json` with `"angular"` | Angular | `web-design` only |
| Plain HTML/CSS/JS (no framework) | Vanilla Web | `web-design` only |
| No UI files (APIs, CLIs, services) | Backend-only | None (use Context7 for library docs) |

### Layer 2: Query Context Analysis (Load Only What's Needed)

Analyze the **user's query** to determine which available skills to **actually load**:

| Query Context | Load Skills | Skip Skills |
|---------------|-------------|-------------|
| Performance, optimization, re-renders, bundle size, waterfalls | `react-performance` | `nextjs-architecture`, `web-design` |
| Folder structure, architecture, server actions, data providers, patterns | `nextjs-architecture` | `react-performance`, `web-design` |
| UI, styling, CSS, accessibility, responsive, design, layout | `web-design` | `react-performance`, `nextjs-architecture` |
| Generic code changes (refactor, rename, fix bug) | Minimal - use Context7 if needed | All unless specifically relevant |
| Multiple concerns (e.g., "optimize rendering + accessibility") | Load matching skills only | Unrelated skills |

**⚠️ CRITICAL**: Only load skills that match BOTH the project type AND the user's current task. Do not load performance optimization rules when the user is asking about CSS styling. Do not load React/Next.js rules when working on a Shopify theme or Vue app.

### Practical Examples

**Example 1: CSS issue in Next.js project**
- Project: Next.js → Available: `react-performance`, `nextjs-architecture`, `web-design`
- Query: "The hover state on this button isn't working"
- **Action**: Load only `web-design` (skip performance and architecture rules)

**Example 2: Data fetching architecture**
- Project: Next.js → Available: `react-performance`, `nextjs-architecture`, `web-design`
- Query: "How should I structure data fetching for this dashboard?"
- **Action**: Load `nextjs-architecture` (maybe also `react-performance` if asking about waterfalls)

**Example 3: Generic refactor**
- Project: Next.js → Available: `react-performance`, `nextjs-architecture`, `web-design`
- Query: "Rename this function and update all usages"
- **Action**: Load no skills (just perform the refactor)

**Example 4: Performance optimization**
- Project: Next.js → Available: `react-performance`, `nextjs-architecture`, `web-design`
- Query: "This component re-renders too often, causing lag"
- **Action**: Load only `react-performance` (skip architecture and design rules)

---

## 📚 Available Skills

### `/skills/react-performance/`

**React and Next.js Performance Optimization**

- **Applies to**: React, Next.js, Preact projects
- **Read when**: 
  - Optimizing performance
  - Reducing bundle size
  - Fixing re-renders or waterfalls
  - Implementing data fetching
  - Server-side rendering optimization
- **Skip if**: Shopify, Vue, Angular, Svelte, plain HTML, or backend-only projects

Contains 45+ rules across 8 categories (eliminating waterfalls, bundle optimization, server-side performance, client data fetching, re-render optimization, etc.)

### `/skills/nextjs-architecture/`

**Next.js App Router Architecture Patterns**

- **Applies to**: Next.js projects using App Router
- **Read when**:
  - Structuring features and folders
  - Implementing server actions
  - Creating context providers
  - Setting up data providers
  - Building reusable components
  - Managing state with useReducer
- **Skip if**: Not using Next.js App Router (Pages Router, other frameworks, etc.)

Contains patterns for: code style, component naming, contexts, data fetching, data providers, feature architecture, server actions, useReducer pattern, and Zod schemas.

### `/skills/web-design/`

**Web Interface Guidelines (UI/UX/Accessibility)**

- **Applies to**: All web projects with user interfaces
- **Read when**:
  - Reviewing UI code
  - Auditing accessibility
  - Checking design compliance
  - Implementing responsive design
- **Skip if**: Backend APIs, CLI tools, or projects without UI

Guidelines for semantic HTML, accessibility (WCAG), responsive design, and modern web interface patterns.

---

## 🔍 Documentation Lookup (Context7 MCP)

When working with **external libraries, frameworks, or APIs**, automatically fetch up-to-date documentation using the Context7 MCP server.

### Auto-Detect Triggers

Use Context7 when you encounter:
- Unfamiliar API or method
- Implementing new integration with external library
- Need to verify current best practices for a library
- Checking latest API changes or features
- Library-specific patterns or conventions

### Process

1. **Resolve Library ID**: Use `resolve-library-id` tool to find the Context7-compatible library ID
   - Example: "React Query" → `/tanstack/query`
   - Example: "Zod" → `/colinhacks/zod`

2. **Query Documentation**: Use `query-docs` tool with the library ID and your specific question
   - Be specific: "How to set up React Query with Next.js App Router?"
   - Not vague: "queries"

3. **Apply Documentation**: Integrate documentation insights with relevant skills from this repository

### Example Workflow

```
Task: Implement authentication with NextAuth.js
  ↓
1. Detect project: Next.js (load react-performance, nextjs-architecture)
  ↓
2. Context7: resolve-library-id("NextAuth.js") → /nextauthjs/next-auth
  ↓
3. Context7: query-docs(/nextauthjs/next-auth, "How to configure JWT sessions?")
  ↓
4. Apply NextAuth docs + nextjs-architecture patterns
```

### Important Notes

- **Do not call Context7 more than 3 times per question** - Be strategic with queries
- **No sensitive data** - Never include API keys, passwords, or credentials in queries
- Context7 works for **any library** - Not limited to the skills in this repo

---

## 🚀 How to Use This Repository

### For AI Agents (You)

1. **Read this file first** to understand project context
2. **Detect the project type** using Layer 1 indicators → filters AVAILABLE skills
3. **Analyze the user query** using Layer 2 context → determines which skills to LOAD
4. **Load ONLY skills relevant to the task** by reading their `AGENTS.md` files
5. **Use Context7** when external library documentation is needed
6. **Apply patterns** from loaded skills to your task

### For Humans

- Each skill contains:
  - `AGENTS.md` - Complete guide for AI agents
  - `SKILL.md` - Metadata and trigger conditions
  - `README.md` - Contribution guidelines (where applicable)
  - `rules/` - Individual rule files (where applicable)

- To add new patterns, create a new skill or extend existing ones
- Follow the nested `AGENTS.md` structure for consistency

---

## 📋 Skill Metadata

Each skill includes a `SKILL.md` file with:
- `name` - Skill identifier
- `description` - What the skill covers
- `applies_to` - File patterns and project indicators
- `triggers` - Keywords/tasks that activate the skill
- `skip_if` - Explicit exclusion conditions

This metadata helps tools like Cursor, Claude, and other AI systems determine when to load each skill.

---

## 🎓 Philosophy

**Quality over Quantity**: Load only what's relevant to avoid context overload.

**Documentation-First**: Use Context7 to fetch library-specific docs rather than guessing APIs.

**Project-Aware**: Patterns are technology-specific and shouldn't bleed across stacks.

**Actionable**: Every rule includes concrete examples of what to do (and what not to do).

---

## 🤝 Contributing

To add a new skill:

1. Create `/skills/[skill-name]/` directory
2. Add `AGENTS.md` with complete guidelines
3. Add `SKILL.md` with metadata (applies_to, triggers, skip_if)
4. Update this root `AGENTS.md` to include the new skill in routing table

Keep skills focused, well-documented, and include clear applicability conditions.
