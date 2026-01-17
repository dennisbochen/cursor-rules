---
name: web-design
description: Review UI code for Web Interface Guidelines compliance. Use when reviewing UI, checking accessibility, auditing design, or ensuring web best practices across any web technology stack.
metadata:
  author: vercel
  version: "1.0.0"
applies_to:
  - "**/*.html"
  - "**/*.tsx"
  - "**/*.jsx"
  - "**/*.vue"
  - "**/*.svelte"
  - "**/*.liquid"
  - "**/*.css"
  - "**/*.scss"
  - "All web projects with UI"
triggers:
  - "review UI"
  - "check accessibility"
  - "audit design"
  - "review UX"
  - "WCAG compliance"
  - "semantic HTML"
  - "responsive design"
  - "web standards"
skip_if:
  - "backend-only APIs"
  - "CLI tools without UI"
  - "pure data processing scripts"
---

# Web Interface Guidelines

Review files for compliance with Web Interface Guidelines.

## How It Works

1. Fetch the latest guidelines from the source URL below
2. Read the specified files (or prompt user for files/pattern)
3. Check against all rules in the fetched guidelines
4. Output findings in the terse `file:line` format

## Guidelines Source

Fetch fresh guidelines before each review:

```
https://raw.githubusercontent.com/vercel-labs/web-interface-guidelines/main/command.md
```

Use WebFetch to retrieve the latest rules. The fetched content contains all the rules and output format instructions.

## Usage

When a user provides a file or pattern argument:
1. Fetch guidelines from the source URL above
2. Read the specified files
3. Apply all rules from the fetched guidelines
4. Output findings using the format specified in the guidelines

If no files specified, ask the user which files to review.
