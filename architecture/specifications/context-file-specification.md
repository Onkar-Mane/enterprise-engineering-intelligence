# Context File Specification

## Objective

Define the internal structure of repository intelligence context files.

---

# Context Philosophy

Context files are NOT intended to store raw source code.

Instead, they store:

- workflow understanding
- summaries
- references
- relationships
- mappings
- line references
- navigation metadata

---

# Context File Structure

Each context file may contain:

- Introduction
- Workflow Summary
- Common Entities
- Region Index
- Method Index
- Brief Summaries
- Detailed Summaries
- Relationship References
- Related Services
- Related DAL
- Related UI
- Line References

---

# Context Levels

## Brief Context

Contains:
- short explanations
- workflow meaning
- keywords
- references

Used for fast reasoning.

---

## Detailed Context

Contains:
- validations
- edge cases
- execution behavior
- deeper reasoning

Loaded only when required.

---

# Important Principle

Raw source code should only be loaded dynamically when structured intelligence becomes insufficient.