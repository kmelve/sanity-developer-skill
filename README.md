# Sanity Development Skill

A comprehensive skill for working with Sanity Content Operating System, following Anthropic's skill authoring best practices.

## Structure

```text
sanity-development/
├── SKILL.md              # Main skill file (overview, tool selection, common patterns)
├── README.md            # This file
└── reference/
    ├── groq-patterns.md # Advanced GROQ query patterns and examples
    ├── schema-design.md # Schema design best practices
    └── debugging.md     # Common pitfalls and debugging workflows
```

## Usage

The skill follows a **progressive disclosure** pattern:

1. **SKILL.md** contains essential information Claude needs for most Sanity operations
2. **Reference files** provide deep-dive content for complex scenarios
3. Claude reads reference files only when needed for specific tasks

## When This Skill Triggers

Claude should use this skill when:

- Working with Sanity projects
- Writing or debugging GROQ queries
- Creating, updating, or querying documents
- Designing schemas or content models
- Managing releases and versioning
- Handling Sanity-specific operations

## Key Principles

### Schema-First Approach

Always check schema before querying content to avoid failed queries and ensure correct field references.

### Tool Selection

- `query_documents` - Search and retrieve
- `create_document` - AI-generated content
- `patch_document` - Precise changes (no AI)
- `update_document` - AI-powered rewrites
- `transform_document` - Format-preserving transformations

### Best Practices

- Verify document existence before operations
- Respect 5-document operation limit
- Use releases for coordinated updates
- Check array vs single reference in schema
- Quote computed field names in GROQ projections

## Quick Reference

### Common GROQ Patterns

```groq
# Basic query with filtering
*[_type == "post" && published == true][0...10]

# With projections (note quoted field names)
*[_type == "post"]{ _id, title, "author": author->name }

# References
*[_type == "post" && author._ref == $id]           # Single reference
*[_type == "post" && $id in authors[]._ref]        # Array reference

# Text search
*[_type == "post" && body match text::query("search")]
```

### Document Operations

```typescript
// Create
create_document({ type, instruction, resource, workspaceName });

// Update with AI
update_document({ operations, paths, resource, workspaceName });

// Precise edit
patch_document({ documentId, operation, resource, workspaceName });

// Publish
publish_document({ id, resource });
```

## File Details

### SKILL.md

Main entry point covering:

- Tool selection guide
- Schema-first principle
- Common GROQ patterns
- Document operations
- Releases workflow
- Anti-patterns
- Best practices

Keep under 500 lines per best practices.

### reference/groq-patterns.md

Deep dive into GROQ:

- Filtering and operators
- Joins and dereferencing
- Aggregations
- Performance optimization
- Error prevention
- Advanced patterns

### reference/schema-design.md

Schema best practices:

- Document type design
- Field types and validation
- References and relationships
- Portable Text patterns
- Localization strategies
- Performance considerations

### reference/debugging.md

Troubleshooting guide:

- Common GROQ errors
- Schema validation issues
- Reference problems
- Performance issues
- Document operation errors
- Tool selection mistakes
- Debugging workflows

## Maintaining This Skill

### Keep It Concise

- Assume Claude knows basics
- Only add Sanity-specific knowledge
- Remove redundant explanations

### Update Patterns

When new patterns emerge:

1. Add to appropriate reference file
2. If critical, add brief mention in SKILL.md
3. Keep SKILL.md under 500 lines

### Test Coverage

Test with:

- Various document types
- Different query complexities
- Multiple tool combinations
- Error scenarios

## Design Decisions

### Why This Structure?

- **SKILL.md** covers 80% of use cases
- **Reference files** handle edge cases and deep dives
- **Progressive disclosure** keeps context window efficient
- **Consistent terminology** throughout all files

### Tool Selection Framework

Tools are categorized by operation type (discovery, creation, modification, lifecycle) rather than alphabetically, making selection intuitive.

### Schema-First Emphasis

Repeated emphasis on checking schema first because it's the single most important pattern for preventing query failures.

### Opinionated Patterns

The skill includes specific recommendations (not just options) because:

- Reduces cognitive load
- Prevents common mistakes
- Follows Sanity best practices
- Provides clear defaults with escape hatches

## Contributing Improvements

When updating this skill:

1. Follow concise writing principles
2. Use concrete examples over abstract explanations
3. Keep SKILL.md under 500 lines
4. Test changes with real Sanity operations
5. Maintain consistent terminology
6. Update README if structure changes

## Version History

- **v1.0** (2025-10-29): Initial skill creation
  - Core tool selection guide
  - GROQ pattern reference
  - Schema design guide
  - Debugging and pitfalls reference
  - Following Anthropic best practices for skills
