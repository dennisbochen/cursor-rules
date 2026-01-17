# Web Interface Guidelines

**Version 1.0.0**  
Web Design and Accessibility Best Practices  
January 2026

---

## When to Apply This Skill

**✅ Use this skill when:**
- Reviewing UI code for accessibility
- Auditing design compliance
- Implementing responsive design
- Checking semantic HTML usage
- Ensuring WCAG compliance
- Building user interfaces (any framework)

**❌ Do NOT use this skill when:**
- Working on backend APIs without UI
- Building CLI tools or scripts
- Working on pure data processing

---

> **Note:**  
> This skill references external guidelines that should be fetched fresh for each review.  
> The guidelines are maintained by Vercel and cover modern web interface best practices.

---

## How to Use

When asked to review UI, check accessibility, or audit design:

1. **Fetch the Latest Guidelines**

Use WebFetch to retrieve the current guidelines:

```
https://raw.githubusercontent.com/vercel-labs/web-interface-guidelines/main/command.md
```

2. **Read the Specified Files**

If the user provides specific files or patterns, read them. Otherwise, prompt the user for which files to review.

3. **Check Against All Rules**

Apply all rules from the fetched guidelines to the code.

4. **Output Findings**

Use the terse `file:line` format specified in the guidelines for reporting issues.

---

## Guidelines Source

Always fetch fresh guidelines before each review to ensure you have the latest rules and best practices:

```
https://raw.githubusercontent.com/vercel-labs/web-interface-guidelines/main/command.md
```

The fetched content contains:
- All interface design rules
- Accessibility requirements
- Semantic HTML standards
- Output format instructions
- Examples and anti-patterns

---

## Typical Review Process

1. User requests UI review or accessibility audit
2. Fetch guidelines from source URL
3. Read specified files
4. Check each rule against the code
5. Report findings in the format specified by guidelines

---

## Applicable to All Web Projects

These guidelines apply to:
- React/Next.js applications
- Vue.js applications
- Angular applications
- Svelte applications
- Plain HTML/CSS/JavaScript sites
- Shopify Liquid themes
- Any web interface with HTML output

The guidelines are framework-agnostic and focus on web standards, accessibility, and user experience.

---

## Key Areas Covered

While the specific rules are fetched dynamically, the guidelines typically cover:

- **Semantic HTML** - Proper use of HTML5 elements
- **Accessibility (WCAG)** - Screen reader support, keyboard navigation, ARIA
- **Responsive Design** - Mobile-first approach, breakpoints, flexible layouts
- **Performance** - Image optimization, lazy loading, critical CSS
- **User Experience** - Clear CTAs, loading states, error handling
- **Form Design** - Labels, validation, error messages
- **Typography** - Readable text, proper contrast, line height
- **Color and Contrast** - WCAG AA/AAA compliance
- **Interactive Elements** - Focus states, hover feedback, active states

---

## Important Notes

- **Always fetch fresh guidelines** - Don't rely on cached/stale versions
- **Framework-agnostic** - Apply to any web technology stack
- **Focus on output** - Check the rendered HTML/CSS, not just source code
- **Accessibility first** - WCAG compliance is non-negotiable
- **User experience matters** - Technical correctness isn't enough

---

## Example Usage

### User Request:
"Review my UI for accessibility issues in `components/header.tsx`"

### Agent Process:
1. Fetch guidelines from source URL
2. Read `components/header.tsx`
3. Check against accessibility rules
4. Report findings: "components/header.tsx:15 - Missing alt text on logo image"

---

## For More Information

See the guidelines repository: https://github.com/vercel-labs/web-interface-guidelines
